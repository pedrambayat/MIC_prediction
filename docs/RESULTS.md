# Results

All numbers transcribed from the original Colab run, recorded in
[`../notebooks/archive/00_full_working_notebook.ipynb`](../notebooks/archive/00_full_working_notebook.ipynb).

---

## 1. In-distribution performance

25,306 peptide–target MIC measurements from DBAASP, 50 bacterial targets. Targets are
log₂(MIC), z-scored. Five seeds (0, 42, 123, 2024, 999), each with its own train/validation
split stratified by target ID, early stopping on validation loss.

| Model | R² | MSE | Spearman ρ |
|---|---|---|---|
| **Hybrid NN** (sequence + target + physicochemical) | **0.6295 ± 0.0239** | 0.3706 | 0.7777 |
| Hybrid NN, 5-seed ensemble | 0.6257 ± 0.0095 | 0.3741 | — |
| Sequence NN (sequence + target only) | 0.6189 ± 0.0139 | 0.3814 | **0.7876** |
| Random Forest | ~0.61 | — | 0.7865 |
| XGBoost | ~0.43 | — | 0.7749 |
| Linear SVM | 0.1691 | — | — |
| Nyström-SGD | 0.1493 | — | — |

**Reading this table.** The three top models are within noise of each other on R² — the
Random Forest, given engineered physicochemical descriptors, essentially matches a neural
network that has to learn them from raw sequence. The linear models are far behind, confirming
the structure–activity relationship is nonlinear.

The ensemble scores slightly *below* the best single model but with roughly one quarter the
seed-to-seed variance (± 0.0095 vs ± 0.0239). The ensemble was used for final evaluation on
that basis.

For context, the APEX paper reports R² = 0.37 and Pearson r = 0.62 on its own split. These are
not directly comparable — see limitations below.

**Figures**
- [Single-seed parity plot](figures/04_seq_encoder_parity_seed42.png)
- [Sequence encoder: learning curves and parity, 5 seeds](figures/05_seq_encoder_curves_and_parity_5seeds.png)
- [Sequence encoder: parity and residuals](figures/06_seq_encoder_parity_residuals.png)
- [Tree baselines](figures/07_baselines_rf_xgb_comparison.png) · [baseline parity](figures/08_baselines_parity_plots.png)
- [Hybrid 3-arm training curves](figures/09_hybrid_3arm_training_curves.png)
- [Ensemble curves and parity](figures/10_ensemble_curves_and_parity.png)

Across all parity plots the model hedges — over-predicting low MICs and under-predicting high
ones, i.e. regressing toward the mean. For candidate screening, rank order matters more than
absolute potency, which is why Spearman ρ is reported alongside R².

---

## 2. Out-of-distribution: 69 extinct peptides

69 peptides computationally resurrected from extinct organisms (Wan et al. 2024) × 11 bacterial
strains = 690 peptide–target pairs. `N.A.` MICs were encoded as 128 µg/mL (inactive).
`SYNTHESIS TYPE` was dropped from the feature set because it is unavailable for this cohort.

| Model | Spearman ρ |
|---|---|
| **Hybrid NN, 3-seed ensemble** | **0.3564** |
| XGBoost | 0.3305 |
| Random Forest | 0.3300 |

Per-seed hybrid results: ρ = 0.3866 (seed 42), 0.3002 (seed 123), 0.2465 (seed 2024).

### The drop

| | Spearman ρ |
|---|---|
| In-distribution | 0.7876 |
| Out-of-distribution | 0.3564 |

**Rank correlation falls by more than half.** The models are better than chance but nowhere near
usable for prospective screening, and the spread across seeds (0.25–0.39) is wide for a
690-point evaluation. The neural network's edge over the tree baselines (0.356 vs 0.330) is
smaller than that seed-to-seed spread and should not be treated as a real advantage.

### A scaling bug worth recording

The first OOD run produced implausible MIC predictions. The model was trained on z-scored
log₂ targets, but the extinct-set targets had not been put on the same scale. Applying the
identical transform moved ensemble ρ from **0.3025 → 0.3564**.

- [Before: unscaled predictions](figures/11_extinct_predictions_unscaled.png)
- [After: 3-seed ensemble parity on z-scored targets, ρ = 0.356](figures/12_extinct_ensemble_parity_zscored.png)
- [Extinct-set tree baselines](figures/14_extinct_baselines_rf_xgb.png)

The corrected parity plot also shows the sentinel problem directly: true log₂(MIC) piles up
heavily at 7 (= 128 µg/mL, "inactive"), and predictions there scatter across the full range.

---

## 3. Does AlphaFold structure help? No.

Five per-peptide structural features from AlphaFold2 predictions:

| Feature | Meaning |
|---|---|
| `pLDDT_mean` | AlphaFold's per-residue confidence, averaged |
| `Frac_Helix` | Fraction of residues in the α-helical Ramachandran region (φ/ψ windows) |
| `Frac_Sheet` | Fraction of residues in the β-sheet Ramachandran region (φ/ψ windows) |
| `Backbone_Rigidity` | Mean circular variance of the φ and ψ angle distributions |
| `Avg_Degree` | Mean residue contact-map degree |

**Method note.** This comparison uses a Random Forest trained on *frozen embeddings* extracted
from the already-trained hybrid network, concatenated with structural features. It is a
different model family from the end-to-end network in sections 1–2, so its absolute numbers are
not comparable to those above — only the three variants below are comparable to each other.

