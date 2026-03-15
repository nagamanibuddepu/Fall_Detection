# 🛡️ FallGuard — IoT-Based Elderly Fall Detection System

> A real-time fall detection system using NodeMCU (ESP8266) and MPU6050 sensor that automatically detects falls and sends emergency alerts to caregivers via IFTTT Webhooks.

---


## 🌐 Live Demo
👉 [Open FallGuard App](https://nagamanibuddepu.github.io/Fall_Detection/)

## 📸 Live Demo Interface
The web interface provides a one-tap emergency button for elderly users, with large text, high contrast mode, and instant caregiver notification.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware Components](#hardware-components)
- [System Architecture](#system-architecture)
- [Fall Detection Algorithm](#fall-detection-algorithm)
- [Acceleration & Trigger Thresholds](#acceleration--trigger-thresholds)
- [Web Interface](#web-interface)
- [Tech Stack](#tech-stack)
- [Circuit Connections](#circuit-connections)
- [Setup & Installation](#setup--installation)
- [IFTTT Webhook Configuration](#ifttt-webhook-configuration)
- [File Structure](#file-structure)

---

## 🔍 Overview

FallGuard is an IoT prototype that continuously monitors the motion of elderly individuals wearing a NodeMCU + MPU6050 device. When a fall pattern is detected through a multi-stage algorithm, it automatically fires an emergency HTTP webhook via IFTTT, alerting caregivers in real time.

A companion web interface allows manual alert triggering — useful if the person is conscious but unable to get up.

---

## 🔧 Hardware Components

| Component | Role |
|---|---|
| **NodeMCU ESP8266** | Microcontroller + Wi-Fi module |
| **MPU6050** | 3-axis Accelerometer + 3-axis Gyroscope (I2C) |
| Power Source | USB / 3.7V LiPo Battery |
| Jumper Wires | I2C connections (SDA, SCL) |

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        WEARABLE DEVICE                          │
│                                                                 │
│   ┌──────────────┐    I2C     ┌──────────────────────────────┐  │
│   │   MPU6050    │ ─────────► │       NodeMCU ESP8266        │  │
│   │              │            │                              │  │
│   │  Accel (XYZ) │            │  Fall Detection Algorithm    │  │
│   │  Gyro  (XYZ) │            │  ┌─────────────────────┐    │  │
│   └──────────────┘            │  │ Trigger 1: Free-fall │    │  │
│                               │  │ Trigger 2: Impact    │    │  │
│                               │  │ Trigger 3: Rotation  │    │  │
│                               │  │ Confirm:  Stillness  │    │  │
│                               │  └─────────────────────┘    │  │
│                               └──────────────┬───────────────┘  │
└──────────────────────────────────────────────┼──────────────────┘
                                               │ Wi-Fi (HTTP GET)
                                               ▼
                                   ┌───────────────────┐
                                   │   IFTTT Webhooks   │
                                   │  maker.ifttt.com   │
                                   └─────────┬─────────┘
                                             │
                               ┌─────────────┼─────────────┐
                               ▼             ▼             ▼
                          📧 Email      📱 SMS        🔔 Push
                         Caregiver   Caregiver      Notification
```

---

## 🧠 Fall Detection Algorithm

The algorithm uses **4 sequential triggers** to distinguish a real fall from normal activities like sitting down quickly or bending over.

```
NORMAL MOTION
     │
     ▼
┌─────────────────────────────────────┐
│  Read Accelerometer + Gyroscope     │  ← every 100ms
│  Compute Amplitude Vector (AM)      │
│  AM = √(ax² + ay² + az²)           │
└────────────────┬────────────────────┘
                 │
                 ▼
        ┌────────────────┐
        │  AM ≤ 0.4g ?   │  ← Sudden drop (free-fall phase)
        │  (Amp ≤ 2)     │
        └───────┬────────┘
           YES  │   NO → reset, keep monitoring
                ▼
        ✅ TRIGGER 1 SET
        (free-fall detected)
                │
                ▼ within 0.5s
        ┌────────────────┐
        │  AM ≥ 3g ?     │  ← High impact (hitting ground)
        │  (Amp ≥ 12)    │
        └───────┬────────┘
           YES  │   NO → TRIGGER 1 DEACTIVATED
                ▼
        ✅ TRIGGER 2 SET
        (impact detected)
                │
                ▼ within 0.5s
        ┌──────────────────────┐
        │  Angle change        │  ← Body rotation 80–100°
        │  30 ≤ ΔAngle ≤ 400  │
        └──────────┬───────────┘
              YES  │   NO → TRIGGER 2 DEACTIVATED
                   ▼
           ✅ TRIGGER 3 SET
           (orientation changed)
                   │
                   ▼ after 1s
        ┌──────────────────────┐
        │  Still lying down?   │  ← No movement (0–10°/s)
        │  0 ≤ ΔAngle ≤ 10    │
        └──────────┬───────────┘
              YES  │   NO → user recovered, reset all
                   ▼
           🚨 FALL CONFIRMED
           → send_event("fall_detect")
           → HTTP GET → IFTTT Webhook → Caregiver Alert
```

---

## 📊 Acceleration & Trigger Thresholds

```
Acceleration Magnitude (g-force) over time during a fall event:

  4.0g  │                        ╔══╗
        │                        ║  ║   ← IMPACT (Trigger 2: AM ≥ 3g)
  3.0g  │                        ║  ║
        │                        ║  ║
  2.0g  │                        ║  ╚══════╗
        │                        ║         ╚═══ normal post-fall
  1.0g  │════════════════════╗   ║              stillness ~1g
        │  normal walking    ║   ║
  0.4g  │· · · · · · · · · · ╚═══╝ ← FREE-FALL ZONE (Trigger 1: AM ≤ 0.4g)
        │
  0.0g  └───────────────────────────────────────────────────────► time
             normal          free   impact      lying
             activity        fall               still
                             ↑
                         Trigger 1
                         activated

Gyroscope (angle change) during a fall event:

 400°/s │              ╔══╗
        │              ║  ║   ← body rotation (Trigger 3: 30–400°/s)
 200°/s │              ║  ╚═╗
        │              ║    ╚═══╗
  30°/s │· · · · · · · ║        ╚═══════════════════════
        │              ║                  ↑
   0°/s └──────────────────────────────────────────────► time
                    rotating          lying still
                                  (Confirm: 0–10°/s)
```

---

## 🌐 Web Interface

The companion web page (`fall_detection.html`) provides:

- **One-tap emergency button** — large, accessible red SEND ALERT button
- **Confirmation dialog** — prevents accidental taps
- **Success feedback** — toast message confirms alert was sent
- **Accessibility toggles** — Large Text mode + High Contrast mode
- **System info panel** — sensor, alert method, connectivity status
- **How-it-works section** — explains the 4-step algorithm to caregivers

The manual alert button triggers the same IFTTT webhook as the hardware sensor, ensuring caregivers receive alerts from both sources.

---

## 💻 Tech Stack

```
┌─────────────────────────────────────────────────────┐
│                    FIRMWARE                          │
│                                                     │
│  Language    :  C++ (Arduino framework)             │
│  IDE         :  Arduino IDE                         │
│  Libraries   :  Wire.h (I2C)                        │
│                 ESP8266WiFi.h (Wi-Fi)               │
│  Protocol    :  HTTP GET over TCP (port 80)         │
│  Board       :  NodeMCU 1.0 (ESP8266)               │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│                  WEB INTERFACE                       │
│                                                     │
│  Language    :  HTML5, CSS3, Vanilla JavaScript     │
│  Fonts       :  Google Fonts (Nunito, Lato)         │
│  Alert API   :  IFTTT Webhooks (HTTP GET)           │
│  Hosting     :  Static file (no server needed)      │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│               ALERT INFRASTRUCTURE                   │
│                                                     │
│  Service     :  IFTTT (If This Then That)           │
│  Trigger     :  Webhooks (maker.ifttt.com)          │
│  Actions     :  Email / SMS / Push Notification     │
│  Event Name  :  fall_detected / fall_detect         │
└─────────────────────────────────────────────────────┘
```

---

## 🔌 Circuit Connections

```
MPU6050        NodeMCU ESP8266
──────────     ───────────────
VCC       →    3.3V
GND       →    GND
SDA       →    D2 (GPIO 4)
SCL       →    D1 (GPIO 5)
```

> ⚠️ Use 3.3V only — the MPU6050 is NOT 5V tolerant on the NodeMCU.

---

## ⚙️ Setup & Installation

### 1. Arduino IDE Setup

```
1. Install Arduino IDE: https://www.arduino.cc/en/software
2. Add ESP8266 board support:
   File → Preferences → Additional Boards Manager URL:
   http://arduino.esp8266.com/stable/package_esp8266com_index.json
3. Tools → Board → Boards Manager → search "esp8266" → Install
4. Select Board: NodeMCU 1.0 (ESP-12E Module)
```

### 2. Install Required Libraries

```
Tools → Manage Libraries → Search and install:
  ✅ Wire        (built-in)
  ✅ ESP8266WiFi (included with ESP8266 board package)
```

### 3. Configure the Code

Open `IoT based Fall Detection using Nod.txt`, update these lines:

```cpp
const char *ssid = "YOUR_WIFI_NAME";
const char *pass = "YOUR_WIFI_PASSWORD";
const char *privateKey = "YOUR_IFTTT_PRIVATE_KEY";
```

### 4. Upload

```
1. Connect NodeMCU via USB
2. Select correct COM port: Tools → Port
3. Click Upload (→)
4. Open Serial Monitor (115200 baud) to see live debug output
```

---

## 🔔 IFTTT Webhook Configuration

```
1. Sign up at https://ifttt.com
2. Create a new Applet:
   IF   → Webhooks → "Receive a web request"
          Event Name: fall_detected
   THEN → Choose action (Email / SMS / Notification)

3. Get your Webhook key:
   https://ifttt.com/maker_webhooks → Documentation
   Copy your key and paste it into privateKey in the .ino file

4. Test your webhook manually:
   https://maker.ifttt.com/trigger/fall_detected/with/key/YOUR_KEY
```

---

## 📁 File Structure

```
Fall_Detection/
│
├── IoT based Fall Detection using Nod.txt   # C++ firmware (Arduino)
│     ├── MPU6050 sensor reading (mpu_read)
│     ├── Fall detection algorithm (4 triggers)
│     └── IFTTT alert sender (send_event)
│
├── fall_detection.html                      # Web interface
│     ├── Manual alert button (IFTTT webhook)
│     ├── Accessibility toggles (large text, contrast)
│     └── System info + algorithm explainer
│
└── README.md                                # This file
```

---

## ⚠️ Limitations & Future Improvements

| Limitation | Potential Fix |
|---|---|
| IFTTT free tier has delays | Use direct SMS API (Twilio) |
| No false-fall cancellation window | Add 30s cancel button after alert |
| Single caregiver alert | Multi-contact webhook chaining |
| No battery level monitoring | Add ADC-based battery indicator |
| Device must be worn consistently | Reminder buzzer at set intervals |

---

## 👤 Author

**Nagamani Buddepu**
[GitHub Profile](https://github.com/nagamanibuddepu)

---

## 📄 License

This project is open source and available for educational use.
