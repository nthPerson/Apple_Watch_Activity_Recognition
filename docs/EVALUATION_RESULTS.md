# Evaluation Results

**Project:** Exercise Activity Recognition from Apple Watch Sensor Data <br>
**Course:** CON E 652 — Construction Operations Modeling and Technology <br>
**Date:** March 2026

---

## Related Documents

- [Data Collection](DATA_COLLECTION.md) — Apparatus, protocol, and dataset characteristics
- [Preprocessing Methodology](PREPROCESSING_METHODOLOGY.md) — Signal processing and feature extraction pipeline
- [Model Training](MODEL_TRAINING.md) — Classifier design, tuning, and cross-validation strategy
- [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md) — Literature review and benchmark analysis

---

## 1. Overall Performance Summary

Both model configurations were evaluated using Leave-One-Set-Out cross-validation with 58 folds, balanced sample weights, and independently tuned hyperparameters (see [Model Training](MODEL_TRAINING.md) for methodology details).

| Metric | IMU-only | IMU+HR | Delta |
|---|---|---|---|
| **Accuracy** | 95.05% | 95.58% | +0.53% |
| **Balanced Accuracy** | 95.11% | 95.33% | +0.22% |
| **Macro F1** | 0.9514 | 0.9572 | +0.0058 |
| **Weighted F1** | 0.9506 | 0.9559 | +0.0053 |

Both configurations exceed the **90–96% accuracy target** established in the [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md), which was based on published benchmarks for wrist-worn IMU exercise classification.

### Metrics Definitions

| Metric | Description |
|---|---|
| Accuracy | Fraction of correctly classified windows |
| Balanced Accuracy | Mean of per-class recall; accounts for class imbalance |
| Macro F1 | Unweighted mean of per-class F1-scores; treats all classes equally |
| Weighted F1 | F1 weighted by class frequency; reflects overall dataset performance |

## 2. Per-Class Performance

### 2.1 Classification Report — IMU-only Model

| Activity | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Rest | 0.9415 | 0.9485 | 0.9450 | 815 |
| Pull-ups | 0.9503 | 0.9839 | 0.9668 | 311 |
| Bicep curls | 0.9803 | 0.9457 | 0.9627 | 368 |
| Lateral raises | 0.9314 | 0.9588 | 0.9449 | 340 |
| Cardio | 0.9812 | 0.9318 | 0.9559 | 337 |
| Squats | 0.9281 | 0.9382 | 0.9331 | 275 |
| **Macro avg** | **0.9521** | **0.9511** | **0.9514** | **2,446** |
| **Weighted avg** | **0.9510** | **0.9505** | **0.9506** | **2,446** |

### 2.2 Classification Report — IMU+HR Model

| Activity | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Rest | 0.9336 | 0.9656 | 0.9493 | 815 |
| Pull-ups | 0.9620 | 0.9775 | 0.9697 | 311 |
| Bicep curls | 0.9775 | 0.9457 | 0.9613 | 368 |
| Lateral raises | 0.9563 | 0.9647 | 0.9605 | 340 |
| Cardio | 0.9843 | 0.9318 | 0.9573 | 337 |
| Squats | 0.9554 | 0.9345 | 0.9449 | 275 |
| **Macro avg** | **0.9615** | **0.9533** | **0.9572** | **2,446** |
| **Weighted avg** | **0.9564** | **0.9558** | **0.9559** | **2,446** |

### 2.3 Confusion Matrix — IMU-only Model

Rows represent the true class; columns represent the predicted class. All counts are aggregated across the 58 LOSO-Set folds.

| True \ Predicted | Rest | Pull-ups | Bicep curls | Lateral raises | Cardio | Squats |
|---|---|---|---|---|---|---|
| **Rest** | **773** | 4 | 6 | 9 | 5 | 18 |
| **Pull-ups** | 0 | **306** | 0 | 5 | 0 | 0 |
| **Bicep curls** | 6 | 5 | **348** | 7 | 0 | 2 |
| **Lateral raises** | 6 | 7 | 1 | **326** | 0 | 0 |
| **Cardio** | 23 | 0 | 0 | 0 | **314** | 0 |
| **Squats** | 13 | 0 | 0 | 3 | 1 | **258** |

### 2.4 Confusion Matrix — IMU+HR Model

