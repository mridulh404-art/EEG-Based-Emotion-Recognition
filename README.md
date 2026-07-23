# EEG-Based Emotion Recognition (GAMEEMO Dataset)

Subject-independent emotion classification from raw EEG signals, using
band-power features and classical machine learning, validated with true
leave-one-subject-out cross-validation.

## Project Overview

- **Dataset:** GAMEEMO (Alakus et al., 2020) — 28 subjects, 14-channel
  Emotiv EPOC+ EEG, 4 emotion classes elicited via 4 different game genres
- **Data Scope:** Full raw EEG signal, 128 Hz, 5-minute recordings per
  subject per game (~2.2 hours of recording total)
- **Preprocessing:** Butterworth bandpass filtering (theta/alpha/beta/gamma),
  5-second non-overlapping windows, per-subject z-score normalization
- **Feature Extraction:** Band power across 14 channels x 4 frequency bands
  (56 features per window)
- **Objective:** Classify 4 emotional states (boring, calm, horror, funny —
  spanning the arousal/valence quadrants) in a way that generalizes to
  unseen subjects, not just unseen recordings

---

## Methodology

### 1. Data Audit
- Verified raw signal structure, channel names, sampling rate (~127.5 Hz
  measured vs. 128 Hz nominal), and label mapping before writing any pipeline
- Identified and corrected two source-data packaging inconsistencies
  (a nested duplicate subfolder, and one subject's files using a different
  naming convention)

### 2. Signal Processing
- 4th-order Butterworth bandpass filters (theta 4-8Hz, alpha 8-13Hz,
  beta 13-30Hz, gamma 30-45Hz), applied via zero-phase filtering (`filtfilt`)
- Validated on real signal before scaling to the full dataset

### 3. Feature Extraction
- 5-second non-overlapping windows (chosen specifically to avoid the
  near-duplicate/overlapping-window leakage documented below)
- Band power (mean squared amplitude) per channel per band
- Per-subject z-score normalization, to correct for inter-individual
  differences in absolute EEG amplitude unrelated to emotional state

### 4. Classification & Validation
- Three models compared: SVM (linear), Random Forest (bagged trees),
  small Neural Network
- **Two validation protocols compared directly**: naive random 5-fold CV
  vs. true leave-one-subject-out (28 folds) — see Key Findings below for
  why this comparison matters

### 5. Evaluation
- Confusion matrix, per-class precision/recall/F1
- Statistical significance testing (paired t-tests vs. chance and
  between models)

### 6. Robustness Testing
- Synthetic artifact injection (broadband noise across 4 of 14 channels)
  to test whether a simple amplitude/variance-based rejection method
  could recover a broken classification

---

## Key Findings

- **A dataset-selection finding, before any modeling began:** an initial
  candidate dataset (Bird et al. "Feeling Emotions") was diagnosed and
  rejected after discovering 95% of its rows shared their nearest
  neighbor's label — strong evidence of overlapping-window leakage from
  its pre-extraction process, on top of having only 2 subjects. See
  `data_audit.md` for the full investigation.
- **Random-fold CV meaningfully inflates accuracy relative to true
  leave-one-subject-out validation**, across every model tested (SVM:
  +10.7 points, Random Forest: +5.9 points, Neural Net: +8.3 points).
  Random Forest's smaller gap is itself informative — it's both the
  most accurate model AND the one whose naive-validation number was
  most trustworthy.
- **Per-subject normalization is the single most important design
  choice** — more than band selection. Without it, accuracy is barely
  above chance (39%); with it, accuracy triples (81%). See the ablation
  study below.
- **No dominant confusion axis** in the best model's errors — mistakes
  don't systematically cluster along either the arousal or valence
  dimension, suggesting the feature set captures genuine 4-way emotional
  structure rather than only one underlying dimension.
- **Per-subject accuracy varies widely (31%–97%)** — some individuals'
  EEG separates emotional states far more cleanly than others', a real
  pattern rather than noise (see per-subject chart below).
- **A single-channel-cluster artifact can break classification, and a
  simple rejection method can recover it** — demonstrated on a real
  example, with the honest caveat that this only works when *some*
  clean channels remain.


## Results

### Signal Processing Validation

![Phase 1 sanity check](results/figures/phase1_sanity_check_S01_G1_AF3.png)

Raw EEG filtered into alpha (8-13Hz) and beta (13-30Hz) bands for one
real channel/trial, confirming the filtering pipeline produces
physiologically plausible output before scaling to the full dataset.


