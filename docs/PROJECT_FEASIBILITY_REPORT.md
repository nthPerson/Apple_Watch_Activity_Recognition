# SensorLog_activity_recognition_on_Apple_Watch

# SensorLog + Apple Watch for gym exercise recognition

**SensorLog is a well-proven, research-grade iOS/watchOS sensor logging app that can capture 6-axis IMU data at up to 50–100 Hz from the Apple Watch — more than sufficient for building a hierarchical exercise classifier across your target activities.** The app costs ~$3–5 as a one-time purchase, supports CSV export with built-in data labeling, and has been used in published academic research. Combined with a straightforward feature engineering pipeline and XGBoost or a CNN-LSTM model, you can realistically achieve **90–96% classification accuracy** on gym exercises using only wrist-worn sensor data. This report covers every step from app configuration through model evaluation.

---

## SensorLog captures rich motion data on Apple Watch

SensorLog (developed by Dr. Bernd Thomas, active since 2010, currently v6.1) logs the following sensors on Apple Watch:

**Core motion sensors (most important for your experiment):**
- **CMDeviceMotion** (sensor-fused): user acceleration X/Y/Z (gravity removed, in G), rotation rate X/Y/Z (rad/s), attitude as yaw/roll/pitch (rad) and quaternion (X/Y/Z/W), gravity vector X/Y/Z (G), calibrated magnetic field X/Y/Z (µT), and heading (°)
- **Raw accelerometer**: acceleration X/Y/Z in G-force units
- **Heart rate**: BPM, BPS, and heart rate variability (seconds) via HealthKit

**Supplementary sensors:** barometric altimeter (relative altitude in meters, pressure in kPa), pedometer (steps, distance, pace, cadence, floors), activity classifier (Apple’s built-in walking/running/stationary detection), GPS location, and battery state.

Notably, the Apple Watch does **not** expose raw gyroscope or raw magnetometer as separate channels in SensorLog. However, the CMDeviceMotion fusion pipeline includes rotation rate data derived from the gyroscope and calibrated magnetic field — so you effectively get **6-axis IMU data** (3-axis acceleration + 3-axis rotation rate) plus attitude estimation through the fused motion output.

**Sampling rates** are the critical configuration decision. On Apple Watch, SensorLog supports up to **100 Hz when logging a single sensor** and up to **50 Hz when logging all sensors simultaneously**. For exercise recognition research, **50 Hz is the recommended rate** — it matches the UCI HAR benchmark standard, aligns with Apple’s own WWDC recommendations for workout analysis, and a 2025 systematic review in *Sports Medicine* found that models performing best captured data between **100 and 128 Hz**, though the accuracy difference versus 50 Hz is marginal for most gym exercises. At 50 Hz with all sensors enabled, you get rich multi-modal data without excessive file sizes or battery drain.

**Background logging has important constraints.** For sessions under 1 hour, SensorLog can log all sensors at up to 50 Hz in the background. For sessions exceeding 1 hour, only accelerometer data (no gyroscope/device motion) is available in the background at 50 Hz. Since a typical gym session runs 45–90 minutes, you should either keep the SensorLog app in the foreground on the Watch (which may require periodic wrist-raises to prevent screen sleep) or structure your data collection into sub-1-hour recording blocks. User reports confirm that logging can pause when the wrist drops and the screen sleeps, so **starting an HKWorkoutSession or keeping the screen active** is advisable during recording.

---

## CSV file export is the best path for your ML workflow

SensorLog supports seven data export methods, but for a Python/Jupyter research pipeline, **local CSV file logging** is the clear winner:

**Local CSV logging** records sensor data directly to files on the Apple Watch or iPhone at full fidelity (up to 100 Hz). The CSV uses configurable field separators (default comma) with a header row containing descriptive column names like `motionUserAccelerationX(G)`, `motionRotationRateY(rad/s)`, `motionQuaternionW(R)`, and `accelerometerTimestamp_sinceReboot(s)`. Each row represents one sampling instant. Files transfer to the iPhone via WatchConnectivity (Bluetooth/WiFi), then export via AirDrop, Files app, or Finder file sharing. The `label(N)` column stores integer labels you assign during recording — this is SensorLog’s built-in data annotation feature, invaluable for marking exercise segments in real time.

