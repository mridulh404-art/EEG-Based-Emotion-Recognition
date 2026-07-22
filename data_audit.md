# Phase 0 — Data Audit Findings (Bird et al. "Feeling Emotions" — REJECTED)

**Dataset:** emotions.csv (Bird et al., "EEG Brainwave Dataset: Feeling Emotions")
**Rows:** 2132  **Columns:** 2549 (2548 features + label)

## What the data is
Pre-extracted statistical/FFT features (mean, fft, stddev, etc. per channel),
not raw voltage. Only 2 subjects.

## Critical finding: near-duplicate leakage
Nearest-neighbor check: 95% of all rows share their nearest neighbor's label
(vs. 33.3% expected by chance for 3 balanced classes). This strongly suggests
overlapping-window feature extraction created near-duplicate rows, which a
naive shuffled train/test split leaks across.

## Why this dataset was rejected
1. Only 2 subjects -- no meaningful subject-independent validation possible
2. Data arrived pre-shuffled with no recoverable subject/session order
3. Near-duplicate leakage inflates naive cross-validation accuracy (measured
   96.95% mean, 5-fold CV -- almost certainly inflated)

**Decision: switched to GAMEEMO dataset (28 subjects, raw signal, real
subject IDs). See data_audit_gameemo.md.**