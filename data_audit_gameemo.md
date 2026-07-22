# Data Audit — GAMEEMO Dataset (dataset actually used)

**Source:** Mendeley Data, "Database for Emotion Recognition System Based on
EEG Signals and Various Computer Games – GAMEEMO"
**Subjects:** 28 (S01-S28; S28 folder contained a nested duplicate (S26)
sub-folder from a packaging error -- excluded, not real data)
**Channels:** 14 (Emotiv EPOC+): AF3, AF4, F3, F4, F7, F8, FC5, FC6, O1, O2,
P7, P8, T7, T8
**Sampling rate:** ~127.5 Hz measured (128 Hz nominal, Emotiv EPOC+ spec)
**Recording:** 4 games x 5 min per subject, raw + preprocessed versions
provided in both .csv and .mat

## Label mapping (game -> emotion quadrant)
- G1 = Train Sim World       -> BORING   (Low Arousal, Negative Valence)
- G2 = Unravel               -> CALM     (Low Arousal, Positive Valence)
- G3 = Slender: The Arrival  -> HORROR   (High Arousal, Negative Valence)
- G4 = Goat Simulator        -> FUNNY    (High Arousal, Positive Valence)

## What this means for the framework
Genuine raw time-series EEG, unlike Bird et al.'s pre-extracted dataset.
Bandpass filtering into alpha/beta/theta/gamma bands applies as originally
designed. Subject identity is explicit (folder-level), enabling real
leave-one-subject-out validation.

## Known data quality issues (found and handled)
- `(S28)` folder contained a nested duplicate `(S26)` sub-folder --
  extraction/packaging artifact, excluded from the pipeline
- S26's raw files use "AllChannels" naming instead of "AllRawChannels"
  used by every other subject -- loader handles both patterns