**A critical timestamp detail**: SensorLog uses two different timestamp bases within the same CSV row. Motion sensors (accelerometer, gyroscope, device motion) use `sinceReboot(s)` timestamps (seconds since device boot), while location and battery sensors use `since1970(s)` (Unix epoch). The `loggingTime(txt)` column provides an ISO 8601 human-readable timestamp for when the row was written, but this does not precisely represent when each sensor sample was captured. For your preprocessing pipeline, use the `motionTimestamp_sinceReboot(s)` column as your primary time reference for IMU data and convert to relative seconds from session start.

**HTTP POST** (JSON or form-encoded, up to 10 Hz on Watch, 20 Hz on iPhone) is useful for real-time streaming to a server but is too slow for exercise recognition — you need ≥25 Hz. **TCP/IP streaming** supports up to 100 Hz but requires same-network connectivity and has been partially removed from newer Watch versions due to watchOS networking changes. **UDP streaming** trades reliability for lower latency but risks data loss. For a batch-processing ML workflow where you collect sessions and analyze them later, these real-time methods add unnecessary complexity.

**Recommended workflow**: Record sessions as CSV files at 50 Hz on the Watch → transfer to iPhone → AirDrop or share to your Mac → load into pandas DataFrames. SensorLog also supports a “Machine Learning Framework friendly log file format” option designed to simplify import into ML tools.

---

## Which sensors matter most for each exercise category

Not all sensors contribute equally to discriminating your target exercises. Research consistently shows that the **accelerometer + gyroscope combination (6-axis IMU)** is the primary discriminator, with heart rate and altimeter serving as useful supplementary signals.

**Upper body exercises** (dumbbell curls, presses, rows, pull-ups, dips) produce the strongest and most distinctive signals on a wrist-worn sensor. Gyroscope data is particularly critical here — rotation rate patterns differ substantially between exercises that might look similar in acceleration alone. For example, the forearm rotation during a bicep curl versus the relatively stable wrist during a pull-up creates clearly separable gyroscope signatures. The 2025 *Sports Medicine* systematic review found that wrist-worn IMUs achieve **>90% accuracy** for upper-body exercise classification. The fused Device Motion quaternion data (attitude) also helps distinguish pressing angles: incline bench produces a different gravity vector orientation than flat or decline, making `motionPitch` and `motionRoll` valuable features.

**Cardio activities** (running, walking) are the easiest to classify. The **accelerometer alone** is sufficient — the periodic gait pattern is highly distinctive, and these activities separate cleanly from resistance exercises in feature space. Pedometer cadence and heart rate provide additional discrimination but are rarely necessary. Walking versus running separates primarily on acceleration magnitude and stride frequency.

**Core exercises** (ab roller) present the biggest challenge for wrist-worn sensors because wrist motion is relatively constrained during ab roller movements. Here, the **gravity vector from Device Motion** (capturing torso angle changes through the wrist) and heart rate elevation become more important supplementary features.

**Skip the magnetometer in your gym environment.** While some studies found that 9-axis data (adding magnetometer) improves accuracy, ferromagnetic interference from dumbbells, barbells, weight machines, and building structures makes magnetometer readings unreliable in gym settings. Most published gym exercise studies explicitly exclude magnetometer data and rely on 6-axis accelerometer + gyroscope.

---

## Structuring your data collection for robust classification

A well-designed data collection protocol directly determines model quality. Based on published HAR experiment designs, here is a concrete protocol for your exercises:

**Session structure**: Each recording session should cover 3–5 exercises performed in a randomized order to prevent temporal bias. For each exercise, perform **3 sets of 8–12 reps** with 30–60 second rest periods between sets. Use SensorLog’s label feature to assign a unique integer to each exercise (e.g., 1=bicep curl, 2=shoulder press, 3=pull-up, etc.) and a distinct label for rest periods (e.g., 0=rest/transition). Change the label at the start and end of each set. Include 2–3 seconds of quiet standing before and after each set to create clean transition boundaries.

