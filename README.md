# WRO Future Engineers 2026 — Self-Driving Car

## Team Vehicle Overview

This repository contains the engineering documentation, source code, and design
files for our autonomous vehicle built for the **World Robot Olympiad (WRO)
Future Engineers 2026 — Self-Driving Cars** category. Our vehicle is designed
to complete the **Open Challenge** and **Obstacle Challenge** by driving three
laps autonomously, avoiding/obeying red and green traffic pillars, and
performing parallel parking.

---

## 1. Hardware Architecture

Our vehicle uses a **dual-controller architecture**: one controller dedicated
to vision processing, and one dedicated to navigation, sensor fusion, and
motor control.

### 1.1 Main Controller — ESP32

The Main ESP32 is the "brain" of the robot. It is responsible for:

- Reading and processing all onboard sensors (ultrasonic, IMU, encoder)
- Executing the navigation and wall-following logic
- Executing steering corrections
- Driving the DC motor via the motor driver
- Counting laps and tracking the current phase of the mission
- Receiving high-level vision commands from the ESP32-S3-CAM and translating
  them into steering/speed actions
- Managing the push-button start sequence required by WRO rules (9.10–9.11)

**Connected peripherals:**
| Component | Interface | Purpose |
|---|---|---|
| Ultrasonic sensor (Front) | GPIO (Trig/Echo) | Obstacle/wall detection ahead |
| Ultrasonic sensor (Left) | GPIO (Trig/Echo) | Left wall distance |
| Ultrasonic sensor (Right) | GPIO (Trig/Echo) | Right wall distance |
| IMU | I2C | Heading angle, drift correction |
| Encoder | GPIO (interrupt) | Speed and distance measurement |
| MG90S Servo | PWM | Ackermann-style front-wheel steering |
| MDD3A Motor Driver | PWM + Direction pins | Driving motor speed/direction |
| Push Button | GPIO (digital in) | Start trigger (per WRO rule 9.11) |

### 1.2 Vision Controller — ESP32-S3-CAM

The ESP32-S3-CAM is dedicated entirely to computer vision so that image
processing does not block real-time navigation and motor control on the main
controller. It is responsible for:

- Capturing and processing camera frames
- Detecting red and green traffic pillars using colour/shape recognition
- Detecting the parking lot boundary markers
- Sending high-level directional commands to the Main ESP32

**Communication protocol:** UART serial link between ESP32-S3-CAM (TX/RX) and
Main ESP32.

Example command exchange:

```cpp
// ESP32-S3-CAM -> Main ESP32
"TURN_LEFT"    // detected green pillar, must pass on the left
"TURN_RIGHT"   // detected red pillar, must pass on the right
"PARK_DETECTED"// parking lot located, initiate parking sequence
```

### 1.3 Full Hardware Connection Flow

```text
                +-------------------+
                |   ESP32-S3-CAM    |
                | Camera Processing |
                +---------+---------+
                          |
                     UART Serial
                          |
+--------------------------------------------------+
|                    ESP32 MAIN                     |
|----------------------------------------------------|
| Ultrasonic Front                                  |
| Ultrasonic Left                                   |
| Ultrasonic Right                                  |
| IMU Sensor                                        |
| Encoder Input                                     |
| Push Button                                       |
| Servo Steering Control (MG90S)                    |
| Motor Speed Control (MDD3A)                        |
+------------+-------------------+------------------+
             |                   |
      +------v-----+      +------v------+
      |  MG90S     |      |   MDD3A     |
      |  Steering  |      | MotorDriver |
      +------------+      +------+------+
                                  |
                           +------v------+
                           | 12V DC Motor |
                           |  (10k RPM)   |
                           +-------------+
```

### 1.4 Power Distribution

We use **two independent 12V batteries** to separate logic power from
high-current motor power, preventing motor noise/voltage sag from affecting
the microcontrollers.

**Battery 1 — Logic Supply**
```text
12V Battery 1 → Buck Converter (12V → 5V) → ESP32 + ESP32-S3-CAM + MG90S Servo
```