### Model Comparison — Random-Fold vs. Leave-One-Subject-Out

![Model comparison](results/figures/model_comparison_bar.png)

| Model | Random 5-Fold CV | Leave-One-Subject-Out | Gap |
|---|---|---|---|
| SVM (linear) | 0.5834 ± 0.0122 | 0.4765 ± 0.0953 | 0.1068 |
| Random Forest | 0.8627 ± 0.0089 | **0.8037 ± 0.1166** | 0.0590 |
| Neural Network | 0.7110 ± 0.0101 | 0.6276 ± 0.1010 | 0.0834 |

Random Forest is both the best-performing model and the most robust to
the validation-leakage effect seen across all three.


### Confusion Matrix & Per-Subject Performance (Random Forest, LOSO)

![Confusion matrix](results/figures/confusion_matrix_rf_loso.png)

**Per-class metrics:**

| Class | Precision | Recall | F1 |
|---|---|---|---|
| boring | 0.8216 | 0.8251 | 0.8233 |
| calm | 0.7787 | 0.8202 | 0.7989 |
| funny | 0.8168 | 0.7906 | 0.8034 |
| horror | 0.8242 | 0.8033 | 0.8136 |

Macro-average F1: **0.8098**

**Statistical significance:**

| Test | t-statistic | df | p-value |
|---|---|---|---|
| RF vs. chance (25%) | 25.12 | 27 | 2.93 × 10⁻²⁰ |
| RF vs. SVM (paired) | 14.55 | 27 | 2.68 × 10⁻¹⁴ |

![Per-subject accuracy](results/figures/per_subject_accuracy_bar.png)

Accuracy ranges from 31% (subject 26) to 96.6% (subject 17) — well
above chance for the great majority of subjects, but with meaningful
individual variation worth investigating further (see Future Work).

### Validated Protocol (Train/Validation/Test Split)
To confirm the reported accuracy wasn't dependent on an untuned
hyperparameter choice, Random Forest was re-evaluated using proper
3-way separation: within each LOSO fold, the 27 non-test subjects were
further split by subject (80/20) into train and validation sets: the
validation set selected the number of trees (25/50/100/150) via grid
search, and the held-out test subject was touched only once, for final
evaluation.

| Metric | Value |
|---|---|
| Mean test accuracy | 0.8131 ± 0.1110 |
| Original (untuned, 100 trees) | 0.8037 ± 0.1166 |
| Most frequently selected NumTrees | 150 |

The near-identical result confirms the original 100-tree configuration
was already close to optimal, rather than accuracy depending on a
lucky hyperparameter choice. See
`results/tables/validated_split_results.csv`.

### Deep Learning Comparison (LSTM on Raw Signal)

To test whether a deep sequence model could outperform feature-engineered
classical methods, a small regularized LSTM (32 hidden units, dropout 0.4)
was trained directly on raw 14-channel EEG sequences (not hand-crafted
features), using a subject-level train(20)/validation(4)/test(4) split.
Full 28-fold LOSO was not computationally practical for a deep model on
CPU within reasonable time, so this uses a single held-out split instead
-- still subject-independent, but a lighter-weight protocol than the
LOSO validation used for the other three models.

| Model | Input | Test Accuracy (same 4 held-out subjects) |
|---|---|---|
| LSTM (32 units, dropout 0.4) | Raw signal | 0.4417 |
| Random Forest | Band-power features | **0.8125** |

The LSTM's validation accuracy plateaued around 34-40% within 5 epochs
before early stopping, rather than continuing to improve -- consistent
with insufficient independent training subjects (20) for a deep model
to learn generalizable temporal patterns from raw signal. This result
reinforces the ablation finding above: for small-N EEG emotion datasets,
domain-informed feature engineering (band power + per-subject
normalization) is more effective than raw-signal deep learning, a
pattern consistent with prior literature on data-constrained EEG
classification. See `results/tables/lstm_vs_rf_comparison.csv` and
`results/tables/lstm_subject_split.csv`.

### Ablation Study — What Actually Mattered

![Ablation study](results/figures/ablation_bar.png)

| Feature Set | Mean LOSO Accuracy | Std | # Features |
|---|---|---|---|
| Alpha + Beta only | 0.3809 | 0.2044 | 28 |
| All 4 bands, no normalization | 0.3926 | 0.1982 | 56 |
| All 4 bands + per-subject normalization | **0.8108** | 0.1132 | 56 |

