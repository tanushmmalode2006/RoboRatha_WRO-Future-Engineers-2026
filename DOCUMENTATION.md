# Engineering Documentation — Mobility, Power, Sense & Obstacle Management

This document provides the required discussion, information, and motivation
for our vehicle's **mobility**, **power**, **sense**, and **obstacle
management** systems, as required by the WRO Future Engineers 2026 GitHub
documentation rules (Section 7 of the General Rules).

---

## 1. Mobility

### 1.1 Chassis and Drive Configuration

Our vehicle is a 4-wheeled vehicle with:

- **One driving axle** (rear), powered by a single 12V DC motor (10,000 RPM
  nominal, geared down through the drivetrain to usable wheel speed)
- **One steering axle** (front), actuated by an MG90S micro servo through an
  Ackermann-style steering linkage

This satisfies WRO rule 11.3, which requires the vehicle to be front-wheel,
rear-wheel, or four-wheel drive with a genuine steering actuator — explicitly
prohibiting differential-drive (tank-style) bases.

**Why rear-wheel drive with front steering?**
We chose this configuration because it most closely mirrors real automotive
Ackermann geometry, which is the design intent of the Future Engineers
category (kinematics different from differential drive). It also keeps the
drivetrain simple: a single motor drives the rear axle through a gearbox,
avoiding the complexity of a front-wheel-drive + steering combination.

### 1.2 Gearing and Speed Control

The 12V DC motor is rated at ~10,000 RPM unloaded. Direct-drive at this speed
would be uncontrollable on the track, so the motor is connected to the rear
axle through a **reduction gearbox**, bringing wheel speed down to a
controllable range. Fine speed control is then handled in software via PWM
duty cycle sent to the MDD3A motor driver, with actual wheel speed verified
using encoder feedback (see Section 3.3).

### 1.3 Steering Mechanism

The MG90S servo drives a steering linkage connected to both front wheels,
providing genuine Ackermann-style turning geometry rather than a single-wheel
pivot. Steering angle is bounded in software to prevent mechanical binding at
full lock, and centre position is calibrated during setup so that "0°
commanded angle" corresponds to true straight-ahead travel.

### 1.4 Testing & Iteration

During bench testing we found that:
- Excess play in the steering linkage caused inconsistent turn radii — this
  was addressed by tightening the linkage and adding a return-spring effect
  in the mounting.
- At full driving speed, the vehicle could not complete tight corners without
  understeering — we compensate for this by reducing commanded motor speed
  automatically whenever a corner (large steering angle) is commanded.

---

## 2. Power

### 2.1 Dual-Battery Architecture

We deliberately separate **logic power** from **motor power** using two
independent 12V battery packs:

- **Battery 1 (Logic):** 12V → Buck converter → 5V rail powering the Main
  ESP32, ESP32-S3-CAM, and MG90S servo.
- **Battery 2 (Motor):** 12V directly into the MDD3A motor driver, which
  drives the 12V DC motor.

**Why two batteries instead of one shared supply?**
DC motors draw large transient currents on start/stop and direction change,
which causes voltage sag ("brownout") on a shared rail. This can reset or
destabilize a microcontroller mid-run. By isolating motor current draw onto
its own battery and regulator, the logic-side 5V rail stays stable regardless
of motor load — this is critical since a microcontroller reset mid-lap would
end the round with zero score.

### 2.2 Voltage Budget

| Component | Voltage | Approx. Current Draw |
|---|---|---|
| ESP32 (Main) | 5V | ~200–300 mA (Wi-Fi disabled to save power) |
| ESP32-S3-CAM | 5V | ~300–500 mA (camera + PSRAM active) |
| MG90S Servo | 5V | ~200 mA idle, up to 700 mA under load |
| Ultrasonic sensors (×3) | 5V | ~15 mA each |
| IMU | 3.3V/5V | <10 mA |
| DC Motor (peak) | 12V | Several amps during acceleration |

The buck converter is sized with margin above the combined peak logic-side
draw (~1.5 A) to avoid brownout during simultaneous camera + servo activity.

### 2.3 Switching and Start Procedure

Per WRO rule 9.10–9.11, the vehicle uses **one switch** to power the entire
system on, followed by **one push button** to begin the run program. This is
implemented by having the main power switch gate both battery packs
simultaneously, and the push button is read as a digital input on the Main
ESP32 which only then enables motor output and begins the navigation state
machine.

---

## 3. Sense

### 3.1 Ultrasonic Sensors (Front, Left, Right)

Three ultrasonic sensors give the vehicle awareness of the track walls and
obstacles ahead:

- **Front sensor:** detects obstacles or the finish-section wall ahead
- **Left/Right sensors:** measure distance to side walls, used to keep the
  vehicle centered in the lane via a simple proportional correction (if left
  distance < right distance, steer slightly right, and vice versa)

