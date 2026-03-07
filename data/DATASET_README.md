# Exercise Activity Recognition Project Dataset Details

## Label Indexes and Descriptions
| Label | Variable | Description |
|-------|----------|-------------|
| 0 | `rest` | Non-exercise activity; rest between sets; transition |
| 1 | `pull-ups` | Pull-ups using pull-up/dip rack |
| 2 | `bicep-curls` | Bicep curls using dumbbell weights |
| 3 | `lateral-raises` | Lateral raises using dumbbell weights |
| 4 | `cardio` | Outdoor walking or running |
| 5 | `squats` | Goblet squats using dumbbell weight |

## Dataset Headers
| Column | Description |
|--------|-------------|
| loggingTime(txt) | Timestamp when the data point was logged |
| accelerometerTimestamp_sinceReboot(s) | Time since device reboot for accelerometer reading (seconds) |
| accelerometerAccelerationX(G) | Raw accelerometer X-axis acceleration (gravitational units) |
| accelerometerAccelerationY(G) | Raw accelerometer Y-axis acceleration (gravitational units) |
| accelerometerAccelerationZ(G) | Raw accelerometer Z-axis acceleration (gravitational units) |
| motionTimestamp_sinceReboot(s) | Time since device reboot for motion data (seconds) |
| motionYaw(rad) | Rotation around Z-axis in radians |
| motionRoll(rad) | Rotation around X-axis in radians |
| motionPitch(rad) | Rotation around Y-axis in radians |
| motionRotationRateX(rad/s) | Angular velocity around X-axis (radians/second) |
| motionRotationRateY(rad/s) | Angular velocity around Y-axis (radians/second) |
| motionRotationRateZ(rad/s) | Angular velocity around Z-axis (radians/second) |
| motionUserAccelerationX(G) | Gravity-removed acceleration X-axis (gravitational units) |
| motionUserAccelerationY(G) | Gravity-removed acceleration Y-axis (gravitational units) |
| motionUserAccelerationZ(G) | Gravity-removed acceleration Z-axis (gravitational units) |
| motionAttitudeReferenceFrame(txt) | Reference frame used for attitude calculations |
| motionQuaternionX(R) | Quaternion X component for device orientation |
| motionQuaternionY(R) | Quaternion Y component for device orientation |
| motionQuaternionZ(R) | Quaternion Z component for device orientation |
| motionQuaternionW(R) | Quaternion W component for device orientation |
| motionGravityX(G) | Estimated gravity vector X-component (gravitational units) |
| motionGravityY(G) | Estimated gravity vector Y-component (gravitational units) |
| motionGravityZ(G) | Estimated gravity vector Z-component (gravitational units) |
| motionMagneticFieldX(µT) | Magnetic field strength X-axis (microtesla) |
| motionMagneticFieldY(µT) | Magnetic field strength Y-axis (microtesla) |
| motionMagneticFieldZ(µT) | Magnetic field strength Z-axis (microtesla) |
| motionHeading(°) | Compass heading in degrees |
| motionMagneticFieldCalibrationAccuracy(Z) | Calibration accuracy level of magnetometer |
| deviceID(txt) | Unique identifier for the Apple Watch device |
| heartRateBPMTimestamp(txt) | Timestamp for heart rate measurement |
| heartRateBPM(N) | Heart rate in beats per minute |
| heartRateBPS(R) | Heart rate sample rate or timestamp (seconds) |
| heartRateVariabilityTimestamp(txt) | Timestamp for heart rate variability measurement |
| heartRateVariability(ms) | Time variation between heartbeats (milliseconds) |
| label | Activity class label (0-5 per Label Indexes mapping) |