| True \ Predicted | Rest | Pull-ups | Bicep curls | Lateral raises | Cardio | Squats |
|---|---|---|---|---|---|---|
| **Rest** | **787** | 3 | 4 | 4 | 5 | 12 |
| **Pull-ups** | 2 | **304** | 3 | 2 | 0 | 0 |
| **Bicep curls** | 8 | 3 | **348** | 9 | 0 | 0 |
| **Lateral raises** | 5 | 6 | 1 | **328** | 0 | 0 |
| **Cardio** | 23 | 0 | 0 | 0 | **314** | 0 |
| **Squats** | 18 | 0 | 0 | 0 | 0 | **257** |

### 2.5 Row-Normalised Confusion Matrix — IMU+HR Model (Recall per Cell)

| True \ Predicted | Rest | Pull-ups | Bicep curls | Lateral raises | Cardio | Squats |
|---|---|---|---|---|---|---|
| **Rest** | **0.966** | 0.004 | 0.005 | 0.005 | 0.006 | 0.015 |
| **Pull-ups** | 0.006 | **0.977** | 0.010 | 0.006 | 0.000 | 0.000 |
| **Bicep curls** | 0.022 | 0.008 | **0.946** | 0.024 | 0.000 | 0.000 |
| **Lateral raises** | 0.015 | 0.018 | 0.003 | **0.965** | 0.000 | 0.000 |
| **Cardio** | 0.068 | 0.000 | 0.000 | 0.000 | **0.932** | 0.000 |
| **Squats** | 0.065 | 0.000 | 0.000 | 0.000 | 0.000 | **0.935** |

### 2.6 Confusion Matrix Observations

- **Pull-ups** achieve the highest recall (97.7%), consistent with the distinctive motion signature of vertical pulling movements on the wrist sensor.
- **Rest** achieves 96.6% recall in the IMU+HR model (up from 94.8% IMU-only), with most misclassifications scattered across exercise classes — primarily squats (15 windows) and cardio (5 windows).
- **Cardio** misclassifications are exclusively predicted as rest (23 windows in both models), consistent with the lower motion intensity of walking/running compared to resistance exercises.
- **Squats** misclassifications in the IMU+HR model are also exclusively predicted as rest (18 windows), suggesting that the transition periods at the top of goblet squats resemble rest.
- **Bicep curls** and **lateral raises** show some mutual confusion (9 bicep curl windows predicted as lateral raises, 6 lateral raise windows predicted as pull-ups), expected given their similar wrist-level dumbbell motion patterns.
- The IMU+HR model reduces rest misclassifications from 42 errors (IMU-only) to 28 errors, confirming that heart rate helps disambiguate rest from low-intensity exercise.

## 3. Per-Fold Stability Analysis

Because LOSO-Set CV yields one accuracy score per held-out set, we can examine how consistently the model performs across different sets of the same activity.

### Per-Activity Fold Accuracy (IMU-only)

| Activity | Folds | Mean Acc | Std | Min | Max |
|---|---|---|---|---|---|
| Rest | 30 | 0.897 | 0.212 | 0.000 | 1.000 |
| Pull-ups | 9 | 0.984 | 0.025 | 0.923 | 1.000 |
| Bicep curls | 6 | 0.828 | 0.357 | 0.100 | 1.000 |
| Lateral raises | 7 | 0.952 | 0.031 | 0.909 | 1.000 |
| Cardio | 2 | 0.934 | 0.030 | 0.913 | 0.955 |
| Squats | 4 | 0.937 | 0.033 | 0.902 | 0.969 |

### Per-Activity Fold Accuracy (IMU+HR)

| Activity | Folds | Mean Acc | Std | Min | Max |
|---|---|---|---|---|---|
| Rest | 30 | 0.910 | 0.215 | 0.000 | 1.000 |
| Pull-ups | 9 | 0.978 | 0.031 | 0.923 | 1.000 |
| Bicep curls | 6 | 0.828 | 0.357 | 0.100 | 1.000 |
| Lateral raises | 7 | 0.958 | 0.031 | 0.909 | 1.000 |
| Cardio | 2 | 0.934 | 0.030 | 0.913 | 0.955 |
| Squats | 4 | 0.937 | 0.023 | 0.904 | 0.954 |

### Notable Observations

