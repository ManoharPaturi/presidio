# Fix: StanzaNlpEngine German multi-word tokens replace the doc text and drop entities (#2249)

Branch: `fix/stanza-mwt-offsets` (base: `main` @ `a7b17c75`)
Files changed:

- `presidio-analyzer/presidio_analyzer/nlp_engine/stanza_nlp_engine.py`
- `presidio-analyzer/tests/test_stanza_nlp_engine.py`

## Problem

With `StanzaNlpEngine` and a German model, any text containing a German
preposition-article contraction (`im`, `am`, `zum`, `zur`, `beim`, `vom`,
`ins`, `ans`) lost every NER entity, and on indented text (JSON, tables,
lists) the analyzer returned HTTP 500
`Did not find word '...' in the list of tokens although it is expected to be found`.

Reproduced on pristine `main` (presidio_analyzer 2.2.364, stanza 1.14.0,
spacy 3.8.16, Python 3.12):

```text
text   = "Wir treffen uns im Büro mit Thomas Bergmann."
doc.text (pristine) = 'Wir treffen uns in dem Büro mit Thomas Bergmann . '  <- replaced
doc.ents (pristine) = []                                                       <- dropped
analyzer  (pristine) = []                                                      <- 200, empty
analyzer "in dem"    = [PERSON 32-47]                                          <- control works
indented text + IP   = ValueError: Did not find word '192.168.10.20' ...       <- 500
```

## Root cause

1. `StanzaNlpEngine.load` builds the pipeline with
   `processors="tokenize,pos,lemma,ner"`. Stanza force-adds the `mwt`
   processor for German and expands `im` into the words `in` + `dem`.
2. `StanzaTokenizer.__get_tokens_with_heads` flattened `token.words`, so the
   token texts (`in`, `dem`) no longer matched the original text (`im`).
3. `__get_words_and_spaces` raised `ValueError`, and `_convert_doc` then
   replaced the whole doc text with `" ".join(tokens)` (the
   "Due to multiword token expansion ..." warning).
4. Stanza NER spans carry offsets of the original text. In the replaced doc
   they hit no token boundary, `doc.char_span` returned `None`, and entities
   were silently dropped.
5. The replaced text is shorter than the original when the text has runs of
   whitespace, so `NlpArtifacts.tokens_indices` ended before pattern matches
   near the end of the original text; for those,
   `LemmaContextAwareEnhancer._find_index_of_match_token` raised `ValueError`
   -> HTTP 500.

This is the presidio-side face of explosion/spacy-stanza#70. Because
Presidio inlines the spacy-stanza tokenizer, the fix has to land here.

## Fix

`StanzaTokenizer.__get_tokens_with_heads` now keeps a multi-word token as a
single surface token instead of flattening its expanded words:

- New `MultiWordTokenSurface` wrapper: `text` and `lemma` are the surface
  form (e.g. `im`); all other annotations (upos, xpos, feats, head,
  deprel, ...) are borrowed from the first expanded word.
- A word-index-to-token-index map per sentence remaps 1-based word heads to
  the collapsed token list, so dependency heads of the words surrounding a
  multi-word token stay correct (exercised by a dedicated unit test; the
  default German model has no parser, but e.g. Spanish/French do).
- Sentence offsets count emitted tokens, keeping heads correct across
  sentence boundaries.

With this, `__get_words_and_spaces` aligns again: `doc.text == text`,
entities keep their original character offsets, `tokens_indices` cover the
whole original text (no more enhancer `ValueError`), and the mwt lemma is
the surface form so lemma-based context words stay meaningful. Ordinary
tokens are emitted exactly as before (`len(token.words) == 1` path is
byte-for-byte the previous behavior; English head/tag assertions in the
existing test still pass).

Trade-off (same as the production patch proposed in the issue): the
morpho-syntactic annotation of the individual expanded words is lost for
that one token. For a PII analyzer this is the right side to be on - the
silent empty result is the harmful outcome.

## Verification

Python 3.12 venv, `pip install -e presidio-analyzer` + `stanza==1.14.0` +
`pytest`, German (`stanza.download("de")`, default packages: tokenize,
mwt, pos, lemma, ner) and English stanza models.

Reproduction script (see below) after the fix:

```text
doc.text == text              True
tokens    ['Wir','treffen','uns','im','Büro','mit','Thomas','Bergmann','.']
ents      [('Thomas Bergmann', 'PER', 28, 43)]
analyzer "im"     -> [PERSON 28-43, score 0.85]
analyzer "in dem" -> [PERSON 32-47, score 0.85]   (control, unchanged)
indented + IP     -> [IP_ADDRESS 73-86]            (no ValueError)
```

Tests (`presidio-analyzer/tests/test_stanza_nlp_engine.py`), following the
existing spacy-stanza test conventions (module fixture with
`pytest.importorskip("stanza")` + `stanza.download`, `skip_engine` marker):

- `test_spacy_stanza_german_multiword_tokens` - the issue's canonical
  sentence: text preserved, no alignment warnings, `im` kept as a single
  surface token (lemma `im`, ADP), PERSON found at original offsets.
- `test_spacy_stanza_german_contractions_keep_text_and_offsets` - all eight
  contractions (im/am/zum/zur/beim/vom/ins/ans) keep the text, the surface
  token, and per-token offset alignment.
- `test_spacy_stanza_german_mwt_token_indices_cover_trailing_matches` - the
  HTTP-500 repro: indented text, every token's span maps back onto itself,
  and the trailing IP match is covered by a token.
- `test_get_tokens_with_heads_collapses_mwt_and_remaps_heads` (pure unit
  test, no models) - mwt collapsed and dependency heads remapped across it.

Results:

- On pristine `main`: the 10 German model-backed tests fail (10 failed),
  the unit test errors on the missing `MultiWordTokenSurface` import.
- With the fix: `tests/test_stanza_nlp_engine.py` 12 passed.
- No regressions: `tests/test_stanza_recognizer.py` (13) and
  `tests/test_stanza_batch_processing.py` (16) pass - English tokenizer
  output, lemmas, POS, heads and the bulk `_convert_doc` path (which shares
  `__get_tokens_with_heads`) are unchanged.
- Lint: `ruff check` / `ruff format` on the touched files report exactly
  the pre-existing findings of `main` (8 in the test file), no new ones.

## Reproduction script

`python /tmp/repro_2249.py` (essentially the in-process repro from the
issue, plus an `AnalyzerEngine` run for the PERSON and IP cases):

```python
from presidio_analyzer.nlp_engine import StanzaNlpEngine

engine = StanzaNlpEngine(models=[{"lang_code": "de", "model_name": "de"}])
engine.load()
doc = engine.nlp["de"]("Wir treffen uns im Büro mit Thomas Bergmann.")
print(repr(doc.text))    # pristine: 'Wir treffen uns in dem Büro ... . '
print(list(doc.ents))    # pristine: []
```
