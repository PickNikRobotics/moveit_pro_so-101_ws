# MoveIt Pro workspace for the SO-101 arm

MoveIt Pro configuration for the **SO-101** (LeRobot) 5-DOF arm with a parallel-jaw
gripper, driven by Feetech STS3215 smart serial-bus servos. Built on
[MoveIt Pro](https://picknik.ai/moveit-pro/) 9.4.1 (ROS 2 Jazzy).

![SO-101 in the MoveIt Pro web UI](so101.png)

## Config packages

| Package | Type | Hardware |
|---|---|---|
| `so101_base_config` | Base / mock | `mock_components/GenericSystem` — no physics, commands echo back as state |
| `so101_hw` | Physical | Real Feetech STS3215 bus via `feetech_ros2_driver`, over a USB-TTL adapter |

`so101_hw` inherits from `so101_base_config` via `based_on_package` (same URDF
geometry, SRDF, controllers, kinematics) and only swaps the mock hardware interface
for the real Feetech driver. See **Physical hardware** below before running it.

## Robot description

The URDF, meshes, and end-effector xacros come from the upstream
[`so101-ros-physical-ai`](https://github.com/legalaspro/so101-ros-physical-ai)
project. Its `so101_description` package is vendored here (tracked, meshes via
git-LFS) at `src/so101_description/`. We build the **follower** variant (the
actuated arm).

The MoveIt Pro wrapper (`src/so101_base_config/description/so101.urdf.xacro`) adds
the `world` TF root, a `grasp_link` tool-center-point frame, and a ros2_control
block.

## Setup

```bash
# Clone (git-LFS is required for the mesh files)
git clone https://github.com/PickNikRobotics/moveit_pro_so-101_ws.git
cd moveit_pro_so-101_ws

# Point MoveIt Pro at this workspace and the base config
moveit_pro configure -w "$(pwd)" -c so101_base_config

# Build and run
moveit_pro build
moveit_pro run
```

Then open the web UI at http://localhost.

> This config targets ROS 2 Jazzy. Ensure `MOVEIT_ROS_DISTRO: jazzy` is set in
> `~/.config/moveit_pro/moveit_pro_config.9.yaml`.

## Basic motion test

From the web UI, run the **Move to Home** or **Move to Ready** objective, or from
a shell:

```bash
ros2 action send_goal --feedback /do_objective \
  moveit_studio_sdk_msgs/action/DoObjectiveSequence \
  "{objective_name: 'Move to Home'}"
```

Gripper: run the **Open Gripper** / **Close Gripper** objectives.

## Physical hardware (`so101_hw`)

Switch to the real-hardware config with:

```bash
moveit_pro configure -w "$(pwd)" -c so101_hw
moveit_pro build   # only needed the first time (compiles feetech_ros2_driver)
moveit_pro run
```

Before this will drive the arm, three things must be true:

1. **The USB-TTL adapter is visible to the OS.** On native Linux the Waveshare/Feetech
   adapter shows up as `/dev/ttyUSB0` (sometimes `/dev/ttyACM0`); set the matching
   `usb_port` in `src/so101_hw/config/config.yaml` (`urdf_params`). On **WSL2** the
   device is not visible until attached with
   [`usbipd-win`](https://learn.microsoft.com/windows/wsl/connect-usb):
   `usbipd list` then `usbipd attach --wsl --busid <id>` from Windows. The `drivers`
   container already bind-mounts `/dev`.
2. **The arm is calibrated for this unit.** `src/so101_hw/config/follower_joints.yaml`
   currently holds the *upstream* arm's calibration as a placeholder. Regenerate the
   `homing_offset` / `range_min` / `range_max` per servo for the office SO-101 (via the
   lerobot calibration flow or the `feetech_ros2_driver` tooling) — otherwise joints
   will be mis-centered and range-clipped.
3. **You confirm the servo variant.** 7.4V vs 12V STS3215 affects torque/power only, not
   the config.

With no device attached, launching `so101_hw` loads the Feetech driver and then aborts
at `Open [/dev/ttyUSB0]: Bad file descriptor` — that is expected; it means everything
except the physical link is wired correctly.

**First real motion is human-supervised.** Once `/joint_states` is live and controllers
are active, use the Bloodhound `/hardware-bringup-ladder` and `verify-hardware-motion`
workflows (single-joint twitch test, e-stop in hand) rather than commanding large moves
directly.

## Stop

```bash
moveit_pro down
```
