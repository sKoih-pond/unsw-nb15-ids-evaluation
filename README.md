# From Benchmark Accuracy to Operational Trust

### Evaluating ML-based intrusion detection on the UNSW-NB15 benchmark

An operational-readiness evaluation of machine-learning intrusion detection: it takes the
**UNSW-NB15** network-traffic benchmark, rebuilds it under controlled conditions, trains and
compares four classifiers, and judges them on **operational metrics rather than headline
accuracy**. The throughline: a model can post 90%+ on every standard metric and still be the
wrong thing to deploy — and only closer inspection shows why.

> Capstone project (ZZBU6601) — Sylvester Koh. The notebook is the analysis;
> this README is the map.

---

## The question

A security manager is shown a model with **96% recall** and **95% accuracy** on a recognised
benchmark. Is that enough to trust it in a Security Operations Centre? This project argues no,
and shows the specific checks that turn a reassuring benchmark number into an informed
deployment decision.

## What the analysis does

1. **Examines the provided split before trusting it.** UNSW-NB15 ships as a train/test split
   whose class balances *disagree* (attacks are 68% of the training partition but 55% of the
   test partition). Inheriting that split would confound model performance with a distribution
   shift — a small instance of the very transfer problem under study, so it's recorded as
   evidence first.
2. **Rebuilds the dataset under controlled conditions.** The partitions are combined
   (257,673 rows) and re-split **70/30, stratified on the 10-class attack category** (not the
   binary label) so every category — down to the rarest — is split in the same proportion on
   both sides.
3. **Controls for leakage.** Drops `attack_cat` (it encodes the answer) and `id` (a
   construction artefact); fits all scaling/encoding on the **training set only** so no test
   statistics leak into preprocessing.
4. **Trains 4 models × 2 conditions.** Logistic Regression, Decision Tree, Random Forest,
   HistGradientBoosting — each **unweighted (baseline)** and **class-weighted** — through one
   shared preprocessing pipeline, so the comparison is like-for-like.
5. **Scores on operational metrics.** Recall (missed attacks), precision (alert fatigue), F1,
   **PR-AUC** (chosen over ROC-AUC, which the abundant normal traffic inflates), and accuracy —
   reported last, deliberately, to show how little it discriminates here.
6. **Disaggregates the headline.** The single recall figure is broken down **per attack type** —
   where aggregate performance meets operational reality.

## Key findings

- **Everything looks strong in aggregate** — all four models clear 90% on recall, precision, F1
  and accuracy. Random Forest is the most balanced (recall ≈ 0.96, precision ≈ 0.96,
  PR-AUC ≈ 0.995). This is exactly the reassurance the project warns against taking at face value.
- **The benchmark's class balance is inverted vs. reality.** Attacks are the *majority* here;
  a real SOC sees normal traffic as the majority and attacks as the exception. A model trained
  on this composition is being trusted to a benchmark author's synthetic design, not real
  network conditions.
- **Class weighting backfires on this benchmark.** Because attacks are the majority,
  `class_weight='balanced'` protects the *normal* minority — pushing recall **down** and
  precision up, the wrong direction for attack detection. The standard fix pulls the wrong way.
- **Detection tracks distinctiveness, not frequency.** Worms (just 52 test cases) are caught at
  ~100% because propagation leaves a distinctive signature; **Fuzzers** — common, but designed
  to mimic normal traffic — are the weak point, detected 25–30 points lower. Fuzzers are the most
  *diagnostic* class for operational readiness, and weighting makes them **worse** (a 13-point
  drop for HistGradientBoosting).
- **Rare-class figures are statistically fragile.** With 52 Worms in the test set, one
  misclassification moves recall by ~2 points — a 100% catch rate can't be claimed confidently.

**Bottom line:** the headline 96% hides a model that is weakest on exactly the category designed
to evade detection. The gap surfaces only under per-class inspection and leakage discipline —
which is precisely why that inspection belongs in deployment due diligence.

## Repository contents

| File | What it is |
|---|---|
| `ids_operational_evaluation.ipynb` | The full analysis — load → examine → re-split → preprocess → train → evaluate, with reasoning at each step. Rendered outputs (tables + charts) are included, so it reads without running. |
| `requirements.txt` | Pinned library versions for reproducibility. |
| `.gitignore` | Keeps the dataset CSVs (and the usual Python/Jupyter noise) out of the repo. |

## Data — not included (download separately)

The two CSVs the notebook expects are the **UNSW-NB15** benchmark partitions and are **not
committed** (they're large, and the dataset is the authors' to distribute, not mine):

- `UNSW_NB15_training-set.csv`
- `UNSW_NB15_testing-set.csv`

Get them from the official **UNSW-NB15** dataset (UNSW Canberra Cyber) or its Kaggle mirror,
then place both CSVs next to the notebook (or update the two path variables in §2).

## Reproduce

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
# place the two UNSW-NB15 CSVs in this directory, then:
jupyter lab ids_operational_evaluation.ipynb   # Run All
```

A fixed `RANDOM_STATE = 42` makes every split and fit reproducible. Developed on
pandas 3.0.2 / numpy 2.4.4 / scikit-learn 1.8.0 (see `requirements.txt`).

## Method notes & honesty

- The **raw-capture** UNSW-NB15 also carries source/destination IP and port columns; this
  partition omits them, and that exclusion is treated as a limitation, not silently ignored.
- `proto` is one-hot encoded across all ~133 values in the baseline; grouping rare protocols is
  noted as a refinement.
- Standard library call patterns (load/split/preprocess/evaluate) follow the pandas and
  scikit-learn docs and are marked as such; the project-specific decisions — target definition,
  leakage controls, the stratified re-split, metric selection and operational interpretation —
  were developed for this project.

---

*Author: Sylvester Koh · ZZBU6601 Capstone. Code is shared for portfolio and educational
purposes; UNSW-NB15 is © its respective authors and licensed separately.*
