# Translation evaluation

This document explains how the benchmark results are evaluated and how to
decide which platform translates better.

## Folder layout

```
data/
├── source/         raw corpus (en-pl-pairs.tsv)
├── preprocessed/   generated app inputs (source;gold format)
├── results/        result files copied from the devices
├── evaluation/     evaluation outputs and reports
├── translation-notebook.ipynb
└── EVALUATION.md
```

The notebook's preprocessing cell reads `source/` and writes `preprocessed/`;
the evaluation cells read `results/` and write into `evaluation/`:

- `evaluation/scores.csv` — all metrics per platform/direction/size
  (chrF, BLEU, bootstrap win rate, COMET, BERTScore). Each flow fills
  only its own columns, so local and Colab runs merge instead of
  overwriting each other.
- `evaluation/disagreements_{direction}_{size}.csv` — the sentences
  where the platforms differ the most, for manual error analysis.

The evaluation runs as four flows: `evaluate("short")` (quick check),
`evaluate("full")` (thesis numbers), COMET, and BERTScore. The last two
need packages installed by the notebook's setup cell (run once, then
restart the session); on this corpus they are best run on Google Colab
with a GPU runtime.

## Input files

Each app produces result files named:

```
{platform}_{source}-{target}_{size}.csv
```

Examples: `ios_pl-en_short.csv`, `android_en-pl_full.csv`.

Each file is semicolon-separated with a header:

```
text;translation;gold
```

- `text` — the source sentence.
- `translation` — what the on-device model produced (Apple Translation on
  iOS, ML Kit on Android).
- `gold` — the reference translation from the Tatoeba corpus.

Copy the result files from the devices into `data/results/`. The
evaluation cells in `translation-notebook.ipynb` discover them there
automatically by the naming pattern.

**Note on row order:** Android writes lines in completion order (its
workers run concurrently), so the same sentence can be on different rows
in the iOS and Android files. The evaluation therefore joins result files
on the `text` column and never compares files row-by-row.

## Metrics

Run the evaluation cells in `translation-notebook.ipynb` after the
preprocessing cell. chrF and BLEU are implemented in the notebook with
the Python standard library, so the main pipeline needs **no packages**.
An optional cell cross-checks the numbers against `sacrebleu` (the
reference implementation) when that package is available.

### chrF (primary string metric)

Character-level F-score (Popović 2015): character 1–6-grams, recall
weighted double (β=2). Character n-grams handle Polish morphology
(case endings, inflection) better than word-level metrics, so chrF is the
main string-based metric here. Range 0–100, higher is better.

### BLEU

Word 1–4-gram precision with a brevity penalty (Papineni et al. 2002).
The standard metric in MT literature, reported for comparability. It
underestimates quality for morphologically rich languages like Polish.
Range 0–100, higher is better. The notebook's tokenizer splits words and
punctuation, close to sacrebleu's `13a` — small differences from
published sacrebleu scores are expected.

### Caveat: one gold reference

There is exactly one gold translation per sentence. A translation can be
correct and still differ from the gold (synonyms, word order). String
metrics punish this. Therefore:

- Do not declare a winner from BLEU alone.
- Prefer chrF over BLEU for ranking.
- Confirm with a neural metric (COMET) before drawing conclusions.

### COMET and BERTScore (neural metrics, optional cells)

- **COMET** (`Unbabel/wmt22-comet-da`) scores meaning using source,
  translation, and gold. It correlates best with human judgment.
  System score is roughly 0–1, higher is better.
- **BERTScore** compares contextual embeddings of translation and gold.
  Reported as F1, higher is better.

Both need large model downloads and benefit from a GPU. The notebook
cells are written so they can be pasted into Google Colab: upload the
result CSVs, uncomment the `pip install` line, and run. On a machine
without the packages the cells print a message instead of failing.

## Statistical significance

A small metric difference can be noise. The paired bootstrap cell
resamples sentences (both platforms are evaluated on the same sample, so
the test is paired) and reports how often each platform wins chrF over
1000 resamples. Read it like this:

- Win rate ≥ 0.95 — the difference is significant (p < 0.05).
- Win rate below that — treat the platforms as tied on this metric; look
  at COMET and at the qualitative sample instead.

The 100-sentence short sets have little statistical power; use the full
sets for final conclusions.

## Deciding "which platform is better"

Correctness is one axis. Report at minimum:

1. **Quality:** chrF, BLEU (and COMET if available) per direction, with
   significance tests. Platforms can differ per direction — pl→en and
   en→pl are separate results.
2. **Speed:** the benchmark prints per-file durations in the app logs;
   compare sentences per second.
3. **Qualitative sample:** inspect the sentences where the two
   platforms disagree the most (the notebook's disagreement cell lists
   them). Categorize errors manually: wrong word sense, grammar,
   omissions, hallucinations.

A defensible conclusion has the form: "Platform X is significantly
better at direction A on chrF and COMET, platforms are tied on
direction B, and platform Y is N× faster."