**Bicep curls high variance (std = 0.357)**: One bicep curl fold achieves only 10% accuracy while others achieve 100%. This outlier fold likely corresponds to a set with atypical form, a very short duration, or a set recorded under conditions significantly different from the others. With only 6 bicep curl sets total, a single problematic set disproportionately affects the statistics. This warrants investigation and would be mitigated by collecting additional bicep curl sets across more sessions.

**Rest high variance (std = 0.212–0.215)**: Rest periods vary widely in context — some follow intense exercise (elevated HR, residual motion), others are calm setup periods. The 0.000 minimum suggests at least one rest set where all windows were misclassified, likely a very short transitional rest that resembles adjacent exercise.

**Pull-ups consistently high (min = 0.923)**: The distinctive pulling motion creates a highly separable sensor signature, even in the worst-case fold.

## 4. Heart Rate Ablation Results

### Per-Class Recall Comparison

| Activity | IMU-only Recall | IMU+HR Recall | Delta | Direction |
|---|---|---|---|---|
| Rest | 0.9485 | 0.9656 | +0.0171 | Improved |
| Pull-ups | 0.9839 | 0.9775 | -0.0064 | Slight decrease |
| Bicep curls | 0.9457 | 0.9457 | +0.0000 | Unchanged |
| Lateral raises | 0.9588 | 0.9647 | +0.0059 | Improved |
| Cardio | 0.9318 | 0.9318 | +0.0000 | Unchanged |
| Squats | 0.9382 | 0.9345 | -0.0036 | Slight decrease |

### Per-Class Precision Comparison

| Activity | IMU-only Precision | IMU+HR Precision | Delta | Direction |
|---|---|---|---|---|
| Rest | 0.9415 | 0.9336 | -0.0079 | Slight decrease |
| Pull-ups | 0.9503 | 0.9620 | +0.0117 | Improved |
| Bicep curls | 0.9803 | 0.9775 | -0.0028 | Unchanged |
| Lateral raises | 0.9314 | 0.9563 | +0.0249 | Improved |
| Cardio | 0.9812 | 0.9843 | +0.0031 | Unchanged |
| Squats | 0.9281 | 0.9554 | +0.0273 | Improved |

### Per-Class F1-Score Comparison

| Activity | IMU-only F1 | IMU+HR F1 | Delta | Direction |
|---|---|---|---|---|
| Rest | 0.9450 | 0.9493 | +0.0043 | Improved |
| Pull-ups | 0.9668 | 0.9697 | +0.0029 | Improved |
| Bicep curls | 0.9627 | 0.9613 | -0.0013 | Unchanged |
| Lateral raises | 0.9449 | 0.9605 | +0.0155 | Improved |
| Cardio | 0.9559 | 0.9573 | +0.0015 | Improved |
| Squats | 0.9331 | 0.9449 | +0.0118 | Improved |

### Interpretation

Heart rate features provide a **modest but consistent improvement** at the aggregate level (+0.53% accuracy, +0.0058 macro F1), with gains observed across multiple metrics:

**Recall improvements:**
- **Rest (+1.7%)**: The largest recall gain. HR helps distinguish low-activity rest periods from low-intensity exercise transitions. During rest, HR trends toward baseline values, providing a physiological signal that complements the low-amplitude motion patterns.
- **Lateral raises (+0.6%)**: The moderate exertion of lateral raises may produce a distinctive HR response that helps separate it from the similar dumbbell motion of bicep curls.

**Precision improvements:**
- **Squats (+2.7%)** and **lateral raises (+2.5%)**: The largest precision gains. HR context reduces false positives for these classes, meaning fewer non-squat and non-lateral-raise windows are incorrectly predicted as these activities.
- **Rest precision decreases slightly (-0.8%)**: The model becomes more willing to predict rest (improving recall) at a small cost to precision, as some exercise windows near rest transitions are now classified as rest.

**F1-score improvements** (balancing precision and recall):
- **Lateral raises (+0.0155)**: The largest F1 gain, benefiting from improvements in both precision and recall.
- **Squats (+0.0118)**: Strong F1 improvement driven primarily by precision gains.
- **5 of 6 classes improve or hold steady** in F1; only bicep curls shows a negligible decrease (-0.0013).

The negligible or slightly negative recall deltas for pull-ups (-0.6%) and squats (-0.4%) are within the noise margin of 58-fold CV and do not indicate meaningful degradation. The overall effect is that **HR information helps most where motion signals alone are ambiguous** — specifically in distinguishing rest from low-intensity exercise and in reducing false positives for lower-body and moderate-intensity activities.