**Data volume targets**: Aim for **200–500 labeled windows per exercise class** at minimum. With 3-second windows at 50% overlap, each 10-rep set of bicep curls (~25–30 seconds) yields roughly 15–18 windows. Three sets produce ~50 windows per session. So you need **4–10 sessions per exercise** to reach adequate training volume. For a dataset of ~12 exercise types, plan for **15–25 total gym sessions** across multiple days to capture natural variability in form and fatigue. The RecGym benchmark used 10 volunteers × 5 sessions each — you can achieve comparable scale as a single participant by collecting across 15+ days.

**Labeling strategy**: Use a **hybrid approach** — tap SensorLog’s label button to set coarse real-time labels (exercise type), then refine boundaries post-hoc. Video recording your sessions with a phone propped on the floor provides ground truth for precise label alignment. A countdown timer or audible beep at set start/stop simplifies synchronization. Studies show classifiers tolerate ~15–20% label noise, so imperfect real-time labels still work well when combined with video review.

**Between-session variability matters.** Collect data across different days, times, fatigue levels, and (if possible) watch placements on the wrist. A model trained on data from a single session will overfit to your specific grip tightness, fatigue level, and watch position that day. Spreading collection across 2+ weeks produces substantially more robust classifiers.

---

## Windowing and preprocessing turn raw signals into model-ready features

**Windowing** segments continuous sensor streams into fixed-length analysis frames. For gym exercises, **2–4 second windows with 50% overlap** are optimal. A bicep curl repetition takes ~2–3 seconds, a pull-up ~3–4 seconds, and a walking stride ~1 second. A 3-second window at 50 Hz produces 150 samples per window — large enough to capture one full exercise repetition while small enough for responsive classification. The 50% overlap doubles your effective training data and ensures no repetition straddles a window boundary without being fully captured in at least one window.

**Preprocessing pipeline** (in order):

1. **Load and parse CSV**: Read with pandas, extract `motionTimestamp_sinceReboot(s)` as time index and all `motionUserAcceleration*`, `motionRotationRate*`, `motionGravity*`, and `heartRateBPM` columns.
2. **Resample to uniform frequency**: SensorLog timestamps may have slight jitter. Use `scipy.interpolate.interp1d` to resample all channels to exactly 50 Hz on a uniform time grid.
3. **Low-pass filter**: Apply a 3rd-order Butterworth low-pass filter at **20 Hz cutoff** using `scipy.signal.filtfilt()` (zero-phase) to remove high-frequency sensor noise while preserving all exercise motion content. The UCI HAR benchmark uses this exact configuration.
4. **Separate gravity components**: If using raw accelerometer instead of Device Motion, apply a 0.3 Hz Butterworth low-pass to extract gravity, then subtract to get body acceleration. If using Device Motion’s `userAcceleration` (gravity already removed) and `gravity` fields, this step is unnecessary — SensorLog’s CMDeviceMotion output provides pre-separated signals.
5. **Compute derived signals**: Calculate acceleration magnitude `√(ax² + ay² + az²)`, rotation rate magnitude, and jerk signals (first derivative of acceleration using `np.diff`).
6. **Normalize**: Apply z-score normalization per-session (subtract mean, divide by standard deviation) to account for variations in watch tightness and placement across sessions.

---

## Feature engineering versus deep learning on raw signals

You have two viable modeling approaches, and the right choice depends on your dataset size and goals.

**Approach 1: Handcrafted features + XGBoost** is the pragmatic starting point. For each 3-second window, extract ~30–60 features per sensor axis: mean, standard deviation, min, max, RMS, zero-crossing rate, interquartile range, skewness, kurtosis (time-domain); dominant frequency, spectral entropy, spectral energy, first 5 FFT coefficients (frequency-domain); plus cross-axis correlations and signal magnitude area. The Python library **TSFEL** (`pip install tsfel`) computes 65+ features automatically from windowed IMU data and is purpose-built for wearable HAR. Alternatively, **tsfresh** generates up to 794 features with built-in relevance filtering. Feed these into XGBoost or Random Forest. On published benchmarks, **XGBoost with engineered features achieves 90–96% accuracy** and actually matches or outperforms deep learning on most HAR tasks, with dramatically lower training time and full interpretability via feature importance scores. A comparative study found XGBoost misclassified only 281 instances on the MHEALTH dataset versus 1,341 for CNN and 2,742 for LSTM.

