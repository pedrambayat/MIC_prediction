# Data

## Where the data lives

| Dataset | Location | In repo? |
|---|---|---|
| Curated peptide tables (`baseline.csv`, `master_peptide_data.csv`) | [HuggingFace: `pedbb/mic_prediction`](https://huggingface.co/datasets/pedbb/mic_prediction) | No — downloaded at runtime |
| Per-peptide AlphaFold structural features | `structural_features/` | ✅ Yes |
| Extinct-peptide MIC table (Wan et al. 2024 supplementary) | `external/` | ✅ Yes — see [`external/README.md`](external/README.md) |
| Raw AlphaFold PDB structures, per-residue CSVs | *not retained* | No |

The HuggingFace datasets are **public and require no token**:

```python
from huggingface_hub import hf_hub_download
path = hf_hub_download(repo_id="pedbb/mic_prediction",
                       filename="baseline.csv", repo_type="dataset")
```

---

## `structural_features/`

Per-peptide summary statistics computed from AlphaFold2 predicted structures. One row per
folded peptide.

| File | Rows | Description |
|---|---|---|
| `train_structural_features.csv` | 713 | DBAASP training peptides |
| `test_structural_features.csv` | 57 | Extinct peptides (Wan et al. 2024) |

No missing values in either file.

### Columns

| Column | Type | Description |
|---|---|---|
| `protein_name` | str | Folding job ID, suffixed `.result`. **Train:** the numeric DBAASP peptide `ID` (e.g. `16088.result`). **Test:** the GenBank accession of the source extinct protein (e.g. `ANT45524.result`). |
| `pLDDT_mean` | float | AlphaFold2 per-residue confidence, averaged over the chain. 0–100; higher is more confident. |
| `pLDDT_std` | float | Standard deviation of per-residue pLDDT — how uneven the confidence is along the chain. |
| `Frac_Helix` | float | Fraction of residues in the α-helical Ramachandran region: `φ ∈ [−90°, −30°]` **and** `ψ ∈ [−80°, −10°]`. 0–1. |
| `Frac_Sheet` | float | Fraction of residues in the β-sheet Ramachandran region: `φ ∈ [−180°, −90°]` **and** `ψ ∈ [90°, 180°]`. 0–1. |
| `Backbone_Rigidity` | float | Mean of the circular variances (`1 − R`, where `R` is the mean resultant length) of the φ and ψ angle distributions. Higher = more angular spread. |
| `Avg_Degree` | float | Mean number of contacts per residue in the contact map. |
| `Avg_Seq_Sep` | float | Mean sequence separation `\|i − j\|` over unique residue contacts. Higher = more non-local structure. |
| `Frac_Long_Range` | float | Fraction of contacts with sequence separation ≥ 5. |

### Which columns the model actually used

Only **five** of the eight numeric columns were fed to the model:

```python
af_cols = ['pLDDT_mean', 'Frac_Helix', 'Frac_Sheet', 'Backbone_Rigidity', 'Avg_Degree']
```

`pLDDT_std`, `Avg_Seq_Sep`, and `Frac_Long_Range` were computed but not used. They are retained
here in case they are useful later.

### Joining to the training data

Train side — strip the suffix and cast to int:

```python
df['ID'] = df['protein_name'].str.replace('.result', '', regex=False).astype(int)
merged = baseline_df.merge(df, on='ID', how='inner')
```

Note that this merge **broadcasts one structure across every measurement row for that peptide** —
a peptide measured against 20 bacterial targets receives the same structural feature values 20
times. Any evaluation that splits over rows rather than peptides will therefore see the same
structure on both sides of the split.

Test side — `protein_name` is a GenBank accession and joins to the extinct sequences by
accession, not by integer ID.

### Coverage caveat

713 training structures against 25,306 training measurements. Only peptides **< 50 residues with
complexity 0** were folded, so the structural arm of the analysis runs on a filtered subset of
the data, not the whole training set.

### How these were generated

`notebooks/02` → `03` → `04` → `05`:

1. **`02_fasta_generation.ipynb`** — filter to peptides < 50 residues, complexity 0; write FASTA.
2. **`03_alphafold2_batch_colabfold.ipynb`** — ColabFold v1.5.5 batch folding → PDB structures.
3. **`04_structure_parsing_dssp.ipynb`** — parse PDBs with BioPython + DSSP; emit per-residue
   CSVs (pLDDT, φ/ψ dihedrals, SASA, DSSP secondary structure, contact maps).
4. **`05_structural_feature_aggregation.ipynb`** — aggregate per-residue CSVs into the
   per-peptide summaries in this directory.

Intermediate artifacts from steps 2 and 3 (raw PDBs, per-residue CSVs) are not checked in.
