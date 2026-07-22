# Literature Comparison — GAMEEMO 4-Class Emotion Classification

| Study | Method | Validation Protocol | Accuracy |
|---|---|---|---|
| Genetically-optimized PDPL (2023) | Dictionary pair learning + genetic algorithm | Subject-independent | 49.0% |
| CLBP + CSA + XGBoost (2025) | Chaotic local binary pattern + cuckoo search + gradient boosting | "Subject-independent" | 99.2% |
| **This work** | **Band power (theta/alpha/beta/gamma) + per-subject z-score normalization + Random Forest** | **True leave-one-subject-out (28 folds)** | **81.1%** |

**Caveat on cross-study comparison:** these results are not directly
comparable -- studies differ in window length, feature engineering, and
what "subject-independent" precisely means in their pipeline (true LOSO
vs. other splits). The 99.2% result in particular warrants caution: work
elsewhere in this project (see ablation_table.csv, classifier_comparison
results) demonstrated that naive random-fold validation on windowed EEG
features can inflate accuracy by 5-11 points relative to true LOSO due to
near-duplicate/overlapping-window leakage. Without access to the full
methodology of the 99.2% result, it is not possible to confirm whether
similar leakage was controlled for.