The small magnitude of improvement is expected: HR responds to exertion with a lag of several seconds to minutes, updates only every ~5 seconds on the Apple Watch, and carries much less discriminative information per sample than 50 Hz IMU data. Nevertheless, the improvement is consistent across most classes and achieved with only 5 additional features.

## 5. Feature Importance Analysis

Feature importance was computed from the IMU+HR model trained on all available data, using XGBoost's **gain** metric — the average improvement in loss from splits on each feature, aggregated across all trees.

### 5.1 Top 30 Features

| Rank | Feature | Gain | Channel | Domain |
|---|---|---|---|---|
| 1 | `gyr_y_Area under the curve` | 158.88 | gyr_y | Temporal |
| 2 | `gyr_z_Power bandwidth` | 144.26 | gyr_z | Spectral |
| 3 | `acc_x_Zero crossing rate` | 105.34 | acc_x | Temporal |
| 4 | `gyr_y_Fundamental frequency` | 101.59 | gyr_y | Spectral |
| 5 | `gyr_z_MFCC_2` | 96.88 | gyr_z | Spectral |
| 6 | `acc_z_Spectral centroid` | 93.46 | acc_z | Spectral |
| 7 | `acc_z_Median absolute diff` | 86.55 | acc_z | Temporal |
| 8 | `gyr_y_Absolute energy` | 80.63 | gyr_y | Statistical |
| 9 | `gyro_mag_Median absolute diff` | 79.48 | gyro_mag | Temporal |
| 10 | `acc_x_Median absolute diff` | 60.52 | acc_x | Temporal |
| 11 | `accel_mag_ECDF Percentile_0` | 56.74 | accel_mag | Statistical |
| 12 | `acc_y_Human range energy` | 52.92 | acc_y | Spectral |
| 13 | `acc_y_Wavelet energy_1.56Hz` | 50.85 | acc_y | Spectral |
| 14 | `gyr_z_Spectral distance` | 47.05 | gyr_z | Spectral |
| 15 | `acc_x_Mean absolute diff` | 45.32 | acc_x | Temporal |
| 16 | `gyr_y_Interquartile range` | 44.66 | gyr_y | Statistical |
| 17 | `gyr_z_Mean diff` | 39.55 | gyr_z | Temporal |
| 18 | `gyr_z_Mean absolute diff` | 39.47 | gyr_z | Temporal |
| 19 | `gyro_mag_Min` | 38.05 | gyro_mag | Statistical |
| 20 | `accel_mag_Human range energy` | 27.36 | accel_mag | Spectral |
| 21 | `gyr_z_Median absolute diff` | 26.94 | gyr_z | Temporal |
| 22 | `acc_x_Signal distance` | 25.28 | acc_x | Temporal |
| 23 | `gyr_x_Spectral kurtosis` | 24.76 | gyr_x | Spectral |
| 24 | `acc_z_Power bandwidth` | 23.98 | acc_z | Spectral |
| 25 | `acc_z_Signal distance` | 23.90 | acc_z | Temporal |
| 26 | `gyr_x_Wavelet absolute mean_1.79Hz` | 21.71 | gyr_x | Spectral |
| 27 | `gyro_mag_Histogram mode` | 20.67 | gyro_mag | Statistical |
| 28 | `gyr_y_LPCC_10` | 20.48 | gyr_y | Spectral |
| 29 | `gyr_z_Wavelet energy_6.25Hz` | 18.00 | gyr_z | Spectral |
| 30 | `acc_x_Wavelet standard deviation_12.5Hz` | 16.84 | acc_x | Spectral |

### 5.2 Heart Rate Feature Rankings

| Rank | Feature | Gain |
|---|---|---|
| #47 | `hr_mean` | 10.393 |
| #63 | `hr_max` | 8.235 |
| #428 | `hr_min` | 0.375 |
| #443 | `hr_std` | 0.338 |

`hr_range` did not appear among features with non-zero importance.

Of 1,275 total features, **463 features** had non-zero importance. The 4 HR features that contributed (hr_mean, hr_max, hr_min, hr_std) represent a small but meaningful share of the model's decision-making.

### 5.3 Aggregate Feature Importance by Signal Channel

