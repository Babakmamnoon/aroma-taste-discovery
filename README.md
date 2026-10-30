# aroma-taste-discovery

**AI-driven discovery of aroma and taste molecules: a merged, provenance-tracked structure–odor / structure–taste database and the models built on it.**

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Babakmamnoon/aroma-taste-discovery/blob/main/aroma_taste_FULL_colab.ipynb)

Public structure–odor data is small, scattered across a dozen sources, and each source speaks its own vocabulary. Most published models train on one of them. This project merges nine aroma sources, two psychophysical panel studies and two taste databases into a single curated database of **9,775 molecules**, then trains and stress-tests models on it the way an industrial team would before trusting them to triage synthesis candidates.

Everything runs end to end in one notebook on a free Colab CPU runtime, in roughly 45–75 minutes.

---

## What's in the database

| | Molecules | Labels |
|---|---|---|
| Odor | 6,166 | 138 harmonized notes (GS-LF vocabulary) |
| Taste | 3,660 | sweet, bitter, umami, sour, salty, tasteless + explicit negatives |
| Both | 236 | |
| Panel ratings | 540 | continuous intensity/pleasantness ratings |

Molecules are standardized with RDKit (`Cleanup` → `FragmentParent` → `Uncharger`) and merged on the **InChIKey connectivity block**, with every stereo variant preserved in its own table — because enantiomers can smell different, and the database should not quietly decide otherwise.

## Results

All numbers are on a held-out **fingerprint-cluster split** that no tuning touched.

| Task | Metric | Result | Baseline |
|---|---|---|---|
| Aroma, 138 notes | macro AUROC / AUPRC | **0.848 / 0.313** | chance AUPRC 0.042; Tanimoto *k*-NN 0.770 / 0.219 |
| Taste: sweet | AUROC | **0.87** | |
| Taste: bitter | AUROC | **0.92** | |
| Sweetness potency | grouped-CV R² | **0.62** | RMSE 0.70 log units |

Best-predicted notes: sulfurous (0.78 AUPRC), onion (0.72), garlic (0.70), amber (0.59).

Two findings worth more than the headline numbers:

* **A random split scores the same as the cluster split** in macro AUROC here (−0.003). Leakage shows up per molecule instead: test molecules with a near-twin in training are ranked far better (LRAP 0.57) than novel ones (0.39). Macro AUROC hides it.
* **The model recovers classical structure–odor rules on its own.** A macrocycle raises predicted musk 46-fold, a thiol raises sulfurous 14-fold, a pyrazine raises roasted 10-fold — each checked against the ratio actually observed in the data.

## Notebook contents

| Part | Content |
|---|---|
| 1 | Data layer: acquisition, standardization, vocabulary harmonization, the merged SQLite database, EDA |
| 2 | Featurization (Morgan counts, chirality-aware fingerprints, MACCS, RDKit 2D, F&F functional groups), chemical space, leakage-safe splits |
| 3 | Models: Tanimoto *k*-NN, random forest, LightGBM one-vs-rest, multi-task net, grouped CV, calibration, applicability domain, taste and sweetness |
| 4 | Interpretability (substructure attribution vs known SOR rules, atom-level maps), the chirality blind spot, a virtual screen with AD gating, deployment in a DMTA cycle |

## Running it

**On Colab** — click the badge. Run the two setup cells at the top, then *Runtime > Run all*. CPU is enough; no GPU is used. The first setup cell restarts the runtime if it installs anything, which is deliberate: a `pip install` that upgrades pandas under a live kernel leaves it half-replaced and fails much later in an unrelated cell.

Colab deletes the working directory when a runtime recycles. To keep the database and the trained models, mount Drive and `%cd` into a folder there before running anything:

```python
from google.colab import drive; drive.mount('/content/drive')
%cd /content/drive/MyDrive/aroma-taste-discovery
```

**Locally**

```bash
pip install -r requirements.txt
jupyter lab aroma_taste_FULL_colab.ipynb
```

**One manual step:** ChemTastesDB. Zenodo rate-limits scripted downloads, so download `ChemTastesDB_database.xlsx` from [the Zenodo record](https://zenodo.org/records/15051366) once and save it as `data/raw/chemtastesdb__ChemTastesDB_database.xlsx`. On Colab the second setup cell prompts you to upload it. Every other source downloads and caches automatically, with a SHA-256 manifest, so a rerun never depends on a third-party server being up.

## Repository layout

```
aroma-taste-discovery/
├── README.md
├── requirements.txt
├── aroma_taste_FULL_colab.ipynb              # all four parts, one runtime  ← start here
└── notebooks/
    ├── aroma_taste_part1_data_layer.ipynb
    ├── aroma_taste_part2_features_splits.ipynb
    ├── aroma_taste_part3_models.ipynb
    └── aroma_taste_part4_interpretation_screening.ipynb
```

The split notebooks are the same material in four pieces, useful for presenting. They must be run in order **in one session**, since each reads what the previous one wrote. The merged notebook is the one to run.

Generated at runtime, and not committed: `data/`, `src/aroma_features.py`, `models/`, `figures/`, `results/`.

Suggested `.gitignore`:

```gitignore
data/
models/
figures/
results/
src/
.ipynb_checkpoints/
__pycache__/
```

## Data sources and licences

| Source | Content | Licence |
|---|---|---|
| OpenPOM GS-LF | 4,983 molecules × 138 odor labels | MIT |
| Pyrfume (Leffingwell, GoodScents, Arctander, Sigma, IFRA, Flavornet, AromaDb) | odor descriptors | per-archive, mostly CC BY |
| Keller 2016, Dravnieks 1985 | psychophysical panel ratings | CC BY |
| ChemTastesDB v2.1 | 4,078 tastant records | CC BY 4.0 |
| SweetenersDB v2.0 | sweetness potency (logSw) | per repository |
| FlavorDB2 | food–molecule links, flavor profiles | CC BY-NC-SA — **non-commercial** |

Each label in the database keeps the source and the raw text it came from, so any number can be traced back.

**BitterDB is deliberately excluded.** Its terms reserve all rights, including use for AI training. The bitter labels here come from ChemTastesDB instead.

Note the FlavorDB2 licence before reusing the merged database commercially. The notebook can be run with the FlavorDB2 scraper disabled (`RUN_FLAVORDB2_SCRAPE = False`) if you need a cleanly licensed subset.

## Known limitations

Stated plainly, because they are the honest part of the work:

* **The models are blind to chirality.** 414 stereoisomer pairs are described by the same source and 81% of them get different descriptions, yet the achiral features give both members identical predictions. Carvone is the standard example. Fixing it needs stereo-resolved labels, not just chirality-aware fingerprints — the data is the limiting factor, not the representation.
* **Rare notes are weak.** Performance tracks the number of training positives per note, and the commercially valuable notes (musk, amber) are the data-poor ones. That argues for targeted data generation over a bigger model.
* **Labels are noisy.** The median inter-source Jaccard agreement between sources describing the same molecule is 0.25, which puts a ceiling on achievable accuracy.
* **Predictions outside the applicability domain are extrapolation** and are flagged as such rather than silently reported.

## Licence

Code: MIT. The merged data inherits the terms of its sources — see the table above.