Per-subject normalization accounts for the overwhelming majority of
model performance — a ~42-point jump — while adding theta/gamma bands
alone contributed roughly 1 point.

---
### Multi-Level Feature Importance Analysis

**Band-wise:**

![Band importance](results/figures/band_importance.png)

| Band | Total OOB Permuted Importance |
|---|---|
| Beta (13-30Hz) | 22.09 |
| Theta (4-8Hz) | 18.10 |
| Gamma (30-45Hz) | 17.67 |
| Alpha (8-13Hz) | 16.57 |

Beta leads modestly — plausible given its established association with
active cognitive engagement and alertness, both relevant to gameplay-
induced emotional states. All four bands contribute meaningfully (no
single band dominates), consistent with the ablation finding above that
adding theta/gamma provided only a small accuracy gain over alpha+beta
alone.

**Channel-wise:**

![Channel importance](results/figures/channel_importance.png)

Top individual channels: T8, P7, O2, F8 — notably, three of the four
top performers are *not* frontal electrodes, despite frontal regions
being commonly emphasized in EEG-emotion literature.

**Region-wise (corrected for channel count):**

![Region importance corrected](results/figures/region_importance_corrected.png)

| Region | Avg. Importance per Channel | # Channels |
|---|---|---|
| Temporal | 6.04 | 2 |
| Parietal | 5.74 | 2 |
| Occipital | 5.46 | 2 |
| Frontal | 4.99 | 8 |

**Important methodological note:** a naive sum of importance across all
14 channels' regions makes Frontal appear dominant (39.9 total) simply
because it contains 8 of the 14 channels, versus 2 each for the other
three regions. Normalizing to *average importance per channel* reverses
this — Temporal, Parietal, and Occipital each edge out Frontal on a
per-electrode basis. This is a useful reminder that unequal channel
coverage across regions (a property of the Emotiv EPOC+ montage itself,
not a modeling choice) can distort naive regional comparisons if not
corrected for.

**Subject-wise:** see the per-subject accuracy analysis above.

---
### Artifact Robustness

![Artifact robustness](results/figures/phase5_artifact_robustness.png)

| Condition | Prediction | Correct? |
|---|---|---|
| Clean window | boring | ✓ |
| Corrupted (broadband noise, 4/14 channels) | calm | ✗ |
| Corrupted + artifact rejection applied | boring | ✓ |

A basic amplitude/variance-threshold rejection method, applied per
1-second sub-window, recovered the correct classification after a
realistic multi-channel artifact broke it.

---

### Computational Cost

| Stage | Time |
|---|---|
| Model training (6,372 rows, 56 features) | 6.3 sec |
| Feature extraction per 5-sec window (56 features) | 314 ms |
| Classification per window | 126 ms |
| **End-to-end per window** | **441 ms (~11x faster than real-time)** |

Measured on desktop hardware (MATLAB R2024b); not tested on embedded
hardware. See `results/tables/runtime_measurements.csv`.


## Literature Comparison

| Study | Method | Validation | 4-Class Accuracy |
|---|---|---|---|
| Genetically-optimized PDPL (2023) | Dictionary pair learning + GA | Subject-independent | 49.0% |
| CLBP + CSA + XGBoost (2025) | Chaotic local binary pattern + gradient boosting | "Subject-independent" | 99.2%* |
| **This work** | Band power + per-subject normalization + Random Forest | **True LOSO (28 folds)** | **81.1%** |

*The 99.2% figure warrants caution — this project's own findings (see
Key Findings above) show naive validation protocols can substantially
inflate reported accuracy on windowed EEG features; the full methodology
of that result was not available to confirm whether similar leakage was
controlled for. See `results/tables/literature_comparison.md` for the
full note.


## Conclusion

This project demonstrates that 4-class emotion recognition from raw EEG
is achievable with a classical, computationally lightweight pipeline
(band-power features + Random Forest) at 81.1% accuracy under genuinely
subject-independent validation. The work's central methodological
contribution is diagnostic rather than architectural: identifying and
quantifying two ways EEG-emotion pipelines commonly overstate performance
— overlapping-window leakage and naive random-fold cross-validation — and
showing that per-subject normalization, not model complexity or band
selection, is what actually closes the resulting gap.

## Limitations

- **Dataset size and diversity**: 28 subjects is modest by machine
  learning standards, though comparable to or larger than many published
  EEG-emotion studies. All subjects were recruited under a single
  protocol (Emotiv EPOC+, 4 specific games); findings may not generalize
  to other EEG hardware, electrode counts, or emotion-elicitation methods
  (e.g. film clips, music, real-world stressors) without further
  validation.
