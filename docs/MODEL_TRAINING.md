# Model Training

**Project:** Exercise Activity Recognition from Apple Watch Sensor Data <br>
**Course:** CON E 652 — Construction Operations Modeling and Technology <br>
**Date:** March 2026

---

## Related Documents

- [Data Collection](DATA_COLLECTION.md) — Apparatus, protocol, and dataset characteristics
- [Preprocessing Methodology](PREPROCESSING_METHODOLOGY.md) — Signal processing and feature extraction pipeline
- [Evaluation Results](EVALUATION_RESULTS.md) — Performance metrics, ablation study, and discussion
- [Project Feasibility Report](PROJECT_FEASIBILITY_REPORT.md) — Literature review and benchmark analysis

---

## 1. Classifier Selection: XGBoost

**XGBoost** (eXtreme Gradient Boosting; Chen & Guestrin, 2016) was selected as the classifier for this project. XGBoost is an ensemble method that builds decision trees sequentially, each correcting the residual errors of its predecessors, with L1 and L2 regularization terms integrated directly into the tree-building objective.

### Rationale

The feature matrix produced by preprocessing consists of **2,446 windows x 1,275 features** — a moderate-sized tabular dataset. This data regime favors gradient-boosted trees over deep learning for several reasons:

1. **Dataset size**: With ~2,400 training windows, the dataset is well below the 10,000+ windows typically recommended for deep learning approaches (CNN-LSTM). Boosted trees are less prone to overfitting on small-to-moderate datasets.
2. **Feature type**: The handcrafted TSFEL features are tabular numeric data — the domain where gradient boosting consistently outperforms neural networks in comparative benchmarks.
3. **Interpretability**: XGBoost provides feature importance scores (gain-based), enabling analysis of which sensor channels and feature types contribute most to classification.
4. **Training efficiency**: A full 58-fold cross-validation completes in a few minutes, compared to hours for equivalent deep learning evaluation.

Comparative studies have shown that XGBoost with engineered features matches or outperforms CNN and LSTM models on standard HAR benchmarks, with dramatically lower training time.

All model training code is implemented in `model_training.ipynb`.

## 2. Feature Pre-Selection

### Method

A **variance-based filter** retains the top 300 IMU features with the highest variance across the dataset, discarding low-information features. Because variance is a property of the input distribution alone — independent of class labels — applying this filter before cross-validation introduces **no data leakage**.

### Configurations

Two feature configurations are maintained for the ablation study:

| Configuration | Selection Rule | Feature Count |
|---|---|---|
| **IMU-only** | Top 300 IMU features by variance | 300 |
| **IMU+HR** | Top 300 IMU features + all 5 HR features | 305 |

For the IMU+HR configuration, all 5 heart rate features (`hr_mean`, `hr_std`, `hr_min`, `hr_max`, `hr_range`) are appended unconditionally to the top-300 IMU features. This ensures heart rate information is always available to the model regardless of whether HR features would have passed the variance threshold on their own.

## 3. Cross-Validation Strategy: Leave-One-Set-Out

### Why Not Random K-Fold?

Random k-fold cross-validation violates the temporal independence assumption of time-series data. Because consecutive windows overlap by 50% (75 shared samples), a randomly assigned test window is almost certainly adjacent to a training window that shares the majority of its raw samples. This constitutes **direct data leakage** — the model is evaluated on data it has effectively already seen — and leads to optimistically biased accuracy estimates.

### Why Not Leave-One-Session-Out?

Leave-One-Session-Out (LOSO by recording file) is the standard approach in HAR literature. However, this dataset has a structural constraint: **cardio data exists only in session S2, and squats data exists only in session S3**. Holding out S2 would give the model zero cardio training examples; holding out S3 would give it zero squats examples. The resulting metrics would reflect missing training data rather than model capacity.

### LOSO-Set Design

A **set** is a single contiguous exercise or rest bout (e.g., one set of pull-ups, one rest period between sets). Each set is assigned a unique `set_id` in the feature matrix. In Leave-One-Set-Out cross-validation:

1. One set is held out as the test fold
2. The model trains on all windows from all other sets
3. This repeats for every set, yielding **58 folds** (one per unique set_id)

This approach provides two key guarantees:

- **No data leakage from overlapping windows** — all windows from a given set move together into the test fold, so no training window shares raw samples with any test window.
- **All classes remain in training at every fold** — even when one of the two cardio sets is held out, the other remains in training. The model is never asked to predict a class it has never seen.

### Fold Coverage

| Activity | Folds Evaluating This Class | Total Folds |
|---|---|---|
| Rest | 30 | 58 |
| Pull-ups | 9 | 58 |
| Bicep curls | 6 | 58 |
| Lateral raises | 7 | 58 |
| Cardio | 2 | 58 |
| Squats | 4 | 58 |

### Known Limitation

LOSO-Set is **less conservative** than LOSO-Session because windows from the same session (but different sets) share session-level characteristics — sensor calibration state, watch position, fatigue level. The LOSO-Set results may therefore be slightly optimistic compared to true session-level holdout. This limitation is imposed by the single-session representation of cardio and squats, and will improve as additional diverse recording sessions are collected.

## 4. Class Imbalance Correction

### Problem

The rest class accounts for approximately 33% of windows (815 out of 2,446), roughly 2–3x more frequent than each exercise class. Without correction, a classifier can achieve high raw accuracy by biasing predictions toward rest while performing poorly on the exercise classes that matter most for the application.

### Solution: Balanced Sample Weights

