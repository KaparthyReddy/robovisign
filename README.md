# robovisign : Autonomous Traffic Sign Recognition — ROS 2

A real-time autonomous robotic system that detects and classifies traffic signs in a simulated environment and executes appropriate robot actions. Built on ROS 2 with a structured 4-stage pipeline — Perception, Cognition, Decision, and Actuation — combining computer vision, deep learning, and robotic control.

---

## Problem Statement

Autonomous robotic systems operating in structured environments must reliably perceive and interpret visual cues such as traffic signs. Errors in perception or delayed responses can lead to incorrect actions, making this a critical challenge in robotics and autonomous navigation. This project addresses the problem of real-time traffic sign recognition and decision execution in a simulated environment using a CNN-based classification pipeline integrated with ROS 2.

---

## Pipeline Overview
```text
Camera Feed (Gazebo)
|
v
┌───────────────────────────────────────────┐
│              4-Stage Pipeline             │
│                                           │
│  Perception → Cognition → Decision → Actuation │
│                                           │
│  HSV masking    CNN classify   Rule map   TwistStamped  │
│  ROI extract    GTSRB model    Fail-safe  /cmd_vel pub  │
└───────────────────────────────────────────┘
|
v
Robot Action
(Stop / Speed 0.2 / 0.4 / 0.6 m/s)
```
---

## System Architecture

### Stage 1 — Perception
Real-time image frames are captured from the robot's simulated camera in Gazebo. Each frame is processed using HSV color-based masking to isolate red-colored regions — the dominant color in regulatory traffic signs. Two HSV ranges are combined (hue 0–10 and 170–180) to capture all red shades. Morphological closing fills holes in the mask, and contour detection extracts candidate bounding boxes sorted by area.

### Stage 2 — Cognition
Extracted regions of interest (ROIs) are preprocessed and passed to the CNN. Preprocessing involves resizing to 32×32 pixels, histogram equalization on the Y channel (YUV space) to normalize lighting, normalization to [0,1], and dimension expansion for batch inference. The CNN classifies each ROI into one of 43 traffic sign categories from the GTSRB dataset.

### Stage 3 — Decision
The classification output is passed through a rule-based decision engine. A heuristic correction layer cross-verifies predictions against the color context — if the CNN predicts a blue sign class but the ROI was extracted from a red mask, the prediction is overridden to the most likely red sign. Low-confidence predictions fall back to a safe state. Valid predictions are mapped to robot actions.

### Stage 4 — Actuation
The selected action is published as a `TwistStamped` velocity command on `/cmd_vel`, which the robot executes in real time within the Gazebo simulation.

---

## Sign-to-Action Mapping

| Detected Sign | Robot Action |
|---------------|-------------|
| STOP | Speed = 0.0 m/s (full stop) |
| Speed 30 | Speed = 0.2 m/s |
| Speed 60 | Speed = 0.4 m/s |
| Speed 100 | Speed = 0.6 m/s |
| Unknown / low confidence | Safe state (hold current) |

---

## Key Algorithms

### CNN — Traffic Sign Classification
Trained on the GTSRB (German Traffic Sign Recognition Benchmark) dataset. Architecture: Convolution → ReLU → MaxPooling → Flatten → Dense → Softmax. Outputs class probabilities across 43 sign categories.

### HSV Color-Based Region Detection
Converts BGR frames to HSV, applies dual red masks, performs morphological closing, finds external contours, filters by minimum area, clips bounding boxes to frame boundaries, and returns boxes sorted largest-first.

### Histogram Equalization (YUV)
Equalizes only the Y (luminance) channel — fixes lighting variations without altering color. Critical for consistent CNN input across varying Gazebo illumination conditions.

### Heuristic Error Correction
Rule-based override: if CNN predicts a blue sign class within a red-masked ROI, the prediction is corrected to the appropriate red sign. Handles model hallucinations caused by BGR/RGB domain shift.

### Rule-Based Control Logic
Lightweight, interpretable decision mapping from sign class to velocity command. Ensures predictable and safe robot behavior in all conditions.

---

## Project Structure
```text
autonomous-traffic-sign-recognition/
├── traffic_sign_node_ros2.py        # Main vision + decision ROS 2 node
├── robot_controller.py              # Translates decisions to velocity commands
├── preprocess.py                    # HSV detection + image preprocessing utilities
├── visualizer_ros2.py               # ROS 2 HUD overlay for camera feed
├── run_traffic_sign_dashboard.py    # Standalone real-time monitoring dashboard
├── test_model.py                    # CNN model evaluation and testing
├── traffic_sign_model_clean.keras   # Pre-trained CNN model weights
├── sign20.png                       # Test images
├── sign30.jpeg
├── sign50.jpg
├── sign80.png
├── stop.jpg
├── stop.png
├── synthetic_stop.jpg
├── turn_left.png
├── requirements.txt
└── README.md
```
---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Robotics Middleware | ROS 2 Jazzy |
| Simulation | Gazebo |
| Computer Vision | OpenCV 4.x |
| Deep Learning | TensorFlow / Keras |
| Language | Python 3.10+ |
| Image Processing | NumPy, CvBridge |
| ROS Client Library | rclpy |

---

## ROS 2 Topics

| Topic | Direction | Description |
|-------|-----------|-------------|
| `/annotated_image` | Published | Camera feed with detection overlay |
| `/detected_sign` | Published | String label of detected sign |
| `/cmd_vel` | Published | TwistStamped velocity command |

---

## How to Run

### Prerequisites
- ROS 2 Jazzy or Humble
- Python 3.10+
- Gazebo Sim
- TensorFlow environment

### Install dependencies
```bash
pip install -r requirements.txt
```

### Run the main detection node
```bash
# From camera (index 0)
python3 traffic_sign_node_ros2.py

# From image file
python3 traffic_sign_node_ros2.py --source stop.jpg

# From video file
python3 traffic_sign_node_ros2.py --source video.mp4
```

### Run the standalone dashboard
```bash
python3 run_traffic_sign_dashboard.py
```

### Monitor ROS 2 topics
```bash
ros2 topic echo /detected_sign
ros2 topic echo /cmd_vel
```

---

## Fail-Safe Logic

The system incorporates three levels of safety:

1. **Confidence threshold** — predictions below 0.5 confidence fall back to safe state, no action taken
2. **Heuristic correction** — color-context validation overrides impossible CNN predictions
3. **Dummy frame mode** — if no camera is available, system generates placeholder frames and continues operating without crashing

---

## Outcomes

- Accurate real-time classification of red traffic signs (STOP, speed limits)
- Correct robot velocity execution mapped to detected signs
- Robust operation under varying Gazebo lighting conditions
- Live monitoring dashboard showing detection results and robot state
- Modular ROS 2 architecture — each node operates independently

---

## Future Improvements

- Extend detection to blue and yellow sign categories
- Train on larger and more diverse datasets
- Deploy on physical hardware (TurtleBot, Jetson Nano)
- Add lane detection and multi-sign handling
- Integrate with full autonomous navigation stack (Nav2)

---

## Author

**Kaparthy Reddy**
