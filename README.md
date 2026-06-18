<div align="center">

# 🎯 Focus Tracker
### Real-Time Operator Attention Monitoring System

![Embedded AI](https://img.shields.io/badge/Embedded-AI-2563EB?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.14-3776AB?style=for-the-badge&logo=python&logoColor=white)
![STM32](https://img.shields.io/badge/STM32N6-Neural--ART_NPU-03234B?style=for-the-badge&logo=stmicroelectronics&logoColor=white)
![OpenCV](https://img.shields.io/badge/OpenCV-4.x-5C3EE8?style=for-the-badge&logo=opencv&logoColor=white)
![License](https://img.shields.io/badge/License-MIT-10B981?style=for-the-badge)

> *"Can internal attention be measured through external signals?"*

**A multi-modal sensor fusion system that classifies human attention into four real-time states using embedded hardware, computer vision, acoustic sensing, and motion detection — running at ~10 FPS on the STM32N657X0 Neural-ART NPU.**

![Focus Tracker Cover](cover.jpg)

</div>

---

## 📋 Table of Contents

- [Overview](#overview)
- [Attention States](#attention-states)
- [Hardware](#hardware)
- [System Architecture](#system-architecture)
- [DSP Pipeline](#dsp-pipeline)
- [Installation](#installation)
- [Usage](#usage)
- [Calibration](#calibration)
- [Results](#results)
- [Industry Applications](#industry-applications)
- [Future Work](#future-work)
- [License](#license)

---

## 🔍 Overview

Focus Tracker is a real-time embedded AI system that monitors human operator attention by fusing three independent sensor streams:

- 👁 **Vision** — OpenCV face and eye detection via webcam
- 🔊 **Acoustic** — Ambient noise level via MEMS microphone
- 📳 **Motion** — Psychomotor activity via 3-axis accelerometer

All three signals are processed through a **digital signal processing pipeline** and fused by a **rule-based classification engine** running on a Python host, with video frames streamed to the **STM32N657X0 Nucleo** via UART at **1.8 Mbaud**.

This project was developed as part of the **Advanced Embedded Systems** coursework at the **University of Tulsa** (April 2026).

---

## 🧠 Attention States

| State | Condition | Colour |
|---|---|---|
| ✅ **FOCUSED** | Face visible + eyes on screen + quiet + still | 🟢 Cyan |
| ⚠️ **DISTRACTED** | Eyes looking away OR moderate noise/motion | 🟠 Orange |
| ❌ **OVERLOADED** | Loud noise (≥1770 RMS) OR active motion (≥1.0g) | 🔴 Red |
| 🟡 **AWAY** | No face detected | 🟡 Yellow |

---

## 🔧 Hardware

| Component | Role | Interface |
|---|---|---|
| **NUCLEO-N657X0** (STM32N6, Cortex-M55 @ 800MHz) | Video streaming + Neural-ART NPU | UART @ 1.8Mbaud |
| **STEVAL-MKBOXPRO SensorTile.box PRO** | Microphone + Accelerometer | USB (WinUSB) |
| **USB Webcam** (640×480) | Face + eye detection | USB |
| **Host Laptop** | Python fusion engine + GUI | — |

### Hardware Setup

```
Webcam (USB)
    │
    ▼
Python Host ────── UART 1.8Mbaud ──────► Nucleo N657X0
    │                                      (video stream)
    │
    └────── USB ──────► SensorTile.box PRO
                          (mic + acc data)
```

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    SENSING LAYER                        │
│  Webcam 640×480  │  MP23DB01HP Mic  │  LSM6DSV16X Acc  │
└────────┬─────────┴────────┬──────────┴────────┬─────────┘
         │                  │                   │
         ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                   PROCESSING LAYER                      │
│  OpenCV Haar     │  RMS Calculation │  Motion Magnitude │
│  Cascade         │  Sliding Window  │  Sensitivity Scale│
│  Face + Eye      │  Calibrated Thr. │  Calibrated Thr.  │
└────────┬─────────┴────────┬──────────┴────────┬─────────┘
         │                  │                   │
         ▼                  ▼                   ▼
┌─────────────────────────────────────────────────────────┐
│                  CLASSIFICATION LAYER                   │
│         Multi-Modal Sensor Fusion Engine                │
│   face_detected + noise_level + motion_level            │
│    → FOCUSED / DISTRACTED / OVERLOADED / AWAY           │
└─────────────────────────────────────────────────────────┘
```

---

## 📡 UART Streaming Protocol

Each video frame follows a 3-step handshake protocol:

```
Python                    Nucleo N657X0
  │                            │
  │──── SYNC (0x55) ──────────►│  "Are you ready?"
  │                            │
  │◄─── ACK  (0xAA) ───────────│  "Yes, ready!"
  │                            │
  │──── 16,384 bytes ─────────►│  128×128 grayscale payload
  │    (512 byte chunks × 32)  │
  │                            │
  │◄─── DONE:N ────────────────│  Frame confirmed
```

**Specifications:**
- Baud rate: `1,843,200`
- Frame size: `128 × 128 × 1 = 16,384 bytes`
- Chunks: `32 × 512 bytes`
- Bandwidth utilisation: `~71%`
- Throughput: `~163,840 bytes/sec`

---

## 🔬 DSP Pipeline

### 1. RMS Calculation (Microphone)
```python
rms = np.sqrt(np.mean(audio.astype(float) ** 2))
```
Converts raw int16 audio samples into a single loudness value.

### 2. Sliding Window
```python
mic_file.seek(-2000, 2)   # last 2000 bytes — microphone
acc_file.seek(-600,  2)   # last 600 bytes  — accelerometer
```
Only the most recent data influences classification.

### 3. Sensitivity Scaling (Accelerometer)
```python
motion = np.mean(np.abs(data.astype(float))) * 0.000488  # g/LSB
```
Converts raw LSM6DSV16X int16 counts to gravitational units (g).

### 4. Calibrated Thresholding
```
threshold = (phase_avg₁ + phase_avg₂) / 2
```
Environment-specific boundaries derived from 45-second per-class calibration.

---

## 💻 Installation

### Prerequisites

```bash
Python 3.10+
STM32CubeIDE (for firmware)
Zadig (WinUSB driver for SensorTile)
```

### Clone Repository

```bash
git clone https://github.com/yourusername/focus-tracker.git
cd focus-tracker
```

### Install Python Dependencies

```bash
pip install opencv-python numpy pyserial pillow
pip install pandas matplotlib scikit-learn

# SensorTile SDK
git clone --recursive https://github.com/STMicroelectronics/stdatalog-pysdk.git
cd stdatalog-pysdk
.\STDATALOG-PYSDK_install_noGUI.bat   # Windows
```

### Flash Nucleo Firmware

1. Open `firmware/Focus_Tracker_Nucleo.ioc` in STM32CubeIDE
2. Build project → `Ctrl+B`
3. Flash → `F11`

---

## 🚀 Usage

### Step 1 — Connect Hardware

```
1. Connect SensorTile.box PRO via USB-C
2. Install WinUSB driver via Zadig
3. Connect Nucleo N657X0 via ST-LINK USB
4. Connect webcam via USB
```

### Step 2 — Run GUI

```bash
python Focus_Tracker_Final_Project.py
```

### Step 3 — Connect and Stream

```
1. Click CONNECT SENSORTILE → wait for ✅
2. Select Nucleo COM port → click CONNECT NUCLEO
3. Wait for AI:OK in serial log
4. Click START STREAM
```

### Step 4 — Watch Attention States

```
✅ Sit quietly facing screen     → FOCUSED
⚠️ Look away or talk normally   → DISTRACTED
❌ Make loud noise or move fast  → OVERLOADED
🟡 Walk away from camera        → AWAY
```

---

## 🎛️ Calibration

Run calibration to adapt thresholds to your specific environment:

```python
python calibrate.py
```

**Procedure:**

| Phase | Duration | Action | Example Result |
|---|---|---|---|
| Silent | 45 seconds | Stay completely quiet | avg = 1,162 RMS |
| Normal | 45 seconds | Talk at normal volume | avg = 1,674 RMS |
| Loud | 45 seconds | Clap, shout loudly | avg = 1,865 RMS |

**Derived thresholds:**
```
FOCUSED    < 1,418 RMS   (midpoint: silent ↔ normal)
DISTRACTED < 1,770 RMS   (midpoint: normal ↔ loud)
OVERLOADED ≥ 1,770 RMS
```

---

## 📊 Results

| Metric | Value |
|---|---|
| Streaming Rate | ~10 FPS |
| UART Baud Rate | 1,843,200 bps |
| Frame Size | 16,384 bytes |
| Bandwidth Utilisation | ~71% |
| Calibration Time | 135 seconds (3 × 45s) |
| Sensor Update Rate | 200ms (5 Hz) |
| Classification Latency | ~100ms |

---

## 🏭 Industry Applications

| Sector | Application | Driver |
|---|---|---|
| 🚛 Transportation | Driver drowsiness detection | EU GSR 2019/2144 (DMS mandatory 2024) |
| ✈️ Aviation | Pilot attention monitoring | EASA fatigue regulations |
| 🏭 Industrial | Heavy machinery operator safety | HSE workplace legislation |
| 📚 Education | Student engagement in e-learning | 80% online dropout rate |
| 🏥 Healthcare | ADHD and cognitive assessment | NHS clinical monitoring |
| 👓 Smart Glasses | Wearable attention tracking | Emerging IoT market |

---

## 🔮 Future Work

- [ ] **NanoEdge AI classifier** — deploy trained model on Nucleo Neural-ART NPU
- [ ] **Data collection pipeline** — labeled training data from real sessions
- [ ] **Heart rate integration** — MAX30102 sensor for HRV physiological signal
- [ ] **Smart glasses prototype** — miniaturised wearable with custom PCB
- [ ] **Gaze tracking** — replace head pose proxy with precise eye tracking
- [ ] **BLE connectivity** — wireless data streaming to mobile app
- [ ] **Longitudinal monitoring** — attention pattern analysis over time

---

## 📁 Project Structure

```
focus-tracker/
├── firmware/
│   ├── Focus_Tracker_Nucleo.ioc      # STM32CubeIDE project
│   ├── Core/
│   │   └── Src/main.c               # Nucleo C firmware
│   └── X-CUBE-AI/
│       └── App/app_x-cube-ai.c      # BlazeFace AI init
├── python/
│   ├── Focus_Tracker_Final_Project.py  # Main GUI application
│   ├── calibrate.py                    # Calibration script
│   └── mock_sensor.py                  # Development mock
├── data/
│   └── attention_data.csv              # Training data
├── docs/
│   ├── Focus_Tracker_Report.docx
│   ├── Focus_Tracker_Flowchart.docx
│   └── Focus_Tracker_Presentation.pptx
├── cover.jpg
├── README.md
└── requirements.txt
```

---

## 📦 Requirements

```
opencv-python>=4.8.0
numpy>=1.24.0
pyserial>=3.5
Pillow>=10.0.0
pandas>=2.0.0
scikit-learn>=1.3.0
matplotlib>=3.7.0
stdatalog-core>=1.3.0
stdatalog-pnpl>=1.3.0
keyboard>=0.13.5
```

---

## 🙏 Acknowledgements

- **STMicroelectronics** — STM32N657X0 Nucleo board and STEVAL-MKBOXPRO SensorTile.box PRO
- **Google** — BlazeFace lightweight face detection model
- **OpenCV** — Haar Cascade face and eye detection
- **University of Tulsa** — Advanced Embedded Systems coursework

---

## 📄 License

```
Copyright (c) 2026 Jula Sherene 
All Rights Reserved.

This repository and all its contents — including source code, documentation,
schematics, reports, and associated files — are made publicly visible for the
sole purpose of portfolio review by prospective employers, collaborators, and
academic reviewers.

No license is granted, express or implied, for any of the following:
  - Reuse of the code or any portion thereof in other projects
  - Modification or derivative works
  - Redistribution in any form
  - Commercial use of any kind
  - Use for training, fine-tuning, or evaluating machine learning models

All rights are reserved by the copyright holder. For any use beyond review,
written permission must be obtained from the author.


```

---

<div align="center">

**Built at University of Tulsa · April 2026**

*Advanced Embedded Systems · Intro to Research*

</div>
