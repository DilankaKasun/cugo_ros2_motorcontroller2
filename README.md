# cugo_ros2_motorcontroller2

ROS 2でクローラロボット開発プラットフォームを制御するArduinoスケッチです。

セットでROS 2パッケージの[cugo_ros2_control2](https://github.com/CuboRex-Development/cugo_ros2_control2)と使用します。

> [!WARNING]
> このArduinoスケッチはクローラロボット開発プラットフォーム CuGo V4 / V3i 専用です。
>
> ROS開発キット CuGo V3 をご利用の方は[こちら](https://github.com/CuboRex-Development/cugo_ros_motorcontroller/tree/uno-udp)を参照してください。

# Table of Contents
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
  - [Arduino: Pico Firmware](#arduino-pico-firmware)
  - [Ubuntu: ROS 2 Setup](#ubuntu-ros-2-setup)
  - [Web Dashboard Setup](#web-dashboard-setup)
- [Usage](#usage)
  - [Launch Robot Control](#launch-robot-control)
  - [Launch Dashboard](#launch-dashboard)
- [Protocol](#protocol)
- [Troubleshooting](#troubleshooting)
- [License](#license)

# Features

cugo_ros2_motorcontroller2では、ROS 2パッケージの [cugo_ros2_control2](https://github.com/CuboRex-Development/cugo_ros2_control2)から受信した目標回転数になるようにモータ制御し、エンコーダから取得したカウント数をPCに返信します。

また、付属のプロポ（MR-8）を用いて、コントローラ操作と自律走行の切り替えができます。左スティックを左に倒すことでコントローラ操作を受け付けるRC-MODEに、右に倒すことでROSの速度ベクトルを受け付けるROS-MODEに切り替えることができます。

自律走行中に誤った判断をして障害物に衝突しそうなシーンでは、プロポによりRC-MODEに変更することで、コントローラ操作に即時に切り替わりロボットを止めることができます。

#### 対応製品
* クローラロボット開発プラットフォーム CuGo V4
* クローラロボット開発プラットフォーム CuGo V3i

# Requirements

### Hardware
* Raspberry Pi Pico (included with robot platform)
* USB cable for Pico-PC connection
*クローラロボット開発プラットフォーム (CuGo V4 / V3i)

### Arduino Libraries
* Servo.h
* PacketSerial.h
* RPi_Pico_TimerInterrupt.h
* hardware/watchdog.h

### Ubuntu 22.04 / ROS 2 Humble
* ROS 2 Humble Hawksbill ([install guide](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html))
* `python3-rosdep`
* `python3-serial`
* `ros-humble-xacro`
* `ros-humble-robot-state-publisher`
* `ros-humble-rosbridge-server`
* `ros-humble-diagnostic-updater`
* Boost.Asio (usually installed with ROS 2)

# Installation

## Arduino: Pico Firmware

1. Install [Arduino IDE](https://www.arduino.cc/en/software) with Raspberry Pi Pico board support
2. Install required libraries via Library Manager:
   - `PacketSerial` by Christopher Baker
   - `RPi_Pico_TimerInterrupt` by khoih-prog
3. Open `cugo_ros2_motorcontroller2/cugo_ros2_motorcontroller2.ino` in Arduino IDE
4. Connect Pico via USB, select board `Raspberry Pi Pico`, and upload

> [!IMPORTANT]
> 出荷直後は安全のため、DIP-SWによってRC-MODEのみに制限されています。
> [準備2-6](https://github.com/CuboRex-Development/cugo-beginner-programming/tree/pico?tab=readme-ov-file#2-6-ld-2%E3%81%AE%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%83%A2%E3%83%BC%E3%83%89%E3%82%92%E6%9C%89%E5%8A%B9%E5%8C%96)の操作を行い、ROS-MODEを有効化してください。

## Ubuntu: ROS 2 Setup

1. **Create workspace and clone packages:**
```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/CuboRex-Development/cugo_ros2_control2.git
```

2. **Install dependencies:**
```bash
sudo apt update
sudo apt install -y python3-rosdep python3-serial ros-humble-xacro \
  ros-humble-robot-state-publisher ros-humble-rosbridge-server \
  ros-humble-diagnostic-updater
sudo rosdep init && rosdep update
rosdep install -i --from-paths ~/ros2_ws/src/cugo_ros2_control2
```

3. **Build the package:**
```bash
cd ~/ros2_ws
colcon build --symlink-install
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

## Web Dashboard Setup

A browser-based control dashboard is available at `~/cugo_dashboard/`:

```bash
mkdir -p ~/cugo_dashboard
```

Place `index.html` and `start.sh` (from this repo's `dashboard/` directory) in `~/cugo_dashboard/`.

The dashboard provides:
- Virtual joystick (mouse + touch support)
- Speed slider
- Encoder and odometry readouts
- Mode indicator (RC / CMD)
- Emergency stop button
- Connection status indicator

# Usage

## Launch Robot Control

1. **Set serial permissions:**
```bash
sudo usermod -aG dialout $USER
# Log out and back in, OR:
sudo chmod 666 /dev/ttyACM0
```

2. **Launch the node:**
```bash
source ~/ros2_ws/install/setup.bash
ros2 launch cugo_ros2_control2 cugov4_launch.py serial_port:=/dev/ttyACM0
```

3. **Publish a test command:**
```bash
# In another terminal
ros2 topic pub -1 /cmd_vel geometry_msgs/Twist "{linear: {x: 0.1}}"
```

4. **To stop:**
```bash
ros2 topic pub -1 /cmd_vel geometry_msgs/Twist "{linear: {x: 0.0}}"
```

## Launch Dashboard

Use the all-in-one launch script (starts ROS 2 node + rosbridge + dashboard web server):

```bash
cd ~/cugo_dashboard && bash start.sh
```

Then open `http://localhost:8000` in your browser.

Or start components manually:

```bash
# Terminal 1: Robot control node
ros2 launch cugo_ros2_control2 cugov4_launch.py serial_port:=/dev/ttyACM0

# Terminal 2: Rosbridge WebSocket server
ros2 launch rosbridge_server rosbridge_websocket_launch.xml

# Terminal 3: Simple HTTP server for dashboard
cd ~/cugo_dashboard && python3 -m http.server 8000
```

Then open `http://localhost:8000` in your browser.

**Note on RC/CMD mode:**
The robot will not move unless the LD-2 driver is in **CMD mode** (ROS mode). If the dashboard shows "Mode: RC", flip the physical RC transmitter switch on the robot to the CMD position. The Pico firmware forwards RPM commands regardless of mode, but the LD-2 ignores RPM commands in RC mode.

# Protocol

The ROS 2 PC and Pico communicate via USB serial at 115200 baud using COBS-encoded 72-byte packets (8-byte header + 64-byte body).

### PC → Pico (Send)

| Offset | Size | Type | Name | Description |
|--------|------|------|------|-------------|
| 0 | 2 | uint16 | product_id | 0=V4, 1=V3i |
| 2 | 2 | uint16 | robot_id | Reserved (0) |
| 4 | 2 | uint16 | length | Always 72 |
| 6 | 2 | uint16 | checksum | Body checksum (16-bit ones complement) |
| 8 | 4 | float | target_rpm_l | Left motor target RPM |
| 12 | 4 | float | target_rpm_r | Right motor target RPM |

### Pico → PC (Receive)

| Offset | Size | Type | Name | Description |
|--------|------|------|------|-------------|
| 0 | 4 | int32 | encoder_l | Left encoder count |
| 4 | 4 | int32 | encoder_r | Right encoder count |
| 8 | 1 | uint8 | run_mode | 0=RC mode, 1=CMD mode |

# Troubleshooting

### Robot does not respond to /cmd_vel

1. **Check mode:**
   Look for `Mode=RC` or `Mode=CMD` in the node logs. If `Mode=RC`, the LD-2 driver is in RC mode. Flip the RC transmitter switch (left stick to the right = CMD/ROS mode).

2. **Check serial port:**
```bash
ls -l /dev/ttyACM0
# Ensure permissions are correct (crw-rw---- with dialout group)
```

3. **Check for port conflicts:**
```bash
sudo fuser -k /dev/ttyACM0   # Kill any process holding the port
sudo fuser -k 9090/tcp        # Kill stale rosbridge processes
```

4. **Verify cmd_vel is published:**
```bash
ros2 topic echo /cmd_vel
# Move the joystick on the dashboard and verify messages appear
```

5. **Check encoder data:**
   The node prints `Encoder: L=<count>, R=<count>` every 100ms. If counts change when motors run, encoder feedback is working.

### Dashboard shows "Disconnected"

- Ensure rosbridge is running: `ros2 run rosbridge_server rosbridge_websocket`
- Check port 9090 is not blocked: `sudo fuser -k 9090/tcp`
- Verify WebSocket URL in dashboard matches `ws://hostname:9090`

### Permission denied on /dev/ttyACM0

```bash
sudo usermod -aG dialout $USER
# Log out and back in, or run:
sudo chmod 666 /dev/ttyACM0
```

# License

このプロジェクトはApache License 2.0のもと、公開されています。詳細はLICENSEをご覧ください。
