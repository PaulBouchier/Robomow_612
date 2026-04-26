
## Project Goal
Convert a **RoboMow 612P** lawn mower base into a fully autonomous ROS 2 robot that:
1. **Phase 1 (current focus):** Detects grass via camera (HSV segmentation) and drives only on grass, avoiding non-grass areas.
2. **Phase 2 (future):** Full autonomous navigation with SLAM-built map and Nav2 waypoint following and updated to use VLA (model TBD) to improve separating grass covered areas from non-grassy areas to avoid.

---

## Hardware

### Robot Base
| Property | Value |
|---|---|
| Base model | RoboMow 612P |
| Drive type | Differential drive |
| Drive wheels | 2 × rear, diameter **216 mm**, separation **534 mm** (centre-to-centre) |
| Caster wheel | 1 × front, diameter **102 mm**, offset **483 mm** forward of drive axle |

### Electronics
| Component | Role | Notes |
|---|---|---|
| Raspberry Pi 5 (4 GB) | Robot Computer — runs all ROS 2 nodes | Active cooler required |
| Raspberry Pi Pico 2 (RP2350) | Microcontroller — micro-ROS firmware (base controller) | Connected to Pi via USB serial `/dev/ttyACM0` @ 921600 baud |
| Raspberry Pi Camera Module 3 (wide) | Grass detection | CSI ribbon to Pi |
| PCA9685 breakout (I2C addr 0x40) | 16-ch PWM expander — Pico → H-Bridges | Shared I2C bus with IMU |
| 2 × BTS7960 43A H-Bridge | Left and right motor power | Motor power from RoboMow battery (24 V nominal) |
| MPU-6050 (GY-521) | IMU — fused with wheel odometry via EKF | I2C addr 0x68, shared bus with PCA9685 |
| 2 × AS5600 magnetic encoder | Wheel odometry (4096 counts/rev) | Mounted on drive wheel axles |
| RPLIDAR A1M8 | 2D LiDAR for SLAM and Nav2 costmap | USB to Pi |
| Buck converter 12V→5V 5A | Powers Raspberry Pi 5 | From RoboMow battery |
| Buck converter 12V→5V 1A | Powers Pico + PCA9685 logic | From RoboMow battery |
| Latching e-stop button | Cuts motor power | In series with motor driver VCC |

### Wiring — I2C bus (Pico GP4=SDA, GP5=SCL)
```
Pico GP4 (SDA0) ──── PCA9685 SDA ──── MPU-6050 SDA
Pico GP5 (SCL0) ──── PCA9685 SCL ──── MPU-6050 SCL
```

### PCA9685 channel map → BTS7960
| CH | Motor | Direction |
|---|---|---|
| 0 | Left (Motor 1) | RPWM (forward) |
| 1 | Left (Motor 1) | LPWM (reverse) |
| 2 | Right (Motor 2) | RPWM (forward) |
| 3 | Right (Motor 2) | LPWM (reverse) |
BTS7960 R_EN and L_EN tied HIGH (3.3 V) — always enabled.

### Pico encoder GPIO pins
| Signal | Pico pin |
|---|---|
| Motor 1 (left) encoder A | GP6 |
| Motor 1 (left) encoder B | GP7 |
| Motor 2 (right) encoder A | GP8 |
| Motor 2 (right) encoder B | GP9 |

---

## Software Stack
| Layer | Technology |
|---|---|
| OS (Pi) | Ubuntu 24.04 |
| ROS 2 distro | Jazzy |
| Base framework | **linorobot2** (clone + modify) |
| Hardware firmware | **linorobot2_hardware** (rolling branch, Pico 2 target) |
| Firmware build tool | PlatformIO |
| SLAM | SLAM Toolbox (online async) |
| Navigation | Nav2 (NavFn planner + DWB controller) |
| Odometry fusion | robot_localization EKF |
| Grass detection | Custom ROS 2 node — OpenCV HSV segmentation |

---

## Repository Strategy
- Fork **`linorobot2`** (ROS 2 software) and **`linorobot2_hardware`** (Pico firmware)
- Add new package `linorobot2_hardware_pi` alongside existing packages (for any Pi-side hardware utilities)
- **Do NOT replace** the micro-ROS / Pico architecture — it is kept as designed
- PCA9685 is integrated into the **Pico firmware** (not driven from Pi directly)

### Key firmware modification: `motor.h` patch
The linorobot2_hardware firmware normally calls `analogWrite()` on GPIO pins for motor PWM.
The patch replaces those calls with `pca9685Motor.setSpeed(channel_fwd, channel_rev, pwm)` from `PCA9685Motor.h`.
Variable names `MOTOR1_IN_A`, `MOTOR1_IN_B` etc. in `config.h` now hold **PCA9685 channel numbers** (0–3), not GPIO pin numbers. No other firmware changes are required.

---

## Config Files — these are based on linorobot2 but customized for the Robomow-612P

All files were generated with exact RoboMow 612P measurements already filled in.

| File | Destination in repo |
|---|---|
| `firmware/config.h` | `linorobot2_hardware/firmware/lib/config/config.h` |
| `firmware/PCA9685Motor.h` | `linorobot2_hardware/firmware/lib/motor/PCA9685Motor.h` |
| `urdf/robomow_properties.urdf.xacro` | `linorobot2_description/urdf/robots/robomow_properties.urdf.xacro` |
| `nav2/ekf.yaml` | `linorobot2_base/config/ekf.yaml` |
| `nav2/navigation.yaml` | `linorobot2_navigation/config/navigation.yaml` |
| `nav2/slam.yaml` | `linorobot2_navigation/config/slam.yaml` |