**Approach 2: CNN-LSTM on raw signals** is the state-of-the-art approach when you have ≥10,000 windows. Feed raw 6-axis IMU windows (shape: 150 timesteps × 6 channels at 50 Hz × 3 seconds) into a 1D CNN (2–3 convolutional layers with batch normalization) to extract local spatial features, followed by an LSTM layer to capture temporal dynamics, then a dense softmax output. This architecture achieved **97.29% accuracy** on the Myogym benchmark (30 exercises, 10 participants). Use PyTorch or TensorFlow/Keras. The key advantage is avoiding manual feature engineering entirely. The disadvantage is requiring substantially more data and compute.

**For hierarchical classification**, train a two-stage model. Stage 1 classifies the broad category (upper body, core, cardio, rest) using coarse features like overall acceleration magnitude statistics and heart rate. Stage 2 trains separate specialist classifiers within each category — e.g., an upper-body classifier that distinguishes curls, presses, pull-ups, and dips using fine-grained rotation rate patterns. This architecture reduces inter-class confusion between dissimilar exercises (running will never be confused with pull-ups) and allows you to optimize each specialist independently.

**Cross-validation**: Use **leave-one-session-out** cross-validation (train on all sessions except one, test on the held-out session, rotate). Never use random k-fold on time-series data — it leaks temporal information and inflates accuracy by 5–15 percentage points.

---

## Existing tools and codebases accelerate development

Several open-source projects provide directly relevant starting points:

- **uLift** (Lim et al., 2024, IEEE Access): A wrist-worn accelerometer workout tracker achieving **90% classification accuracy** across 15 exercises from 35 participants using dynamic time warping — no training required. The dataset is open-sourced and contains multi-joint exercises collected in a real gym.
- **blakete/Wearable-Data-Analysis** on GitHub: An Apple Watch ML training data engine that processes CSV exports, extracts features, and trains classifiers. Supports activities including running, sitting, and walking with 62K samples.
- **jtrejo13/fitness-learning** on GitHub: An Apple Watch app with real-time workout recognition using Core Motion accelerometer and gyroscope data, recognizing exercises including rowing, treadmill, and pushups.
- **TSFEL** (Fraunhofer Portugal): The single most useful library for your pipeline. It provides windowing, resampling, 65+ feature extraction functions (statistical, temporal, spectral, fractal), and includes a HAR example notebook. Install with `pip install tsfel` and use `tsfel.time_series_features_extractor()` on your windowed DataFrames.
- **RecGym dataset** (UCI Machine Learning Repository): 10 volunteers performing 12 gym exercises across 50 sessions with wrist-worn IMU at 20 Hz. Useful for benchmarking your pipeline against a known dataset before running on your own Apple Watch data.

For the complete data format reference, SensorLog’s official website (sensorlog.berndthomas.net) provides sample CSV files from both iPhone and Apple Watch Ultra recordings, along with a PHP example script for HTTP POST server endpoints.

---

## Conclusion

The complete pipeline from collection to classification breaks down into a clear sequence: configure SensorLog to log CMDeviceMotion + accelerometer + heart rate at **50 Hz** on Apple Watch, use the **integer label feature** to mark exercises during sets, export **CSV files** via AirDrop, preprocess with Butterworth filtering and z-score normalization, window into **3-second frames at 50% overlap**, extract features with **TSFEL**, and classify with **XGBoost** as your baseline model before experimenting with CNN-LSTM architectures. The hierarchical approach — first classifying exercise category, then specific exercise — will handle your diverse exercise set (upper body dumbbell work, bodyweight exercises, core, and cardio) most effectively.

Three non-obvious insights from this research: first, the magnetometer should be disabled entirely in gym environments due to metal interference, saving battery and avoiding noise. Second, SensorLog’s Device Motion output already provides gravity-separated acceleration and fused rotation rate, eliminating the most complex preprocessing step. Third, for a single-participant study, collecting across **15+ sessions on different days** matters more than collecting many exercises in a single marathon session — between-day variability in form, fatigue, and watch position is the primary source of real-world generalization challenge.