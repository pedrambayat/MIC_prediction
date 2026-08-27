# External data

Third-party data, **not produced by this project**. Not covered by this repository's MIT
license — each file remains under its original publisher's terms.

## `41551_2024_1201_MOESM4_ESM.xlsx`

**Supplementary Table 4** from:

> Wan, F., Torres, M.D.T., Peng, J. & de la Fuente-Nunez, C.
> *Deep-learning-enabled antibiotic discovery through molecular de-extinction.*
> **Nature Biomedical Engineering** 8, 854–871 (2024).
> https://doi.org/10.1038/s41551-024-01201-x

**Contents.** Experimentally measured MICs for peptides computationally resurrected from
extinct organisms, tested against 11 bacterial strains:

```
A. baumannii ATCC19606          E. coli ATCC11775       E. coli AIG221
E. coli AIG222                  K. pneumoniae ATCC13883 P. aeruginosa PAO1
P. aeruginosa PA14              S. aureus ATCC12600     S. aureus ATCC BAA-1556 (MRSA)
vancomycin-resistant E. faecalis ATCC700802
vancomycin-resistant E. faecium ATCC700221
```

**How it is used here.** As the out-of-distribution test set — 69 peptides × 11 strains, of
which 690 peptide–target pairs survive filtering and target-name mapping. It is *never* used
for training. Loading and cleaning happens in `notebooks/06_modeling.ipynb` §6:

- Read with `header=1` (the first row is a spanning header, not column names).
- Melt wide → long, one row per (peptide, strain).
- `N.A.` and unparseable values → **128 µg/mL**, the inactive sentinel. Note that this creates a
  large pile-up at the ceiling; see the limitations section of [`../../docs/RESULTS.md`](../../docs/RESULTS.md).
- Strip non-standard characters and keep only canonical 20-amino-acid sequences.
- Map strain names onto the target IDs learned during training; strains absent from the training
  set are dropped.

## A note on redistribution

This file is checked in for reproducibility. Springer Nature supplementary material is generally
provided under the article's own terms, which vary — an open-access article under CC BY permits
redistribution with attribution, while a subscription article may not.

**If in doubt, the safer option is to remove this file and have users download it from the
publisher.** To do that:

```bash
git rm data/external/41551_2024_1201_MOESM4_ESM.xlsx
```

then point `EXTINCT_XLSX` in `notebooks/06_modeling.ipynb` §0 at a local copy the user obtains
from https://doi.org/10.1038/s41551-024-01201-x. Nothing else in the repository needs to change.
