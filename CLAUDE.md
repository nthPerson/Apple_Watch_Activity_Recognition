# Activity Recognition Project — Claude Code Guidelines

## Project Overview
This project implements data analysis, experiments, and evaluations for an exercise activity recognition system as part of CON E 652 (Construction Operations Modeling and Technology). The goal is to design, implement, and evaluate a prediction model that classifies various exercise activities from Apple Watch sensor data collected via the SensorLog app.

## Label Mappings (Authoritative — from data/DATASET_README.md)
| Label | Variable | Description |
|-------|----------|-------------|
| 0 | `rest` | Non-exercise activity; rest between sets; transition |
| 1 | `pull-ups` | Pull-ups using pull-up/dip rack |
| 2 | `bicep-curls` | Bicep curls using dumbbell weights |
| 3 | `lateral-raises` | Lateral raises using dumbbell weights |
| 4 | `cardio` | Outdoor walking or running |
| 5 | `squats` | Goblet squats using dumbbell weight |

**IMPORTANT:** The `pilot_test.ipynb` notebook uses outdated/contradictory label mappings (e.g., label 2 = Dips, label 3 = Bicep curl, only 4 classes). Always use the mappings above from `data/DATASET_README.md` as the single source of truth.

## Data Collection
- **Device:** Apple Watch with SensorLog app
- **Sampling rate:** 50 Hz
- **Sensors:** 6-axis IMU (accelerometer + gyroscope via CMDeviceMotion), heart rate
- **Export format:** CSV files with built-in integer labeling
- **Primary time reference:** `motionTimestamp_sinceReboot(s)` column
- **Magnetometer:** Disabled (metal interference in gym environments)

## Key Sensor Columns
- `accelerometerAcceleration{X,Y,Z}(G)` — raw accelerometer
- `motionUserAcceleration{X,Y,Z}(G)` — gravity-removed acceleration
- `motionRotationRate{X,Y,Z}(rad/s)` — gyroscope rotation rates
- `motionGravity{X,Y,Z}(G)` — gravity vector
- `motionYaw(rad)`, `motionRoll(rad)`, `motionPitch(rad)` — attitude
- `motionQuaternion{X,Y,Z,W}(R)` — quaternion attitude
- `heartRateBPM(N)` — heart rate

## Preprocessing Pipeline
1. Load CSV into pandas; use `motionTimestamp_sinceReboot(s)` as time index
2. Resample to uniform 50 Hz grid (handle timestamp jitter)
3. Apply 3rd-order Butterworth low-pass filter at 20 Hz cutoff
4. Compute derived signals: acceleration magnitude, rotation magnitude, jerk
5. Z-score normalize per session

## Windowing
- 3-second windows with 50% overlap (150 samples per window at 50 Hz)

## Modeling Approaches
- **Baseline:** Handcrafted features (TSFEL) + XGBoost (target 90-96% accuracy)
- **Advanced:** CNN-LSTM on raw 6-channel windows (needs >= 10k windows)
- **Hierarchical:** Stage 1 = broad category, Stage 2 = specific exercise
- **Cross-validation:** Leave-one-session-out (never random k-fold on time-series)

## Project Structure
```
Activity_Recognition/
├── CLAUDE.md                 # This file
├── README.md                 # Project README
├── pilot_test.ipynb          # Preliminary analysis (uses OUTDATED labels)
├── data/
│   ├── DATASET_README.md     # Authoritative label definitions
│   └── stream_Apple_Watch_sample.csv  # Sample sensor data
└── .venv/                    # Python virtual environment
```

## Development Notes
- Python 3.12 virtual environment at `.venv/`
- Activate with: `source .venv/bin/activate`
- Key libraries to use: pandas, numpy, scipy, scikit-learn, xgboost, tsfel, matplotlib