### 3.2 IMU (Heading Correction)

The IMU provides heading angle to correct for drift caused by wheel slip or
uneven ground. We track a target heading (typically 0°, ±90°, or ±180°
depending on section direction) and apply steering correction proportional to
the deviation from that target. This is especially important on the
straightforward sections, where ultrasonic wall-following alone can drift if
one wall is temporarily out of range (e.g., missing wall segment).

### 3.3 Encoder (Speed & Distance)

The rotary encoder on the drive axle is used to:
- Maintain constant vehicle speed (closed-loop PWM adjustment)
- Estimate distance travelled within a section, which helps disambiguate
  corner sections from straight sections when combined with IMU heading
  changes
- Support lap counting logic alongside section-boundary detection

### 3.4 Camera (ESP32-S3-CAM)

The camera is dedicated to:
- **Pillar detection:** identifying red vs. green traffic signs by colour
  segmentation, and estimating their approximate position (left/right of
  centre) to determine which side to pass on
- **Parking lot detection:** identifying the magenta parking lot boundary
  markers during the final approach phase

Running vision on a **separate controller** from navigation was a deliberate
design decision: image processing (especially colour segmentation across
full camera frames) is computationally heavy and would otherwise introduce
latency into the time-critical wall-following and steering control loop
running on the Main ESP32.

### 3.5 Sensor Fusion Priority

When sensor outputs conflict, our control logic applies the following
priority order (highest to lowest):

1. **Obstacle/wall safety** (ultrasonic) — always takes precedence to avoid
   collisions or disqualification for touching walls
2. **Wall following** (ultrasonic left/right)
3. **IMU heading correction**
4. **Vision-based pillar/parking decisions**

This ordering ensures the vehicle never sacrifices basic safety (not hitting
walls) in favor of a vision-based maneuver.

---

## 4. Obstacle Management

### 4.1 Traffic Pillar Rule Compliance

Per WRO rules 9.19–9.20, the vehicle must:
- Pass **red pillars on the right**
- Pass **green pillars on the left**
- Avoid moving pillars outside their designated circle (rule 9.20 / Appendix
  A Section 1)

Our approach:
1. The ESP32-S3-CAM continuously scans for red/green blobs in the forward
   camera view.
2. Once a pillar is detected and its approximate lateral position is
   estimated, a command (`TURN_LEFT` / `TURN_RIGHT`) is sent to the Main
   ESP32 over UART.
3. The Main ESP32 blends this vision command with the existing wall-following
   correction, biasing the lane position toward the correct side **before**
   reaching the pillar, rather than reacting at the last moment — this
   reduces the risk of clipping the pillar.
4. Once the pillar is passed (confirmed via front ultrasonic distance
   increasing again, or camera losing the pillar from view), the vehicle
   returns to centered wall-following.

### 4.2 Parking Sequence

After completing three laps, the vehicle executes the following parking
sequence:

1. Detect the parking lot via camera (magenta boundary markers)
2. Reduce speed for a controlled approach
3. Align the vehicle parallel to the outer wall using IMU heading
4. Perform a reverse-and-straighten maneuver if needed to fit within the
   parking lot boundaries (parking lot length = 1.5 × robot length, per rule
   and Figure 4 of the General Rules)
5. Stop once the vehicle's projection is fully within the parking lot and
   parallel to the wall (within the ±2 cm tolerance defined in Appendix A,
   Section 6)

### 4.3 Failure Handling

- If the front ultrasonic sensor detects an unexpected obstacle mid-corridor
  (e.g., a pillar closer than expected), the vehicle reduces speed
  proportionally to distance rather than stopping abruptly, to avoid
  triggering an unnecessary "vehicle stopped" state.
- If vision loses track of a pillar mid-pass (e.g., due to lighting), the
  Main ESP32 falls back to ultrasonic-only wall following until the pillar
  is confirmed clear, since safety (rule 4.1's priority order) takes
  precedence over vision-based accuracy.

---

## 5. Summary of Key Engineering Decisions

| Decision | Alternative Considered | Why We Chose Our Approach |
|---|---|---|
| Dual ESP32 (Main + Cam) | Single ESP32 running both nav + vision | Avoids vision-processing latency affecting real-time steering control |
| Dual battery power | Single shared battery | Prevents motor current draw from causing microcontroller brownouts |
| Ackermann steering (MG90S) | Differential drive | Differential drive is disallowed under rule 11.3; Ackermann better matches real automotive kinematics |
| Ultrasonic wall-following as top priority | Vision-first navigation | Ultrasonic is faster and more reliable for immediate collision avoidance than frame-by-frame vision |
| Pre-emptive lane bias before pillar pass | React only when pillar directly ahead | Reduces risk of clipping pillars and getting disqualified for moving them outside the seat circle |

This documentation will be updated as we continue testing and tuning ahead of
the competition, with version history tracked via GitHub commits.
