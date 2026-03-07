# Exercise Activity Recognition with Apple Watch Sensor Data

A CON E 652 (Construction Operations Modeling and Technology) project that designs, implements, and evaluates an activity recognition prediction model for classifying exercise activities using wrist-worn IMU sensor data.

## Overview

This project uses the **SensorLog** app on Apple Watch to capture 6-axis IMU data (accelerometer + gyroscope) at 50 Hz during gym exercises. The sensor data is exported as labeled CSV files and processed through a machine learning pipeline to classify exercise types in real time.

## Activity Classes

| Label | Activity | Description |
|-------|----------|-------------|
| 0 | Rest | Non-exercise activity; rest between sets; transition |
| 1 | Pull-ups | Pull-ups using pull-up/dip rack |
| 2 | Bicep curls | Bicep curls using dumbbell weights |
| 3 | Lateral raises | Lateral raises using dumbbell weights |
| 4 | Cardio | Outdoor walking or running |
| 5 | Squats | Goblet squats using dumbbell weight |

## Data Collection

- **Device:** Apple Watch with SensorLog app (v6.1)
- **Sampling rate:** 50 Hz (all sensors simultaneously)
- **Sensors captured:**
  - CMDeviceMotion: user acceleration (gravity-removed), rotation rate, attitude (yaw/roll/pitch + quaternion), gravity vector
  - Raw accelerometer (X/Y/Z in G-force)
  - Heart rate (BPM, BPS, HRV)
- **Labeling:** Integer labels assigned in real time via SensorLog's built-in label feature
- **Export format:** CSV with descriptive column headers

## Pipeline

1. **Preprocessing:** Load CSV, resample to uniform 50 Hz, apply 3rd-order Butterworth low-pass filter (20 Hz cutoff), compute derived signals, z-score normalize per session
2. **Windowing:** 3-second windows with 50% overlap (150 samples/window)
3. **Feature extraction:** Time-domain and frequency-domain features via TSFEL (65+ features per axis)
4. **Classification:** XGBoost baseline; CNN-LSTM for advanced modeling
5. **Evaluation:** Leave-one-session-out cross-validation

## Project Structure

```
Activity_Recognition/
├── README.md
├── CLAUDE.md
├── pilot_test.ipynb            # Preliminary data suitability analysis
├── data/
│   ├── DATASET_README.md       # Label definitions and descriptions
│   └── stream_Apple_Watch_sample.csv
└── .venv/                      # Python 3.12 virtual environment
```

## Setup

```bash
# Activate the virtual environment
source .venv/bin/activate

# Install dependencies (as needed)
pip install pandas numpy scipy scikit-learn xgboost tsfel matplotlib
```

## References

- [SensorLog](https://sensorlog.berndthomas.net/) — iOS/watchOS sensor logging app
- [TSFEL](https://github.com/fraunhoferportugal/tsfel) — Time Series Feature Extraction Library
- [uLift dataset](https://doi.org/10.1109/ACCESS.2024.XXXXXXX) — Wrist-worn accelerometer workout tracker (15 exercises, 35 participants)
- [RecGym dataset](https://archive.ics.uci.edu/ml/datasets/) — UCI benchmark (12 gym exercises)
