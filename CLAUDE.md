# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A single Jupyter notebook that turns a raw Excel export of loan/dossier rejection history into a structured Excel report: [.notebook/04_analyse_complete_motifs_rejet.ipynb](.notebook/04_analyse_complete_motifs_rejet.ipynb). There is no application code, package, or test suite — the notebook *is* the deliverable. All comments, variable names, and printed output in the notebook are in French; keep new code consistent with that.

## Environment setup

A `.venv` already exists at the repo root (created with `uv`, Python 3.14) but only has Jupyter kernel dependencies installed — pandas/numpy/matplotlib/seaborn/scikit-learn/openpyxl are **not** preinstalled. There is no `requirements.txt` or `pyproject.toml`; the notebook's own first cell is the install step:

```
%pip install pandas openpyxl matplotlib seaborn scikit-learn
```

Run that cell (or `./.venv/Scripts/python.exe -m pip install pandas openpyxl matplotlib seaborn scikit-learn` from the shell) once per environment before executing the rest of the notebook.

To run the notebook headlessly end-to-end (e.g. to regenerate outputs after an edit):

```
./.venv/Scripts/python.exe -m jupyter nbconvert --to notebook --execute --inplace ".notebook/04_analyse_complete_motifs_rejet.ipynb"
```

## Data flow / architecture

The notebook is a linear, single-pass pipeline — there's no module boundary to navigate, just read cells top to bottom. Each stage feeds the next through in-memory DataFrames; understanding the shape of the data at each stage matters more than any individual function:

1. **Load** — reads `data/query_result.xlsx` (sheet `"Résultats de la requête"`), validates required columns (`Historique Rejets`, `Region`, `Nom Operateur Court`, `Niveau`, `Nb Occurrences Rejet`, `Nb Dossiers Distincts`) → `df` (one row per dossier).
2. **Event extraction** — regex-parses the free-text `Historique Rejets` column (format `#N [level]: dd/mm/yyyy HH:MM | body ... → Corrigé: ...`) into one row per rejection event → `events`, exported to `data/output/rejection_events.csv`.
3. **Reason splitting** — splits each event's body on bullets/sentences into atomic candidate reasons → `motifs` (one row per raw reason), exported to `data/output/rejection_reasons_detailed.csv`.
4. **Text normalization** — `reason_raw` is never mutated; `reason_clean` is a readable normalized form, `reason_model` is the tokenized form used for n-grams/clustering. Business actions (e.g. "à compléter") are deliberately normalized to stable tokens like `information_a_completer` rather than stripped, so meaning survives cleaning — see `ACTION_NORMALIZATIONS` in cell 7.
5. **N-gram exploration** — TF-IDF over `reason_model` (unigrams/bigrams/trigrams) purely to inspect dominant phrasing, params `NGRAM_RANGE`, `MIN_DF`, `MAX_DF` in cell 2.
6. **Clustering** — `AgglomerativeClustering` (cosine, average linkage) over TF-IDF vectors groups similar motifs into `cluster_id` for human review; `N_CLUSTERS` in cell 2 controls granularity. Falls back to a looser vectorizer config if the dataset is too small for the primary settings.
7. **Theme assignment** — `THEME_TAXONOMY` (cell 11) is a hand-maintained list of regex rules mapping normalized text to a stable business theme (`stable_theme`). Priority order: rule match → manual `CLUSTER_THEME_MAPPING` (cluster_id → theme, filled in after reviewing `cluster_summary`) → `autre_a_revoir` (needs manual review). This is the piece most likely to need editing as new rejection phrasing appears — extend `THEME_TAXONOMY` rather than post-processing its output.
8. **Aggregation & charts** — crosstabs by region/operator, monthly evolution, delay stats; matplotlib/seaborn figures saved to `data/output/figures/`.
9. **Excel report** — writes every intermediate table to its own sheet in `data/output/rapport_analyse_motifs_rejet.xlsx` via `openpyxl`, embeds the charts on a `graphiques` sheet, and applies uniform auto-filter/frozen-header/column-width formatting across all sheets.

## Working on the taxonomy

When rejection reasons aren't being categorized correctly, the fix almost always belongs in one of two places in cell 11:
- `THEME_TAXONOMY`: add/adjust a regex pattern against `reason_model` (the normalized, underscore-joined token form, not `reason_raw`).
- `CLUSTER_THEME_MAPPING`: after inspecting `cluster_summary`'s `top_ngrams`/`examples` for a given `cluster_id`, map it to an existing `stable_theme` if no single regex rule cleanly covers it.

`coverage_pct` and the `motifs_a_revoir` sheet in the generated report are the signal for how much of the data still falls into `autre_a_revoir`.

## Data files are committed

Unusually, `data/query_result.xlsx` (source) and everything under `data/output/` (generated CSVs, the Excel report, PNG figures) are tracked in git — there is no `.gitignore` rule excluding them (`.gitignore` at the repo root is empty). When regenerating outputs, expect the diff to include these binary/generated files, and check `git status` before committing to confirm only the intended output files changed.
