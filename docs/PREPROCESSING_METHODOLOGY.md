# Preprocessing Methodology

**Project:** Exercise Activity Recognition from Apple Watch Sensor Data <br>
**Course:** CON E 652 — Construction Operations Modeling and Technology <br>
**Date:** March 2026

---

## Related Documents

- [Data Collection](DATA_COLLECTION.md) — Apparatus, protocol, and dataset characteristics
- [Model Training](MODEL_TRAINING.md) — Classifier design, tuning, and cross-validation strategy
- [Evaluation Results](EVALUATION_RESULTS.md) — Performance metrics, ablation study, and discussion
- [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md) — Literature review and sensor justification

---

## 1. Pipeline Overview

The preprocessing pipeline transforms raw Apple Watch sensor CSV files into a clean, labeled feature matrix suitable for training an activity recognition classifier. The pipeline consists of ten sequential steps:

1. Load raw CSVs and split on recording gaps
2. Resample to a uniform 50 Hz grid
3. Apply a low-pass Butterworth filter
4. Compute derived signals (magnitude, jerk)
5. Z-score normalize per recording session
6. Segment into fixed-length sliding windows
7. Extract time-domain, frequency-domain, and statistical features via TSFEL
8. Compute heart rate summary statistics
9. Post-extraction feature cleaning
10. Export the feature matrix

All preprocessing code is implemented in `preprocessing.ipynb`.

### Master Parameter Table

| Parameter | Value | Justification |
|---|---|---|
| Target sampling rate | 50 Hz | Matches device capture rate; sufficient for exercise motion (<10 Hz) |
| Sample interval | 20 ms | Inverse of 50 Hz |
| Gap threshold | 0.5 s | 25x nominal interval; splits discontinuous recording bursts |
| Butterworth filter order | 3 | Balance between attenuation steepness and phase distortion |
| Filter cutoff frequency | 20.0 Hz | Preserves all biomechanical content; removes 20–25 Hz sensor noise |
| Filter implementation | Zero-phase (sosfiltfilt) | Eliminates phase distortion; effective order = 6 |
| Window length | 3.0 s (150 samples) | Captures 1–3 exercise repetitions; HAR literature consensus |
| Window stride | 75 samples (50% overlap) | Doubles window count; prevents boundary feature loss |
| Min label fraction | 0.60 (60%) | Discards ambiguous transition windows |
| IMU resampling method | Cubic spline | Smooth interpolation; minimizes artifacts |
| Label resampling method | Nearest-neighbor | Preserves discrete integer values |
| HR resampling method | Nearest-neighbor | Preserves sparse physiological signal dynamics |
| Normalization | Per-session z-score | Prevents session identity leakage; session-invariant representation |

## 2. Gap-Splitting

### Problem