Rather than physically resampling the data (e.g., SMOTE oversampling or random undersampling), **balanced sample weights** are applied. Each training window receives a weight inversely proportional to its class frequency:

> w_i = N / (K x N_yi)

where N is the total number of training windows, K is the number of classes (6), and N_yi is the count of windows in class y_i.

Sample weights are recomputed **inside each CV fold** using only the training split, ensuring that the weight calculation does not leak information from the test fold. This approach is preferred over physical resampling because it avoids creating synthetic data artifacts and works directly with XGBoost's weighted loss function.

## 5. Hyperparameter Tuning

### Strategy

Hyperparameters are tuned using **RandomizedSearchCV** with an inner cross-validation loop that is separate from the outer LOSO-Set evaluation:

| Component | Configuration |
|---|---|
| Search method | Randomized search (30 random combinations) |
| Inner CV | GroupKFold (5 splits), grouped by `set_id` |
| Scoring metric | Macro F1-score (`f1_macro`) |
| Outer evaluation | Full LOSO-Set CV (58 folds) |

This separation prevents circular optimization bias — hyperparameters are not tuned on the same folds used for final performance reporting. The GroupKFold inner CV respects the set structure, preventing data leakage during tuning.

### Search Space

| Hyperparameter | Candidate Values |
|---|---|
| `n_estimators` | 50, 100, 150, 200, 300 |
| `learning_rate` | 0.01, 0.05, 0.1, 0.2 |
| `max_depth` | 3, 4, 5, 6, 8 |
| `subsample` | 0.6, 0.7, 0.8, 0.9, 1.0 |
| `colsample_bytree` | 0.3, 0.5, 0.7, 0.9 |
| `min_child_weight` | 1, 3, 5, 7 |
| `gamma` | 0, 0.1, 0.3, 0.5 |
| `reg_alpha` | 0, 0.01, 0.1, 1.0 |
| `reg_lambda` | 0.5, 1.0, 2.0, 5.0 |

### Fixed Parameters

| Parameter | Value |
|---|---|
| `objective` | `multi:softprob` |
| `eval_metric` | `mlogloss` |
| `num_class` | 6 |
| `tree_method` | `hist` |
| `random_state` | 42 |

### Tuning Results

Tuning was performed independently for both configurations:

**IMU-only model** (tuning time: 119.6 seconds):

| Hyperparameter | Best Value |
|---|---|
| `colsample_bytree` | 0.3 |
| `gamma` | 0.5 |
| `learning_rate` | 0.01 |
| `max_depth` | 8 |
| `min_child_weight` | 5 |
| `n_estimators` | 100 |
| `reg_alpha` | 1.0 |
| `reg_lambda` | 5.0 |
| `subsample` | 0.6 |

Inner CV best macro F1: **0.8941**

**IMU+HR model** (tuning time: 114.4 seconds):

| Hyperparameter | Best Value |
|---|---|
| `colsample_bytree` | 0.9 |
| `gamma` | 0.3 |
| `learning_rate` | 0.2 |
| `max_depth` | 5 |
| `min_child_weight` | 3 |
| `n_estimators` | 50 |
| `reg_alpha` | 0 |
| `reg_lambda` | 0.5 |
| `subsample` | 0.8 |

Inner CV best macro F1: **0.8176**

The two configurations converged on notably different hyperparameter profiles: the IMU-only model favors a deeper, more regularized ensemble with slow learning (depth 8, 100 trees, lr 0.01, strong L1/L2), while the IMU+HR model favors a shallower, faster-learning ensemble with less regularization (depth 5, 50 trees, lr 0.2). This suggests the additional HR features provide complementary signal that allows a simpler model to achieve good performance.

## 6. Ablation Study Design

### Purpose

The ablation study quantifies the **marginal contribution of heart rate** to activity recognition accuracy. Heart rate provides physiological context that motion sensors alone cannot capture — for example, elevated HR during cardio or post-exertion HR during rest periods — but the relationship is indirect and delayed, and a wrist-worn IMU may already provide sufficient discriminative information from motion patterns alone.

### Experimental Design

Both configurations are evaluated under identical conditions:

| Aspect | IMU-only | IMU+HR |
|---|---|---|
| Features | 300 (top-variance IMU) | 305 (300 IMU + 5 HR) |
| Cross-validation | LOSO-Set (58 folds) | LOSO-Set (58 folds) |
| Sample weights | Balanced (per-fold) | Balanced (per-fold) |
| Hyperparameters | Independently tuned | Independently tuned |

The only difference between the two configurations is the presence or absence of the 5 heart rate summary statistics. Results are compared on the same metrics — accuracy, balanced accuracy, macro F1, and per-class recall — to isolate the effect of HR features.

### References

- Chen, T. & Guestrin, C. (2016). XGBoost: A Scalable Tree Boosting System. *KDD 2016*.

## 7. Model Persistence

After cross-validation evaluation, final models are trained on the **entire dataset** (all 2,446 windows from all 58 sets) using the tuned hyperparameters with 300 boosting rounds. These deployment-ready models are saved in two formats:

| Model | JSON | Joblib |
|---|---|---|
| `xgboost_imu_only` | 1,922 KB | 1,918 KB |
| `xgboost_imu_hr` | 915 KB | 1,315 KB |

Tuned hyperparameters for both configurations are saved to `models/tuned_hyperparameters.joblib` for reproducibility.

The expected real-world performance of these models is estimated by the LOSO-Set CV metrics reported in the [Evaluation Results](EVALUATION_RESULTS.md).