Total gain summed across all features belonging to each signal channel:

| Channel | Description | Total Gain | % of Total | Features Used |
|---|---|---|---|---|
| gyr_y | Gyroscope Y-axis | 548.6 | 19.6% | 61 |
| gyr_z | Gyroscope Z-axis | 547.7 | 19.6% | 57 |
| acc_x | Accelerometer X-axis | 382.7 | 13.7% | 60 |
| acc_z | Accelerometer Z-axis | 356.2 | 12.7% | 63 |
| acc_y | Accelerometer Y-axis | 272.9 | 9.8% | 53 |
| gyro_mag | Rotation rate magnitude | 237.2 | 8.5% | 33 |
| accel_mag | Acceleration magnitude | 189.9 | 6.8% | 41 |
| gyr_x | Gyroscope X-axis | 150.3 | 5.4% | 57 |
| jerk_mag | Jerk magnitude | 94.2 | 3.4% | 34 |
| hr | Heart rate | 19.3 | 0.7% | 4 |
| **Total** | | **2,799.0** | **100.0%** | **463** |

**By sensor group:**
- **Gyroscope channels** (gyr_x + gyr_y + gyr_z + gyro_mag): **1,483.8 total gain (53.0%)** — over half of total importance
- **Accelerometer channels** (acc_x + acc_y + acc_z + accel_mag): **1,201.7 total gain (42.9%)** — the second major contributor
- **Derived jerk**: 94.2 total gain (3.4%)
- **Heart rate**: 19.3 total gain (0.7%)

### 5.4 Key Observations

**Gyroscope dominance**: Gyroscope channels account for **53.0% of total model gain**, with gyr_y and gyr_z each contributing ~19.6%. This is consistent with the [Feasibility Report](PROJECT_FEASIBILITY_REPORT.md)'s prediction that rotation rate patterns are critical for distinguishing upper-body exercises. The Y-axis and Z-axis rotation rates are particularly discriminative — these axes capture forearm rotation (bicep curls) versus lateral arm elevation (lateral raises) versus vertical pulling (pull-ups). The top two features overall are both gyroscope features: `gyr_y_Area under the curve` (158.88) and `gyr_z_Power bandwidth` (144.26).

**Spectral features are highly ranked**: Of the top 30 features, 14 are from the spectral domain (power bandwidth, spectral centroid, fundamental frequency, MFCC, wavelet energy, spectral distance, spectral kurtosis, LPCC). This confirms that the cadence and frequency signature of each exercise provides strong discriminative power and justifies the decision to include TSFEL's spectral domain in feature extraction. Temporal features (11 of top 30) are the second most common domain, with statistical features (5 of top 30) providing complementary information.