### Key values in configs
```
WHEEL_DIAMETER        = 0.216 m
LR_WHEELS_DISTANCE    = 0.534 m
CASTER_RADIUS         = 0.051 m
CASTER_X_OFFSET       = 0.483 m
COUNTS_PER_REV        = 4096    (AS5600 encoder — verify with calibration)
MOTOR_MAX_RPM         = 80      (estimate — MUST verify with calibration spin test)
MOTOR_OPERATING_VOLTAGE = 24 V
PWM_BITS              = 12      (PCA9685 12-bit)
PWM_FREQUENCY         = 1000 Hz
BAUDRATE              = 921600
IMU                   = USE_MPU6050_IMU
MOTOR DRIVER          = USE_GENERIC_2_IN_MOTOR_DRIVER (patched for PCA9685)
Nav2 max linear vel   = 0.25 m/s
Nav2 max angular vel  = 0.80 rad/s
Nav2 footprint        = [[0.325,-0.250],[0.325,0.250],[-0.325,0.250],[-0.325,-0.250]]
Nav2 inflation radius = 0.30 m
```

---

## Development Phases & Current Status

### ✅ Phase 0 — Architecture & Design (COMPLETE)
- [x] System architecture designed (Pi 5 + Pico 2 + PCA9685 + BTS7960)
- [x] Hardware shopping list finalised (~$260 total)
- [x] linorobot2 integration strategy decided
- [x] All config files generated with RoboMow measurements

### 🔲 Phase 1 — Hardware Bring-up (TODO)
- [ ] Wire Pico → PCA9685 over I2C; verify with `i2cdetect` equivalent
- [ ] Bench test PCA9685 channels with a simple Arduino sketch (spin motors manually)
- [ ] Connect BTS7960 boards; verify left/right motor direction
- [ ] Mount AS5600 encoders; verify pulse counting in PlatformIO serial monitor
- [ ] Run linorobot2_hardware **calibration firmware** — `spin` then `sample` commands
- [ ] Update `MOTOR_MAX_RPM` in `config.h` from calibration result
- [ ] Verify/flip `MOTOR1_INV`, `MOTOR2_INV`, `MOTOR1_ENCODER_INV`, `MOTOR2_ENCODER_INV`

### 🔲 Phase 2 — micro-ROS Base Controller (TODO)
- [ ] Flash linorobot2_hardware firmware to Pico 2
- [ ] Run `micro_ros_agent` on Pi: `ros2 run micro_ros_agent micro_ros_agent serial --dev /dev/ttyACM0 -b 921600`
- [ ] Confirm `/odom/unfiltered` and `/imu/data` topics appear
- [ ] Drive with teleop: `ros2 run teleop_twist_keyboard teleop_twist_keyboard`
- [ ] Verify robot drives straight; tune PID if oscillating (K_P=0.6, K_I=0.8, K_D=0.5)

### 🔲 Phase 3 — Full linorobot2 Stack (TODO)
- [ ] Launch `bringup.launch.py`; confirm EKF filtered odometry on `/odometry/filtered`
- [ ] Mount RPLIDAR A1M8; confirm `/scan` topic
- [ ] Run SLAM Toolbox; drive manually to build garden map; save map
- [ ] Run Nav2 with saved map; test waypoint navigation

### 🔲 Phase 4 — Grass Detection (TODO)
- [ ] Write grass detector ROS 2 node (HSV segmentation on `/image_raw`)
- [ ] Tune HSV thresholds for your specific lawn (use `rqt_image_view` to visualise mask)
- [ ] Implement visual servoing controller (P-controller on grass centroid offset)
- [ ] Add safety monitor (stop if grass coverage < 20% of frame)
- [ ] Integrate grass mask as a Nav2 costmap layer (marks non-grass as obstacles)

---

## Grass Detection — Design (not yet implemented)

### ROS 2 node plan
```
/image_raw  →  [grass_detector_node]  →  /grass_mask (Image)
                                      →  /grass_centroid (Point)

/grass_centroid  →  [navigation_controller_node]  →  /cmd_vel
/grass_mask      →  [safety_monitor_node]         →  /cmd_vel (emergency stop)
```

### HSV tuning approach
- Use `rqt_image_view` on the Pi to view the raw feed
- Open a Python script with `cv2.inRange()` and use trackbars to find the H/S/V min-max for your lawn's green
- Typical starting values: H: 30–90, S: 40–255, V: 30–200 (adjust for your lighting)

### Visual servoing logic
- If grass centroid is left of image centre → turn left (positive angular.z)
- If grass centroid is right of image centre → turn right (negative angular.z)
- If grass coverage < 20% → stop and rotate to find grass
- Controller type: proportional (P) on horizontal centroid offset, start with gain ~0.005

---

## Important Notes & Decisions Made (CONFIRM WITH PAUL, OTHER PAUL AND THE TEAM)
1. **PCA9685 is on the Pico's I2C bus, NOT driven from the Pi.** The Pico generates all motor PWM signals.
2. **BTS7960** was chosen over L298N because the RoboMow motors are high-torque and need >10 A capability.
3. **linorobot2_hardware rolling branch** is the target (Jazzy-compatible, Pico 2 supported).
4. **MOTOR_MAX_RPM = 80** is an estimate for 24 V operation. Must be verified with calibration tool before any autonomous driving.
5. **two_d_mode = true** in EKF — robot operates on flat garden ground, no 3D pose needed.
6. **Nav2 max speed = 0.25 m/s** — conservative for outdoor grass; can increase after testing.
7. The grass detector will plug into Nav2 as a **costmap layer**, not fight against Nav2's planner.
8. The URDF `robomow_properties.urdf.xacro` must be referenced from the existing `2wd.urdf.xacro` by updating its `<xacro:include>` line.