| Model | R² | Spearman ρ |
|---|---|---|
| Sequence only | −0.6985 | **0.2010** |
| Structure only | −1.0162 | 0.1466 |
| Sequence + structure | −0.6719 | 0.1824 |

Adding structure improves R² marginally (−0.699 → −0.672) while making **rank correlation
worse** (0.201 → 0.182). Since rank order is the quantity of interest, this is not an
improvement. All three R² values are negative — every variant is worse than predicting the
training mean.

> ⚠️ **The notebook prints `✅ SUCCESS: Hybrid model shows improvement!` here. Disregard it.**
> The condition is `r2_hy > r2_seq or rho_hy > rho_seq` — an `or`, so the R² gain alone fires
> the message even though ρ regressed. The table above is the honest read.

**Feature attribution: sequence 99.1%, structure 0.9%.**

[Results and feature importance](figures/15_alphafold_hybrid_results_and_importance.png)

### Why structure contributed so little

Correlation between helix fraction and MIC: **0.0152**.
[Helix fraction vs MIC](figures/16_helix_fraction_vs_mic.png)

Distribution of the structural features across 2,828 peptides:

| | pLDDT_mean | Frac_Helix | Frac_Sheet | Backbone_Rigidity | Avg_Degree |
|---|---|---|---|---|---|
| mean | 74.78 | 0.116 | 0.034 | 0.134 | 4.75 |
| std | 7.20 | 0.278 | 0.104 | 0.126 | 1.30 |
| min | 28.73 | 0.000 | 0.000 | 0.003 | 1.00 |
| 25% | 71.43 | 0.000 | 0.000 | 0.036 | 4.00 |
| **50%** | 74.51 | **0.000** | **0.000** | 0.097 | 4.40 |
| 75% | 78.42 | 0.000 | 0.000 | 0.196 | 5.89 |
| max | 94.80 | 1.000 | 1.000 | 0.782 | 7.00 |

Two observations explain the result:

1. **Median helix and sheet fraction are exactly 0.** Most predicted structures have no assigned
   secondary structure at all.
2. **The features barely vary.** Mean pLDDT ≈ 75 with σ ≈ 7 — a narrow band of moderate-to-low
   confidence across the whole set. A feature that doesn't vary can't discriminate.

This is mechanistically expected rather than surprising. AMPs are typically **disordered in
solution** and adopt amphipathic helical conformations only **upon contact with a bacterial
membrane**. AlphaFold2 predicts a single static structure for an isolated chain — close to the
wrong conformational state for this problem, and it flags its own uncertainty via low pLDDT.
The properties that actually drive membrane disruption (membrane-bound conformation,
hydrophobic moment, insertion depth, oligomerization) are not captured by these five summaries.

---

## Limitations

**1. Peptide-level leakage in the in-distribution split.** The train/validation split is taken
over the *melted* table — one row per (peptide, target) measurement — stratified by target ID.
Because each peptide contributes roughly one row per bacterial target, the same sequence appears
in both train and validation with different targets. The encoder can memorize a peptide from its
training rows and be scored on it again at validation.

So **R² = 0.63 / ρ = 0.79 is a *seen-peptide, unseen-target* result, not a held-out-peptide
result.** It is a legitimate metric, but it is not what "in-distribution generalization"
normally implies, and it inflates the apparent size of the in-distribution → OOD gap. A
`GroupShuffleSplit` on `SEQUENCE`, or the cluster-based split used by the APEX paper, would
give the comparable number. This is also why the R² = 0.63 here should not be read as beating
the paper's R² = 0.37.

*(This concern was flagged during the original work — see the analysis cell following the
five-seed sequence encoder run in the archive notebook.)*

**2. Small and imbalanced OOD set.** 690 measurements over 69 peptides, with inactive MICs
encoded as a single sentinel value (128 µg/mL). Many true MICs pile up at that ceiling, which
compresses the range Spearman is computed over. Seed-to-seed variance of 0.25–0.39 reflects
how little signal 690 points provide.

**3. Two model families across sections.** Sections 1–2 use the end-to-end hybrid network.
Section 3 uses a Random Forest on frozen embeddings. Numbers are comparable *within* sections,
not across them. The two OOD Spearman figures that appear in this document — 0.356 and 0.201 —
come from these two different setups on the same peptides.

**4. Structural features cover a subset.** Only peptides < 50 residues with complexity 0 were
folded (714 training, 57 test structures in `data/structural_features/`), against 25,306
training measurements. The structural arm of the comparison therefore runs on a filtered subset.

**5. Single dataset, single source.** All training data is DBAASP. MIC measurements aggregated
across labs and protocols carry substantial assay-to-assay variance that is not modeled here.

## What would be worth trying next

- Re-run the in-distribution evaluation with a peptide-grouped or sequence-identity-clustered
  split, to get a leakage-free number comparable to published results.
- Replace static AlphaFold summaries with membrane-context descriptors: hydrophobic moment,
  amphipathicity, helical-wheel geometry — cheap to compute and mechanistically closer.
- Try a pretrained protein language model (ESM-2) as the sequence arm; the 99.1% sequence
  attribution suggests that is where the remaining headroom is.
- Treat the 128 µg/mL ceiling as censored data and fit a censored-regression objective rather
  than a point target.