**Battery 2 — Motor Supply**
```text
12V Battery 2 → MDD3A Motor Driver → 12V DC Motor
```

| Component | Supply Voltage |
|---|---|
| ESP32 | 5V (regulated) |
| ESP32-S3-CAM | 5V (regulated) |
| MG90S Servo | 5V |
| Ultrasonic Sensors | 5V |
| IMU | 3.3V / 5V |
| MDD3A Motor Driver | 12V |
| 12V DC Motor | 12V |

### 1.5 Steering & Drive System

Per WRO vehicle regulations (rules 11.3–11.5), our vehicle uses:

- **Ackermann-style front-wheel steering** via the MG90S servo
- **A single driving axle** driven by one 12V DC motor connected through a
  gearbox to the rear axle
- **No differential wheeled base** and no electronic differential — the
  vehicle is a proper 4-wheeled, front-steered vehicle, compliant with rule
  11.3

---

## 2. Repository Structure

```text
PROJECT/
│
├── main/
│   ├── main.ino
│   ├── config.h
│   └── pins.h
│
├── sensors/
│   ├── ultrasonic.cpp
│   ├── imu.cpp
│   ├── encoder.cpp
│   └── camera.cpp
│
├── control/
│   ├── steering.cpp
│   ├── motor.cpp
│   └── pid.cpp
│
├── navigation/
│   ├── wall_follow.cpp
│   ├── obstacle.cpp
│   ├── lap_counter.cpp
│   └── parking.cpp
│
├── communication/
│   └── serial_comm.cpp
│
├── vision/                    (runs on ESP32-S3-CAM)
│   ├── camera_init.cpp
│   ├── pillar_detection.cpp
│   └── parking_detection.cpp
│
├── test/
│   ├── test_motor.ino
│   ├── test_servo.ino
│   ├── test_ultrasonic.ino
│   └── test_camera.ino
│
├── docs/
│   └── DOCUMENTATION.md       (engineering discussion — mobility, power, sense, obstacle management)
│
└── README.md
```

---

## 3. How to Build / Compile / Upload the Code

### 3.1 Requirements

- Arduino IDE (or PlatformIO)
- ESP32 board support package installed via Boards Manager
- ESP32-S3-CAM board support package
- Libraries: `ESP32Servo`, `Wire` (I2C for IMU), relevant IMU library,
  `esp32-camera`

### 3.2 Uploading to the Main ESP32

1. Open `main/main.ino` in Arduino IDE
2. Select **Board: ESP32 Dev Module**
3. Select the correct COM port
4. Set upload speed to 921600 (or lower if upload fails)
5. Click Upload
6. Open Serial Monitor at 115200 baud to verify sensor readings

### 3.3 Uploading to the ESP32-S3-CAM

1. Open the `vision/` sketch folder
2. Select **Board: ESP32S3 Dev Module**
3. Enable PSRAM in board settings (required for camera frame buffers)
4. Hold the BOOT button while connecting if the board doesn't enter
   flash mode automatically
5. Upload and verify camera streaming/detection via Serial Monitor

### 3.4 Wiring Verification Before First Run

- Confirm both ESP32 boards share a common ground
- Confirm UART TX (CAM) → RX (Main) and RX (CAM) → TX (Main) are crossed
  correctly
- Confirm the push button is wired per rule 9.10–9.11 (single switch to power
  on, single button to start)

---

## 4. Compliance Notes

- No Wi-Fi/Bluetooth is used during competition runs, as required by rule
  11.10 (any wireless hardware present on the ESP32/ESP32-S3-CAM is disabled
  in firmware and can be inspected by judges)
- Vehicle dimensions and weight are verified to stay within 300×200×300 mm
  and 1.5 kg (rules 11.1–11.2)
- Steering and driving are mechanically separated per rule 11.3

---

## 5. Further Documentation

See [`docs/DOCUMENTATION.md`](docs/DOCUMENTATION.md) for a full engineering
discussion of our mobility, power, sensing, and obstacle-management design
decisions, including testing and iteration notes.
