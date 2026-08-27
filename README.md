# Multi-modal Sequence Encoding for Antimicrobial Peptides

Predicting the **minimum inhibitory concentration (MIC)** of antimicrobial peptides (AMPs) from
sequence, physicochemical descriptors, and AlphaFold2-derived structural features — and testing
whether any of it generalizes to peptides from extinct organisms.

Final project for **CIS 5200: Machine Learning** (University of Pennsylvania).
Project write-up: **[pedrambayat.github.io/amp](https://pedrambayat.github.io/amp/)**

---

## The question

Antimicrobial resistance needs new antibiotics, and AMPs are a promising source. Machine
learning can rank candidate peptides by predicted potency — but models trained on modern,
well-studied peptides tend to fall apart on genuinely novel sequences. That distribution shift
is the whole problem in practice.

Wan et al. (2024) computationally resurrected AMPs from extinct organisms, creating a natural
out-of-distribution test set. This project asks two things:

1. Can a deep sequence encoder beat classical baselines at MIC regression?
2. **Does adding AlphaFold2 structural information improve generalization to extinct peptides?**

## Headline results

| | Spearman ρ | R² |
|---|---|---|
| In-distribution (modern peptides, 25,306 measurements) | **0.79** | **0.63** |
| Out-of-distribution (69 extinct peptides, 690 measurements) | **0.36** | — |

**The answer to question 2 is no.** AlphaFold structural features received **0.9%** of total
feature attribution and *reduced* rank correlation on the extinct set (ρ 0.201 → 0.182).
Helix fraction correlated with MIC at 0.0152 — essentially zero.

This is a negative result, and it is the most interesting thing here. AMPs are largely
disordered in solution and fold into amphipathic helices only on contact with a bacterial
membrane. AlphaFold2 predicts one static structure in isolation, at low confidence
(mean pLDDT ≈ 75, σ ≈ 7) — with a median helix and sheet fraction of exactly **0**. It is
predicting the wrong conformational state for this problem, and predicting it uncertainly.

📊 **[Full results, model comparison, and limitations →](docs/RESULTS.md)**

## Approach

**Data.** 25,306 peptide–target MIC measurements from [DBAASP](https://dbaasp.org). The raw
export is a peptide × 7,879-target matrix that is ~99% empty, so targets were filtered to those
with ≥ 200 measurements (leaving 50 bacterial targets) and the table melted to one row per
(peptide, target) interaction — target identity becomes a learned embedding rather than an
output dimension. Targets are modeled as log₂(MIC), z-scored.

**Model.** A three-arm encoder, following the APEX architecture:

```
peptide sequence ──► embedding ──► bidirectional GRU ──► attention ──┐
bacterial target ──► embedding ───────────────────────────────────────┼──► dense head ──► log₂(MIC)
11 physicochemical descriptors ──► dense block ───────────────────────┘
```

Five seeds, stratified splits, early stopping on validation loss, mean-ensembled.

**Baselines.** Random Forest, XGBoost, Nyström-SGD, linear SVM — all as regressors, for a
like-for-like comparison.

**Structural features.** AlphaFold2 (via ColabFold) on peptides < 50 residues, then five
per-peptide summaries: `pLDDT_mean`, `Frac_Helix`, `Frac_Sheet`, `Backbone_Rigidity`
(circular variance of backbone dihedrals), `Avg_Degree` (mean contact-map degree).

## Repository layout

```
notebooks/
├── 01_dataset_curation.ipynb            DBAASP API pull → curated peptide table
├── 02_fasta_generation.ipynb            filter to <50 residues → FASTA for folding
├── 03_alphafold2_batch_colabfold.ipynb  ColabFold batch folding (vendored, see below)
├── 04_structure_parsing_dssp.ipynb      PDB + DSSP → per-residue structural CSVs
├── 05_structural_feature_aggregation.ipynb  per-residue → per-peptide summaries
├── 06_modeling.ipynb                    ★ models, evaluation, OOD test — start here
└── archive/
    └── 00_full_working_notebook.ipynb   unedited working log (see below)

data/
├── structural_features/                 per-peptide AlphaFold summaries (in repo)
└── external/                            third-party supplementary data

docs/
├── RESULTS.md                           full results, comparisons, limitations
└── figures/                             all 16 figures from the original run
```

### Pipeline order

`01` → `02` → `03` → `04` → `05` → `06`. Notebooks `01`–`05` build the structural features;
**`06_modeling.ipynb` is the one to read** if you only read one. It has no dependency on `01`–`05`
at runtime — it pulls the curated datasets from HuggingFace and the structural features from
`data/`.

### About the archive notebook

[`archive/00_full_working_notebook.ipynb`](notebooks/archive/00_full_working_notebook.ipynb)
is the original 161-cell working notebook, preserved with all its outputs. It is a chronological
record rather than a presentation: it contains an abandoned first-attempt encoder, the discovery
that it was overfitting on raw MIC, the fix, and a target-scaling bug found during OOD
evaluation. `06_modeling.ipynb` is the same final code path, reordered and annotated, with the
dead ends removed.

Two changes were made to the archived file and nothing else: a header cell was added, and a
HuggingFace token that had been pasted into a markdown cell was redacted (that token has since
been revoked, and the dataset is public — no token is needed).

## Running it

```bash
pip install -r requirements.txt
jupyter lab notebooks/06_modeling.ipynb
```

**Caveats, stated plainly:**

- `06_modeling.ipynb` **has not been re-executed** since being reorganized. Outputs are stripped;
  every number quoted in it is transcribed from the original run. Treat it as a readable record,
  not a verified one-click reproduction.
- Training was done in Google Colab on a GPU. CPU training is possible but slow.
- Notebooks `01`–`05` reference Google Drive paths from the original Colab environment and
  require intermediate artifacts (raw PDB structures, per-residue CSVs) that are not checked in.
  They are included for methodological transparency, not as a turnkey pipeline.
- Notebook `03` is a Colab notebook and must run there.

Curated datasets are public on the HuggingFace Hub — **no token required**:

```python
from huggingface_hub import hf_hub_download
path = hf_hub_download(repo_id="pedbb/mic_prediction", filename="baseline.csv", repo_type="dataset")
```

[huggingface.co/datasets/pedbb/mic_prediction](https://huggingface.co/datasets/pedbb/mic_prediction)

## Attribution

- **ColabFold** — `03_alphafold2_batch_colabfold.ipynb` is an unmodified copy of ColabFold v1.5.5
  AlphaFold2 Batch, included so the exact folding configuration is on record.
  Mirdita et al., *ColabFold: making protein folding accessible to all*, **Nature Methods** (2022).
  [github.com/sokrypton/ColabFold](https://github.com/sokrypton/ColabFold)
- **DBAASP** — training data source. Pirtskhalava et al., *Nucleic Acids Research* (2021).
  [dbaasp.org](https://dbaasp.org)
- **APEX architecture and extinct-peptide test set** — Wan et al., *Deep-learning-enabled
  antibiotic discovery through molecular de-extinction*, **Nature Biomedical Engineering** (2024).
  [doi.org/10.1038/s41551-024-01201-x](https://doi.org/10.1038/s41551-024-01201-x)
  See [`data/external/README.md`](data/external/README.md) for the supplementary file's provenance.
- **modlAMP** — physicochemical descriptors. Müller et al., *Bioinformatics* (2017).

## License

Code released under the MIT License — see [LICENSE](LICENSE). This does **not** extend to
third-party data in `data/external/`, which remains under its original publisher's terms.