Examination of the raw data (see [Data Collection](DATA_COLLECTION.md#62-sampling-rate-verification)) revealed maximum inter-sample gaps of 85–297 seconds within individual CSV files. These gaps occur when the SensorLog app was paused and resumed mid-workout, creating a single file composed of multiple discontinuous recording bursts.

If left unsplit, these gaps produce two problems:

1. **Invalid windows at boundaries** — a 3-second window straddling a gap would contain data from two temporally disjoint recording periods, producing meaningless features.
2. **Corrupted resampling and filtering** — both cubic spline interpolation and the Butterworth filter assume temporally adjacent samples. A timestamp jump of minutes would cause wild interpolation extrapolation and filter ringing.

### Implementation

Any pair of consecutive samples whose timestamp difference exceeds **0.5 seconds** (25x the nominal 20 ms interval) is treated as a segment boundary. Segments shorter than one window length (150 samples) are discarded.

### Result

| Source File | Segments |
|---|---|
| `exercise_data_0.csv` | 2 |
| `exercise_data_1.csv` | 1 |
| `exercise_data_2.csv` | 1 |
| `exercise_data_3.csv` | 1 |
| **Total** | **5** |

Segment lengths: min = 2,190 samples, max = 93,569 samples, mean = 44,402 samples.

## 3. Resampling to Uniform 50 Hz Grid

### Problem

The Apple Watch does not guarantee perfectly uniform sample timing. Timestamp jitter of ±1–2 ms is normal, and occasional missed samples create isolated larger gaps. Most downstream processing — the Butterworth filter (which assumes fixed sample period), TSFEL's FFT-based spectral features (which require uniform spacing), and sliding-window segmentation (which counts samples) — implicitly assumes a uniform time axis.

### Implementation

Each segment is interpolated onto a perfectly uniform time grid at 50 Hz:

- **Continuous sensor channels** (6 IMU axes): Cubic spline interpolation (`scipy.interpolate.CubicSpline`), which provides smooth, continuous estimates between samples and minimizes interpolation artifacts compared to linear interpolation. Duplicate timestamps are removed (keep first) before interpolation, as cubic spline requires strictly increasing x-values.
- **Integer labels**: Nearest-neighbor interpolation, preserving discrete label values exactly.
- **Heart rate**: Nearest-neighbor interpolation. HR is a sparse physiological signal that updates approximately every 5 seconds via HealthKit; cubic interpolation would create artificial dynamics between updates.

### Verification

After resampling:
- Mean inter-sample interval: **20.000 ms**
- Standard deviation: **0.0000 ms** (perfectly uniform)
- Total resampled samples: **185,298** across all 5 segments
- Heart rate column successfully carried through resampling

## 4. Butterworth Low-Pass Filter

### Design

A 3rd-order Butterworth low-pass filter with a 20 Hz cutoff frequency is applied to all 6 IMU channels (3-axis user acceleration, 3-axis rotation rate).

### Why Butterworth?

The Butterworth filter provides a **maximally flat magnitude response** in the passband, meaning it does not distort the amplitudes of exercise motion frequencies. This is the standard choice in biomechanical signal processing and IMU-based HAR literature (Winter, 2009; Anguita et al., 2013; Banos et al., 2014).

| Filter Type | Passband | Transition Band | Trade-off |
|---|---|---|---|
| **Butterworth** | Maximally flat (no ripple) | Moderate roll-off | Best amplitude fidelity |
| Chebyshev Type I | Ripple in passband | Steeper roll-off | Faster attenuation, less amplitude fidelity |
| Elliptic | Ripple in both bands | Steepest roll-off | Narrowband applications |

### Cutoff Frequency Rationale

Human voluntary limb movements during exercise are well-contained below 10 Hz. The Apple Watch samples at 50 Hz, giving a Nyquist frequency of 25 Hz. A 20 Hz cutoff removes sensor noise in the 20–25 Hz band while preserving all biomechanically relevant motion frequencies with a comfortable margin.

### Implementation

The filter is designed using `scipy.signal.butter()` in **second-order sections (SOS) representation** for improved numerical stability, and applied using `scipy.signal.sosfiltfilt()` — a zero-phase implementation that applies the filter forward and backward. This eliminates all phase distortion at the cost of doubling the effective filter order (net effective order = 6).

The filter is applied to the 6 raw IMU channels only. Derived signals are computed *after* filtering to avoid amplifying high-frequency noise in derivative calculations.

## 5. Derived Signals

Three rotation-invariant signals are computed from the filtered IMU channels:

| Signal | Formula | Units | Physical Meaning |
|---|---|---|---|
| `accel_mag` | sqrt(ax^2 + ay^2 + az^2) | G | Overall linear acceleration intensity |
| `gyro_mag` | sqrt(wx^2 + wy^2 + wz^2) | rad/s | Overall angular velocity (rotational intensity) |
| `jerk_mag` | d\|a\|/dt (finite difference x FS) | G/s | Rate of change of acceleration magnitude |

### Motivation

The watch's orientation on the wrist varies between sessions and even between sets within a session. **Magnitude signals are rotation-invariant** — they describe the overall intensity of movement regardless of watch orientation. This compensates for the uncontrolled variability in device placement across recording sessions.

Jerk is particularly discriminative for repetitive weightlifting exercises because each repetition produces a characteristic acceleration burst followed by rapid deceleration. The temporal pattern of jerk peaks encodes both cadence (repetition rate) and motion sharpness, which differ substantially across exercise types.

### Computation Order

Derived signals are computed **after** the Butterworth filter to avoid amplifying high-frequency noise through the derivative operation. The finite-difference jerk calculation would amplify any residual noise; filtering first ensures the jerk signal reflects genuine motion dynamics.

## 6. Per-Session Z-Score Normalization

### Method

Z-score standardization (zero mean, unit variance) is applied to all 10 feature channels (6 IMU + 3 derived + 1 heart rate) using scikit-learn's `StandardScaler`. The scaler is fit on all data within each source file (session) and applied to that session's segments.

### Why Per-Session?

Global normalization would compute statistics across all sessions combined, allowing unusually active or quiet sessions to dominate the global mean and variance. Per-session normalization makes the representation **session-invariant**: it removes session-level DC offset and scale caused by differences in watch fit, wrist position, and individual biomechanics across recording sessions.

This approach also preserves the integrity of cross-validation — no data from other sessions leaks into the normalization of a given session.

### Verification

Post-normalization statistics for `exercise_data_0.csv` (representative):
- All channel means: ~0.0 (achieved)
- All channel standard deviations: ~1.0 (achieved)

## 7. Sliding Window Segmentation

### Parameters

- **Window length**: 3.0 seconds = 150 samples at 50 Hz
- **Stride**: 75 samples = 50% overlap
- **Label assignment**: Majority vote — the most frequent label within the window becomes the ground truth
- **Ambiguity filter**: Windows where the majority label accounts for less than 60% of samples are discarded

### Design Rationale

**Window length (3 seconds)**: Typical exercise cadences range from 0.3 to 1.0 repetitions per second, so a 3-second window captures 1–3 full repetitions. Windows shorter than 2 seconds risk capturing only partial repetitions, making frequency-domain features unreliable. Windows longer than 4–5 seconds reduce the total window count and may span multiple label transitions. This is the consensus window length in HAR literature (Anguita et al., 2013; Banos et al., 2014).

**Overlap (50%)**: Doubles the number of training windows from a fixed recording duration, which is especially helpful for classes with fewer recorded sets (cardio, squats). Also ensures that features of any given moment appear in at least two windows, reducing boundary effects.

### Set ID Assignment

Each window inherits a **set ID** identifying the contiguous label run (exercise set or rest bout) it predominantly belongs to. This set_id enables Leave-One-Set-Out cross-validation during model training, which prevents data leakage from overlapping windows (see [Model Training](MODEL_TRAINING.md#3-cross-validation-strategy-leave-one-set-out)).

### Result

| Metric | Value |
|---|---|
| Total windows | 2,446 |
| Unique set IDs | 58 |

**Windows per class:**

| Label | Activity | Windows | % of Total |
|---|---|---|---|
| 0 | Rest | 815 | 33.3% |
| 1 | Pull-ups | 311 | 12.7% |
| 2 | Bicep curls | 368 | 15.0% |
| 3 | Lateral raises | 340 | 13.9% |
| 4 | Cardio | 337 | 13.8% |
| 5 | Squats | 275 | 11.2% |

## 8. Feature Extraction

### 8.1 TSFEL Features (IMU Channels)

The **Time Series Feature Extraction Library** (TSFEL; Barandas et al., 2020) is used to compute a comprehensive set of handcrafted features from each window. Three of four available feature domains are used:

| Domain | Features/Channel | Examples |
|---|---|---|
| **Temporal** | ~15 | Mean absolute deviation, autocorrelation, zero-crossing rate, peak count |
| **Statistical** | ~20 | Mean, variance, skewness, kurtosis, interquartile range, entropy |
| **Spectral** | ~26 | Dominant frequency, spectral centroid, spectral energy, MFCC |

The **fractal domain** is excluded because its features (Hurst exponent, detrended fluctuation analysis) are computationally expensive and of limited discriminative value for short 3-second exercise windows.

TSFEL features are computed **independently for each of the 9 IMU signal channels**:

1. `acc_x` — User acceleration X (G)
2. `acc_y` — User acceleration Y (G)
3. `acc_z` — User acceleration Z (G)
4. `gyr_x` — Rotation rate X (rad/s)
5. `gyr_y` — Rotation rate Y (rad/s)
6. `gyr_z` — Rotation rate Z (rad/s)
7. `accel_mag` — Acceleration magnitude (G)
8. `gyro_mag` — Rotation rate magnitude (rad/s)
9. `jerk_mag` — Jerk magnitude (G/s)

**Raw TSFEL output**: 1,404 features (approximately 156 features per channel across 9 channels).

### 8.2 Heart Rate Summary Statistics

Heart rate (BPM) is recorded by the Apple Watch via HealthKit and updates approximately every 5 seconds. Because HR is a sparse physiological signal — not a high-frequency time series — running TSFEL's full spectral analysis on a 3-second HR window would produce meaningless frequency-domain features (the signal is nearly constant within each window).

Instead, **5 simple summary statistics** are computed per window:

| Feature | Description |
|---|---|
| `hr_mean` | Mean heart rate in the window |
| `hr_std` | Standard deviation (within-window HR variability) |
| `hr_min` | Minimum HR value |
| `hr_max` | Maximum HR value |
| `hr_range` | Max - Min (spread within window) |

These summary statistics capture the physiological context of each window while avoiding the artifacts that would result from applying spectral analysis to a sparse signal. The separate treatment of HR features also enables an ablation study comparing classifier performance with and without heart rate information (see [Evaluation Results](EVALUATION_RESULTS.md#4-heart-rate-ablation-results)).

**Total raw features**: 1,404 TSFEL + 5 HR = **1,409 features** per window.

### References

- Barandas, M., et al. (2020). TSFEL: Time Series Feature Extraction Library. *SoftwareX*, 11, 100456.
- Anguita, D., et al. (2013). A Public Domain Dataset for Human Activity Recognition Using Smartphones. *ESANN 2013*.

## 9. Post-Extraction Cleaning

### 9.1 NaN Column Removal

Any feature column containing at least one NaN across all 2,446 windows is dropped. NaN values in even one window would propagate to undefined predictions for that window in downstream models.

**Result**: 0 NaN columns dropped (all features successfully computed for all windows).

### 9.2 Near-Constant Feature Removal

TSFEL features with variance below 1e-10 across all windows are removed. These constant or near-constant features carry no discriminative information and can cause numerical issues during model training.

**Important**: HR features are explicitly excluded from this variance filter and always preserved, regardless of their variance. Their interpretive value for the ablation study justifies unconditional retention.

**Result**: 134 near-constant TSFEL features dropped.

### 9.3 Feature Quality Audit

After cleaning:
- NaN values in feature matrix: **0**
- Inf values in feature matrix: **0**

## 10. Final Feature Matrix

### Dimensions

| Metric | Value |
|---|---|
| Rows (windows) | 2,446 |
| IMU/TSFEL features | 1,270 |
| HR features | 5 |
| **Total features** | **1,275** |
| Metadata columns | 3 (label, source_file, set_id) |
| **Total columns** | **1,278** |

### Metadata Columns

| Column | Description |
|---|---|
| `label` | Integer activity class (0–5) |
| `source_file` | Originating CSV filename (session identifier) |
| `set_id` | Unique identifier for the contiguous exercise set within a segment |

### Export Format

The feature matrix is saved in two formats:
- **Parquet** (`data/features.parquet`, 27,319 KB) — efficient binary format; preserves dtypes; fast I/O
- **CSV** (`data/features.csv`, 57,929 KB) — human-readable; importable into any tool
