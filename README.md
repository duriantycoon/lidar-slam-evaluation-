# KISS-SLAM with ROS 2: Installation and Operation Guide

This document describes the procedure for installing and running KISS-SLAM (https://github.com/PRBonn/kiss-slam) using the ROS 2 wrapper kiss-slam-ros2 (https://github.com/abwerby/kiss-slam-ros2) on recorded LiDAR bag data. It combines the wrapper's published instructions with the additional steps required to run it successfully, based on direct testing.

The wrapper is a third-party package, not an official release from the KISS-SLAM authors.

## 1. System Description

KISS-SLAM extends standard LiDAR odometry with mapping and loop closure. Standard odometry estimates sensor motion between consecutive scans only, so estimation errors accumulate without correction. KISS-SLAM additionally:

1. constructs and maintains a map during operation,
2. detects loop closures, defined as the recognition of a previously visited location, and
3. corrects the map and trajectory when a loop closure is confirmed.

## 2. Prerequisites

```bash
sudo apt update
sudo apt install -y git python3-pip python3-colcon-common-extensions \
  libeigen3-dev libsuitesparse-dev
```

## 3. Installing KISS-SLAM

The published installation command is:

```bash
pip install kiss-slam
```

On recent Ubuntu releases, this command fails because system-wide package installation via pip is disabled by default (the environment is marked as "externally managed"). The following command must be used instead:

```bash
pip install --break-system-packages kiss-slam
```

### 3.1 Python Environment Requirement

The package must be installed into the system Python interpreter, not a virtual environment, if the ROS 2 wrapper will be used. `ros2 launch` executes nodes using the system Python interpreter. If `kiss-slam` is installed only inside a virtual environment, the node process will not have access to it, and will fail with an error of the form:

```
ModuleNotFoundError: No module named 'kiss_icp'
```

This occurs even if the module imports correctly within the virtual environment itself, because the ROS 2 process does not use that environment. The installation command in Section 3 should be run without an active virtual environment.

A virtual environment installation is only appropriate if the standalone `kiss_slam_pipeline` command-line tool will be used without ROS 2.

## 4. Standalone Pipeline (Optional, Non-ROS Usage)

```bash
kiss_slam_pipeline --help
kiss_slam_dump_config
```

The second command generates a `kiss_slam.yaml` configuration file, which can be edited and supplied via the `--config` option.

Note: `kiss_slam_pipeline` requires its own input file format and does not accept ROS 2 bag files (`.bag` or `.db3`). If a ROS bag is passed to it, the process exits immediately without processing data. For ROS bag input, use the ROS 2 wrapper described below.

Note on naming: the command name used above (`kiss_slam_pipeline`) is taken from the wrapper's published README. Records of the original test run referred to this command as `kiss_icp_pipeline`, which does not match the published README. This may reflect a naming inconsistency in the source notes, or a difference between package versions. This should be verified locally with `kiss_slam_pipeline --help` before relying on either name.

## 5. Building the ROS 2 Workspace

```bash
mkdir -p ~/slam_ws/src
cd ~/slam_ws/src
git clone https://github.com/abwerby/kiss-slam-ros2.git

cd ~/slam_ws
colcon build --symlink-install
source ~/slam_ws/install/setup.bash
```

`colcon build` must be run from the workspace root directory (`~/slam_ws`), not from within the package directory. Running it from inside the package directory produces `build/`, `install/`, and `log/` directories in an incorrect location, which will cause sourcing errors.

If this occurs, remove the incorrectly placed directories and rebuild from the workspace root:

```bash
cd ~/slam_ws/src/kiss-slam-ros2
rm -rf build install log
cd ~/slam_ws
colcon build --symlink-install
```

## 6. Launch Arguments

| Argument | Default | Description |
|---|---|---|
| `topic` | `/ouster/points` | Input point cloud topic |
| `visualize` | `true` | Launches RViz |
| `bagfile` | none | Path to a bag file to play automatically |
| `use_sim_time` | `true` | Uses bag-recorded timestamps rather than system clock; should remain `true` for bag playback |
| `base_frame` | `base_link` | Robot base frame |
| `odom_frame` | `odom` | Odometry frame |

The default `topic` value assumes an Ouster sensor. For other sensors, this argument must be set explicitly. For example, for a Livox sensor publishing on `/livox/lidar`:

```bash
ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

Parameters can also be modified directly in `ros2/src/kiss_slam_ros/config/params.yaml`.

## 7. Running the System

Three terminal sessions are required.

**Terminal 1: launch the SLAM node and RViz**

```bash
cd ~/slam_ws
source install/setup.bash
ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

**Terminal 2: verification commands**

```bash
ros2 topic list -t
ros2 topic hz /global_voxel_map
```

**Terminal 3: play the bag file**

```bash
ros2 bag play lidar_bags/lidar_bag
```

This is the standard ROS 2 command for bag playback, using the bag path referenced in Section 1. If no data appears on the input topic, confirm that the bag has been started in this terminal.

### 7.1 Relevant Topics

| Topic | Description |
|---|---|
| `/deskewed_points` | Motion-corrected point cloud |
| `/global_path` | Estimated trajectory |
| `/global_voxel_map` | Map under construction |
| `/odometry` | Position and motion estimate |
| `/tf` | Coordinate frame tree |

## 8. RViz Configuration

RViz does not display any topic data by default; only the TF axis marker is shown initially. The following must be set manually:

1. Set the Fixed Frame to `odom`.
2. Add displays for the map, path, and point cloud topics (`/global_voxel_map`, `/global_path`, `/deskewed_points`) using the Add button.

### 8.1 QoS Reliability Mismatch

If displays remain empty after being added, and the terminal output includes a warning referencing incompatible QoS settings, the cause is a Reliability policy mismatch: the SLAM node publishes using the "Best Effort" policy, while RViz subscribes using "Reliable" by default. No connection is established between publisher and subscriber, and no explicit error is produced beyond the QoS warning.

Resolution: in each RViz display's settings, set the Reliability field to "Best Effort" to match the publisher.

## 9. Verifying Correct Operation

Indicators of correct operation:

- The map and point cloud grow progressively in RViz over the duration of playback.
- `ros2 topic hz /global_voxel_map` reports an update rate of approximately once every 1.0 to 1.3 seconds. This is lower than the raw scan rate, which is expected, as the map does not require updating on every scan.
- Loop closure events appear in the node's log output. Accepted closures were observed with overlap scores in the range of approximately 0.5 to 0.9. At least one closure with a lower overlap score (0.39) was rejected, indicating that closures are evaluated against a quality threshold rather than accepted unconditionally.
- No discontinuities or duplicated geometry appear in the constructed map.

## 10. Summary of Issues and Resolutions

| Issue | Cause | Resolution |
|---|---|---|
| `pip install kiss-slam` fails | System-wide pip installation blocked by default on recent Ubuntu | Use `pip install --break-system-packages kiss-slam` |
| `ModuleNotFoundError: kiss_icp` when launching the node | Package installed in a virtual environment; ROS 2 uses the system Python interpreter | Install into system Python using the command above, without an active virtual environment |
| `install/` directory created in an unexpected location | `colcon build` run from within the package directory instead of the workspace root | Remove `build`, `install`, `log` directories and rebuild from the workspace root |
| No data on the input LiDAR topic | Bag playback not yet started | Start `ros2 bag play <bagfile>` in a separate terminal |
| RViz displays no data | No displays are added by default | Manually add displays for map, path, and point cloud topics |
| RViz still shows no data; QoS warning in terminal | Reliability policy mismatch between publisher (Best Effort) and subscriber (Reliable) | Set RViz display Reliability to Best Effort |
| Timestamp warnings during looped bag playback | Timestamps decrease when a looped bag restarts, which ROS interprets as delayed data | Play the bag through once rather than looping, particularly when capturing results |
| `kiss_slam_pipeline` exits immediately when given a bag file | The standalone pipeline requires its own file format, not ROS bag files | Use the ROS 2 launch file for ROS bag input |

## 11. General Observation

When a ROS 2 node produces no visible output but no explicit error is logged, the cause is frequently a configuration mismatch, such as an incorrect topic name, a Python environment discrepancy, or a QoS policy mismatch, rather than a defect in the underlying code. These should be checked first when diagnosing similar issues.
