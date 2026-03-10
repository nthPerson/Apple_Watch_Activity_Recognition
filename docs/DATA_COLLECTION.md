# Data Collection

**Project:** Exercise Activity Recognition from Apple Watch Sensor Data
**Course:** CON E 652 — Construction Operations Modeling and Technology
**Date:** March 2026

---

## Related Documents

- [Preprocessing Methodology](PREPROCESSING_METHODOLOGY.md) — Signal processing and feature extraction pipeline
- [Model Training](MODEL_TRAINING.md) — Classifier design, tuning, and cross-validation strategy
- [Evaluation Results](EVALUATION_RESULTS.md) — Performance metrics, ablation study, and discussion
- [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md) — Literature review, sensor justification, and benchmark analysis
- [Dataset Column Reference](../data/DATASET_README.md) — Authoritative column definitions and label mappings

---

## 1. Introduction

This document describes the data collection apparatus, protocol, and dataset characteristics for an exercise activity recognition system built on wrist-worn inertial measurement unit (IMU) and heart rate data. The system classifies six activity classes — five distinct exercises plus a rest/transition category — from sensor streams recorded on a consumer Apple Watch during real gym workouts.

Wearable sensing for human activity recognition (HAR) has applications beyond fitness tracking, including construction workforce monitoring, where recognizing worker activities from wrist-worn sensors can inform productivity analysis and safety compliance. The feasibility analysis, literature review, and sensor selection rationale for this project are detailed in the [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md).

## 2. Apparatus and Sensor Configuration

### 2.1 Device

Data was collected using an **Apple Watch Series 9** running the **SensorLog** app (v6.1, developed by Dr. Bernd Thomas). SensorLog is a research-grade iOS/watchOS sensor logging application that captures raw and fused motion data from the Apple Watch's onboard sensors, exports in CSV format to a connected iPhone via Bluetooth, and supports real-time integer labeling during recording.

### 2.2 Sampling Rate

All sensors were logged simultaneously at **50 Hz** (20 ms nominal inter-sample interval). This rate was chosen based on the following considerations:

- It matches the UCI HAR benchmark standard and Apple's WWDC recommendations for workout analysis.
- Human voluntary exercise movements are well-contained below 10 Hz; 50 Hz provides a Nyquist frequency of 25 Hz with comfortable margin.
- SensorLog supports up to 50 Hz when logging all sensors simultaneously on Apple Watch.
- Published systematic reviews found marginal accuracy improvements above 50 Hz for gym exercise classification.

### 2.3 Sensors Enabled

The following sensor streams were recorded:

| Sensor Group | Channels | Units | Source |
|---|---|---|---|
| User acceleration (gravity-removed) | X, Y, Z | G | CMDeviceMotion |
| Rotation rate (gyroscope) | X, Y, Z | rad/s | CMDeviceMotion |
| Gravity vector | X, Y, Z | G | CMDeviceMotion |
| Attitude (Euler angles) | Yaw, Roll, Pitch | rad | CMDeviceMotion |
| Attitude (quaternion) | X, Y, Z, W | unitless | CMDeviceMotion |
| Raw accelerometer | X, Y, Z | G | CMAccelerometer |
| Heart rate | BPM | beats/min | HealthKit |
| Heart rate variability | HRV | ms | HealthKit |

The primary time reference for all motion data is the `motionTimestamp_sinceReboot(s)` column (seconds since device boot). Each CSV row contains 35 columns total, including the activity label. The full column specification is maintained in the [Dataset Column Reference](../data/DATASET_README.md).

### 2.4 Magnetometer

The magnetometer was **disabled** during all recording sessions. Ferromagnetic interference from dumbbells, weight racks, and building structures in gym environments produces unreliable magnetic field readings. Published gym exercise studies consistently exclude magnetometer data for this reason. The 6-axis IMU (3-axis accelerometer + 3-axis gyroscope via CMDeviceMotion) provides sufficient discriminative information for the target activity classes.

## 3. Activity Classes

Six activity classes were defined for this experiment. Labels were assigned in real time during recording using SensorLog's built-in integer label feature.

| Label | Activity | Description | Equipment |
|---|---|---|---|
| 0 | Rest | Non-exercise activity; rest between sets; transitions | None |
| 1 | Pull-ups | Pull-ups | Pull-up/dip rack |
| 2 | Bicep curls | Bicep curls | Dumbbell weights |
| 3 | Lateral raises | Lateral raises | Dumbbell weights |
| 4 | Cardio | Outdoor walking or running | None |
| 5 | Squats | Goblet squats | Dumbbell weight |

The activities span three broad movement categories: **upper-body resistance** (pull-ups, bicep curls, lateral raises), **lower-body resistance** (squats), and **locomotion** (cardio). The rest class captures all non-exercise intervals, including transitions between exercises, rest between sets, and setup/teardown periods.

## 4. Data Collection Protocol

### 4.1 Session Structure

Data was collected across **4 recording sessions**, each saved as a separate CSV file. Within each session, exercises were performed in sets of approximately 8–12 repetitions with rest periods between sets. The SensorLog label button was tapped at the start and end of each exercise set to mark activity boundaries in real time.

### 4.2 Session Contents

