# Collector-state-aware feature selection for IoT gateway flow monitoring

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.24250391.svg)](https://doi.org/10.5281/zenodo.24250391)

Open-source implementation and complete results for state-aware two-pass feature selection
on the RT-IoT2022 dataset.

A flow collector on an IoT gateway keeps one record per active flow and updates it on every
packet. Its memory and per-packet work are set by the **state** behind the flow features, and
features share that state. This repository selects collector state under a byte budget,
instead of selecting a number of classifier inputs, and measures what the selected state costs.

## What is included

| Path | Content |
|---|---|
| `Gateway_State_Selection.ipynb` | The complete pipeline, executed, with all outputs |
| `results/tables/` | Every number reported, as CSV and JSON |
| `results/figures/` | Figures (PDF and PNG) |
| `results/folds/` | Per-fold out-of-fold predictions and selection records (15 folds) |
| `results/fold_run_log.txt` | Progress log of the cross-validated evaluation |

The notebook contains:

1. **Collector-state cost model.** 72 per-flow state primitives (384 bytes) that reproduce
   the 79 Zeek FlowMeter flow-statistics features of RT-IoT2022, and the map from each
   feature to the primitives it needs (`results/tables/state_primitives.csv`,
   `results/tables/feature_state_map.csv`).
2. **Sliceable C collector.** A generator that emits a streaming collector for any subset of
   primitives, a reference transcription of the Zeek FlowMeter script, and an exactness
   check: the full collector matches the reference, and every evaluated subset reproduces the
   full collector's feature values bit for bit.
3. **Two-pass selection.** Pass 1 computes class-specific SHAP attributions from a random
   forest trained on all features. Pass 2 adds, at each step, the state with the largest
   measured out-of-bag detection gain per byte, using the attributions only to shortlist
   candidates, and stops when the estimate is within a tolerance of the full-state value.
4. **Evaluation.** Three repetitions of grouped, stratified five-fold cross-validation
   against cost-blind rankings (SHAP, impurity, mutual information), cost-only and
   gain-only ablations, a unit-cost variant, random orderings, and the fixed top-20
   feature design; per-class recall; paired bootstrap intervals; identity-field analysis;
   and per-packet timing of every generated collector.

## Main results

Pooled out-of-fold values, mean over three repetitions (`results/tables/headline.json`).

| Configuration | State per flow | Macro-F1 | Benign false alarms | State update per packet |
|---|---|---|---|---|
| Full state | 384 B | 0.953 | 1.6% | 75.9 ns |
| Top-20 SHAP features | 88 B | 0.950 | 1.7% | 35.0 ns |
| Selected state (tolerance 0.01) | 9.3 B (7–17) | 0.944 | 2.7% | 12.1 ns |

To come within 0.01 of the full-state macro-F1, the state-aware path needs 8–16 bytes, a
cost-blind SHAP ranking 64–96 bytes, and random feature orderings 192 bytes or more
(`results/tables/method_summary_by_seed.csv`).

Timings are medians of five interleaved rounds on a shared virtual machine, replaying
synthetic packets through the state-update routine only; packet capture, parsing, flow
lookup and expiry are not included. Compare configurations within one run rather than
across machines.

## Reproducing the results

1. Download the RT-IoT2022 CSV from the UCI Machine Learning Repository:
   https://doi.org/10.24432/C5P338
2. **Google Colab:** place the file at `MyDrive/Datasets/RT_IOT2022.csv`, open the notebook
   and run all cells. **Elsewhere:** set the environment variables `GW_DATA` (CSV path),
   `GW_WORK` (scratch folder) and `GW_OUT` (results folder), then run the notebook.
3. No GPU is needed. The full evaluation takes about one hour on two CPU cores and prints
   one progress line per fold. Setting `QUICK_TEST = True` in the first cell runs a short
   installation check.
4. To regenerate tables and figures without repeating the evaluation, copy
   `results/folds/` into the results folder before running: completed folds are restored
   from these files.

Requirements: Python 3 with scikit-learn, pandas, NumPy, SciPy, matplotlib and SHAP, and
`gcc` for the collector benchmark. The versions, CPU model and SHA-256 of the dataset used
for the reported run are in `results/tables/run_metadata.json`.

## Scope

All results concern one public dataset of completed flow records. Folds are grouped by
identical feature vectors; the dataset does not provide capture-session identifiers, so
correlated flows from one capture can appear in both training and test folds. Source port,
destination port and the service label are excluded from the main analysis because they act
as shortcuts on this benchmark; their effect is reported separately
(`results/tables/input_groups_by_seed.csv`).

## Data

RT-IoT2022 is distributed by the UCI Machine Learning Repository and is not included here.
Its flow features are produced by the Zeek FlowMeter script
(https://github.com/zeek-flowmeter/zeek-flowmeter).

## Archive

Version 1.0.0 is archived on Zenodo: https://doi.org/10.5281/zenodo.24250391

## License

MIT; see `LICENSE`.