- **Deep learning comparison used a lighter validation protocol.** The
  LSTM comparison used a single subject-level train/validation/test split
  rather than full leave-one-subject-out, due to CPU training time
  constraints. This makes the LSTM result directionally informative but
  statistically weaker evidence than the LOSO-validated results for
  SVM/RF/NN. A GPU-accelerated or more heavily optimized LSTM might
  narrow the gap, though the magnitude of underperformance observed
  (44.2% vs. 81.25% on identical test subjects) makes a full reversal
  under LOSO unlikely.
- **Ground-truth emotion labels are inferred from game genre, not
  continuously self-reported.** Each 5-minute recording is labeled by
  the intended emotional target of its game (e.g. "horror" for Slender:
  The Arrival), not by moment-to-moment subjective report. The dataset
  does include post-hoc SAM (Self-Assessment Manikin) arousal/valence
  ratings per game, which could be used to verify or refine labels in
  future work, but were not used here.
- **Per-subject accuracy variation (31%-97%) is observed but not
  explained.** This project did not investigate whether this variation
  correlates with the SAM self-ratings, individual differences in EEG
  signal quality, or other factors -- flagged in Future Work.
- **Artifact robustness testing used one synthetic corruption scenario**
  (broadband noise across 4 of 14 channels). Real-world artifacts
  (eye blinks, muscle tension, movement, electrode displacement) vary
  widely in signature and severity; the rejection method's performance
  under other artifact types or full-channel corruption was not tested
  and is not guaranteed.
- **Computational cost was measured on desktop hardware only.** The
  "faster than real-time" claim has not been validated on embedded or
  wearable hardware, where memory and processing constraints differ
  substantially from a desktop CPU.
- **Window size (5 seconds) was chosen to avoid the overlapping-window
  leakage identified during dataset selection, not optimized for
  classification performance.** Shorter or longer windows, or an
  overlap-aware validation scheme, were not systematically compared.
- **Cross-dataset validation was attempted but not completed.** A
  second, differently-instrumented open dataset was evaluated for use
  (see Future Work), but was excluded after determining that its
  emotion-condition labels could not be reliably verified from available
  documentation. This project's findings are therefore validated on
  GAMEEMO only, and the normalization/feature-importance findings
  should be treated as hypotheses to test on other datasets, not yet
  confirmed to generalize across hardware/paradigms.
  
## Future Work

- Test on additional datasets (e.g. DEAP, SEED) pending institutional
  data-access approval, to assess whether the normalization finding
  generalizes across recording setups
- Investigate why per-subject accuracy varies so widely (31%-97%) —
  possible links to self-reported SAM arousal/valence ratings included
  in the source dataset
- Test the artifact-rejection method under a wider range of corruption
  severities and channel-coverage levels
- Deploy the pipeline on actual embedded/edge hardware to validate the
  real-time-capability claim beyond desktop measurement
- A second open-access dataset (Kumar et al., 2026, "An EEG Dataset for
  Brainwave Recording During Emotion Elicitation via Video Clips" — 30
  subjects, 8-channel Unicorn Hybrid Black, CC BY 4.0) was identified and
  downloaded as a candidate for cross-dataset validation. However, the
  available documentation did not specify which of the 12 raw recording
  files per subject corresponds to which emotion condition, and no
  associated peer-reviewed paper confirming this mapping was found.
  Rather than assume an unverified mapping, this dataset was not used.
  Verifying the file-to-emotion mapping via manual video review (or
  contacting the dataset authors directly) is a clear, concrete next
  step for genuine cross-dataset validation.
---

## Notes on Dataset Availability

The GAMEEMO dataset is not included in this repository. It is freely
available (no license request required) from Mendeley Data:
https://data.mendeley.com/datasets/b3pn4kwpmn/3

## Requirements

- MATLAB R2024b (or compatible)
- Signal Processing Toolbox
- Statistics and Machine Learning Toolbox
- Deep Learning Toolbox

## Project Structure (this public repository)

public/
├── README.md
├── data_audit.md # investigation of the initially-used,
│ # later-rejected Bird et al. dataset
├── data_audit_gameemo.md # audit of the GAMEEMO dataset actually used
└── results/
├── figures/ # all result visualizations
└── tables/ # all result tables (CSV + literature comparison)


Full source code (MATLAB scripts and functions) is maintained in a
private companion repository.