| Session | File | Activities Recorded |
|---|---|---|
| S0 | `exercise_data_0.csv` | Rest, Pull-ups |
| S1 | `exercise_data_1.csv` | Rest, Pull-ups, Bicep curls, Lateral raises |
| S2 | `exercise_data_2.csv` | Rest, Cardio |
| S3 | `exercise_data_3.csv` | Rest, Pull-ups, Bicep curls, Lateral raises, Squats |

### 4.3 Session Coverage Limitations

Not all activity classes appear in every session:

| Activity | Sessions Present | Count |
|---|---|---|
| Rest | S0, S1, S2, S3 | 4 |
| Pull-ups | S0, S1, S3 | 3 |
| Bicep curls | S1, S3 | 2 |
| Lateral raises | S1, S3 | 2 |
| Cardio | S2 only | 1 |
| Squats | S3 only | 1 |

The single-session representation of cardio and squats has implications for the cross-validation strategy (see [Model Training](MODEL_TRAINING.md#3-cross-validation-strategy-leave-one-set-out)).

## 5. Dataset Summary Statistics

### 5.1 Overall Scale

| Metric | Value |
|---|---|
| Total samples | 222,008 |
| Total duration | ~74 minutes (4,440.2 seconds) |
| Total disk size | 121.06 MB |
| Recording sessions | 4 |
| Contiguous exercise/rest sets | 307 |

### 5.2 Per-File Breakdown

| File | Disk Size | Samples |
|---|---|---|
| `exercise_data_0.csv` | 10,714 KB | 19,139 |
| `exercise_data_1.csv` | 52,343 KB | 93,569 |
| `exercise_data_2.csv` | 15,544 KB | 28,211 |
| `exercise_data_3.csv` | 45,367 KB | 81,089 |

### 5.3 Per-Class Distribution

| Label | Activity | Samples | Duration | % of Total |
|---|---|---|---|---|
| 0 | Rest | 73,488 | 24.5 min | 33.1% |
| 1 | Pull-ups | 31,581 | 10.5 min | 14.2% |
| 2 | Bicep curls | 32,604 | 10.9 min | 14.7% |
| 3 | Lateral raises | 30,683 | 10.2 min | 13.8% |
| 4 | Cardio | 27,562 | 9.2 min | 12.4% |
| 5 | Squats | 26,090 | 8.7 min | 11.8% |

The rest class accounts for approximately one-third of all samples, which is expected given the rest periods between exercise sets. This class imbalance is addressed during model training through balanced sample weights (see [Model Training](MODEL_TRAINING.md#4-class-imbalance-correction)).

### 5.4 Samples Per File Per Class

| File | Rest | Pull-ups | Bicep curls | Lateral raises | Cardio | Squats | Total |
|---|---|---|---|---|---|---|---|
| `exercise_data_0.csv` | 9,389 | 9,685 | 65 | 0 | 0 | 0 | 19,139 |
| `exercise_data_1.csv` | 32,385 | 10,121 | 25,260 | 25,803 | 0 | 0 | 93,569 |
| `exercise_data_2.csv` | 649 | 0 | 0 | 0 | 27,562 | 0 | 28,211 |
| `exercise_data_3.csv` | 31,065 | 11,775 | 7,279 | 4,880 | 0 | 26,090 | 81,089 |

### 5.5 Set-Level Statistics

A "set" is a contiguous run of the same activity label within a single recording file. Sets serve as the fundamental grouping unit for cross-validation.

| Activity | Sets | Mean Duration (s) | Min Duration (s) | Max Duration (s) |
|---|---|---|---|---|
| Rest | 135 | 1.2 | — | 138.5 |
| Pull-ups | 66 | 2.7 | — | 47.3 |
| Bicep curls | 26 | 10.4 | — | 148.9 |
| Lateral raises | 38 | 6.2 | — | 95.9 |
| Cardio | 5 | 101.7 | — | 277.2 |
| Squats | 37 | 7.5 | — | 161.3 |

Note: Some sets show very short or negative computed durations due to unexplained SensorLog issues and timestamp resolution at set boundaries. These boundary artifacts are handled during preprocessing by the gap-splitting and majority-vote windowing steps (see [Preprocessing Methodology](PREPROCESSING_METHODOLOGY.md)).

## 6. Data Quality Assessment

A comprehensive data quality analysis was performed in `data_inspection.ipynb`. Key findings:

### 6.1 Missing Values

**No missing values** were detected in any sensor column across all four recording files. All 35 CSV columns are fully populated for every sample.

### 6.2 Sampling Rate Verification

The raw inter-sample intervals show a median of approximately 20 ms (consistent with 50 Hz) with minor jitter of ±1–2 ms. Maximum gaps of 85–297 seconds were observed within individual session files, corresponding to points where the SensorLog app was paused and resumed mid-workout. These recording discontinuities are addressed by the gap-splitting step in the [preprocessing pipeline](PREPROCESSING_METHODOLOGY.md#2-gap-splitting).

### 6.3 Visualizations

The `data_inspection.ipynb` notebook provides the following quality visualizations:

1. **Samples per activity** — bar chart confirming class distribution
2. **Session timelines** — color-coded label bar with accelerometer and gyroscope traces for each recording file
3. **Sensor distributions** — violin plots of acceleration and gyroscope magnitude per activity class
4. **Heart rate by activity** — box plot showing HR distribution across classes
5. **Set duration distributions** — box plot of exercise set lengths by activity
6. **Per-set detail inspection** — zoomed sensor traces for every exercise set with labeled boundaries
7. **Sampling rate histograms** — inter-sample interval distributions per file