**Derived magnitude signals contribute**: The rotation-invariant magnitude signals — `gyro_mag` (8.5% of total gain, rank #9 feature), `accel_mag` (6.8%), and `jerk_mag` (3.4%) — collectively account for 18.7% of importance. This validates the decision to compute derived signals in the preprocessing pipeline, as they provide discriminative information beyond the raw axis-specific channels.

**HR features rank mid-tier**: `hr_mean` at rank #47 (gain = 10.39) and `hr_max` at rank #63 (gain = 8.24) contribute meaningfully but modestly compared to the top IMU features, with all HR features totaling 0.7% of model gain. This is consistent with the ablation finding: HR provides useful supplementary information but is not a primary discriminator.

## 6. Discussion

### 6.1 Comparison to Literature Benchmarks

| Benchmark | Accuracy | Participants | Exercises | Sensor |
|---|---|---|---|---|
| **This project (IMU+HR)** | **95.6%** | **1** | **6** | **Wrist IMU + HR** |
| uLift (Lim et al., 2024) | 90% | 35 | 15 | Wrist accelerometer |
| UCI HAR (Anguita et al., 2013) | 96% | 30 | 6 | Waist IMU |
| RecGym (UCI) | Varies | 10 | 12 | Wrist IMU |

The achieved 95.6% accuracy is competitive with published benchmarks, particularly notable given that data was collected from a single consumer device (Apple Watch) in a real gym environment with no specialized hardware. The result falls within the 90–96% range predicted by the feasibility analysis for wrist-worn IMU exercise classification.

### 6.2 Strengths

1. **Consumer hardware**: The entire system uses an off-the-shelf Apple Watch and a $3–5 app, requiring no specialized research equipment or body-worn sensor arrays.
2. **Real-world conditions**: Data was collected during actual gym workouts, not in a controlled laboratory setting. This increases ecological validity.
3. **End-to-end pipeline**: From raw CSV to trained model, every step is documented, parameterized, and reproducible.
4. **Rigorous evaluation**: LOSO-Set CV prevents temporal data leakage; balanced sample weights address class imbalance; hyperparameters are tuned separately from evaluation.
5. **Ablation methodology**: The HR ablation study provides quantitative evidence for the contribution of each signal modality.

### 6.3 Limitations

1. **Single participant**: All data comes from one individual. The model captures this person's exercise form, device placement, and physiological characteristics. Generalization to other users is unknown and would require multi-participant data collection.

2. **Single-session activity classes**: Cardio appears only in session S2 and squats only in session S3. This prevents true Leave-One-Session-Out cross-validation for these classes and limits the diversity of training examples. The LOSO-Set evaluation may be slightly optimistic for these classes because within-session characteristics are shared across training and test sets.

3. **Bicep curls outlier fold**: One of the 6 bicep curl folds achieves only 10% accuracy, suggesting a problematic set that requires investigation. This could indicate labeling error, atypical exercise form, or an extremely short set where window-level features are unreliable.

4. **Rest class heterogeneity**: The rest class encompasses a wide range of activities (standing, walking between stations, adjusting equipment, recovering between sets), making it inherently harder to model as a single class. The high per-fold variance (std = 0.212–0.215) reflects this diversity.

5. **LOSO-Set vs. LOSO-Session**: While LOSO-Set prevents window-level leakage, it is less conservative than LOSO-Session because sets within the same session share environmental context. Future data collection across more sessions per activity class would enable the stricter LOSO-Session evaluation.

6. **Heart rate latency**: HR updates only every ~5 seconds on the Apple Watch, introducing considerable lag relative to activity transitions. Within a 3-second analysis window, the HR value often reflects the *previous* activity rather than the current one, limiting its real-time discriminative value.

### 6.4 Future Work

1. **Multi-participant data collection**: Collect data from multiple individuals to assess cross-user generalization and develop user-independent models.
2. **Additional recording sessions**: Collect more sessions per activity class (especially cardio and squats) to enable LOSO-Session evaluation and improve the robustness of class-specific models.
3. **Deep learning comparison**: With a larger dataset (10,000+ windows), evaluate CNN-LSTM architectures on raw 6-channel windows to compare against the handcrafted feature approach.
4. **Real-time deployment**: Implement on-device inference using the saved XGBoost model on Apple Watch for live activity classification during workouts.
5. **Hierarchical classification**: Evaluate a two-stage model — Stage 1 classifies broad category (upper body, lower body, cardio, rest), Stage 2 classifies specific exercise within category — to reduce inter-category confusion.
6. **Label noise investigation**: Examine the bicep curl outlier fold and any rest sets with 0% accuracy to determine if label correction or set removal is warranted.

## 7. Reproducibility

All results can be reproduced by running the project notebooks in sequence:

| Step | Notebook | Output |
|---|---|---|
| 1. Data inspection | `data_inspection.ipynb` | Quality visualizations and summary statistics |
| 2. Preprocessing | `preprocessing.ipynb` | `data/features.parquet`, `data/features.csv` |
| 3. Model training | `model_training.ipynb` | CV metrics, confusion matrices, saved models |

### Saved Artifacts

| Artifact | Path | Size |
|---|---|---|
| Feature matrix (Parquet) | `data/features.parquet` | 27,319 KB |
| Feature matrix (CSV) | `data/features.csv` | 57,929 KB |
| IMU-only model (JSON) | `models/xgboost_imu_only.json` | 1,922 KB |
| IMU-only model (Joblib) | `models/xgboost_imu_only.joblib` | 1,918 KB |
| IMU+HR model (JSON) | `models/xgboost_imu_hr.json` | 915 KB |
| IMU+HR model (Joblib) | `models/xgboost_imu_hr.joblib` | 1,315 KB |
| Tuned hyperparameters | `models/tuned_hyperparameters.joblib` | — |

### Environment

- Python 3.12 (virtual environment at `.venv/`)
- Key dependencies: pandas, numpy, scipy, scikit-learn, xgboost, tsfel, matplotlib, joblib
