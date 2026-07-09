# LiDAR SLAM System Integration Project

## Selected System
KISS-SLAM (Option A)

## Repository
- Official repository: https://github.com/PRBonn/kiss-slam
- ROS 2 wrapper (third-party, experimental): https://github.com/abwerby/kiss-slam-ros2

## System Summary
KISS-SLAM is a LiDAR-only SLAM system built on top of KISS-ICP odometry. It adds:
- local map construction (voxel-based),
- loop closure detection via local-map overlap,
- pose graph optimization to correct drift once a closure is accepted.

Unlike pure LiDAR odometry, KISS-SLAM maintains a global map and corrects accumulated
drift whenever it revisits a previously mapped area (loop closure), rather than only
integrating relative motion between scans.

## Input Data
- **Bag used:** `$HOME/lidar_bags/lidar_bag`
- **LiDAR topic:** `/livox/lidar` (`sensor_msgs/msg/PointCloud2`)
- **IMU topic:** `/livox/imu` (`sensor_msgs/msg/Imu`) — available in the bag; KISS-SLAM
  as configured here runs LiDAR-only (see System section below)
- **Topic remapping:** none required — the wrapper's `topic:=` launch argument was set
  directly to `/livox/lidar`, overriding its default (`/ouster/points`)

## Installation
```bash
sudo apt update
sudo apt install -y git python3-pip python3-colcon-common-extensions \
  libeigen3-dev libsuitesparse-dev

# Ubuntu 23.04+/newer 22.04 block system-wide pip (PEP 668).
# Installed into system Python so the ROS2 node (system Python subprocess)
# can import kiss_slam at runtime:
pip install --break-system-packages kiss-slam

# sanity check
kiss_slam_pipeline --help
```

## Build Instructions
```bash
mkdir -p ~/slam_ws/src
cd ~/slam_ws/src
git clone https://github.com/abwerby/kiss-slam-ros2.git

cd ~/slam_ws        # IMPORTANT: build from the workspace root, not inside the package
colcon build --symlink-install
source ~/slam_ws/install/setup.bash
```

## Running on the Provided Bag
Three terminals were used.

**Terminal 1 — launch the SLAM node + RViz:**
```bash
cd ~/slam_ws
source install/setup.bash
ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  bagfile:=$HOME/lidar_bags/lidar_bag \
  visualize:=true \
  use_sim_time:=true
```

**Terminal 2 — verify topics are live:**
```bash
ros2 topic list -t
ros2 topic hz /global_voxel_map
```

**Terminal 3 — play the bag:**
```bash
ros2 bag play $HOME/lidar_bags/lidar_bag --clock
```

## Topic and Frame Configuration
- **Input:** `/livox/lidar` → passed via `topic:=/livox/lidar`
- **Fixed frame (RViz):** `odom`
- **Published outputs observed via `ros2 topic list -t`:**
  - `/deskewed_points` (`sensor_msgs/msg/PointCloud2`)
  - `/global_path` (`nav_msgs/msg/Path`)
  - `/global_pose` (`geometry_msgs/msg/PoseStamped`)
  - `/global_voxel_map` (`sensor_msgs/msg/PointCloud2`)
  - `/odometry` (`nav_msgs/msg/Odometry`)
  - `/tf`, `/tf_static`
- No manual topic remapping was needed beyond the `topic:=` launch argument.

## Parameter or Config Changes
| Parameter | Original value | New value | Reason |
|---|---|---|---|
| `topic` (launch arg) | `/ouster/points` | `/livox/lidar` | Wrapper defaults to an Ouster topic name; had to point it at the Livox bag's actual topic |
| `use_sim_time` | `false` (implied) | `true` | Required so the node uses bag-clock time (`/clock`) instead of wall-clock time while replaying a bag |

## Results
- `ros2 topic hz /global_voxel_map` confirmed the map topic publishing at roughly
  0.7–0.8 Hz during the run (expected — map updates are less frequent than raw scans).
- RViz screenshots show a growing voxel map (white grid mesh) and accumulated
  `deskewed_points` (red) forming a recognizable structure/corridor shape, with the
  `Path`/`Odometry` trajectory tracking the sensor's motion.
- See `images/rviz_early.png` (early in the run, small map) and
  `images/rviz_result.png` (further into the run, larger accumulated point cloud/map).

## Evaluation
- **Did the system run successfully?** Yes — the node started, subscribed to
  `/livox/lidar`, and published odometry, TF, path, and map topics.
- **Trajectory/map quality:** The accumulated point cloud in `rviz_result.png` shows a
  consistent, non-fragmented structure, suggesting odometry was reasonably accurate
  over the observed segment.
- **Loop closure:** In an earlier full run (see Problems section), the terminal log
  showed multiple `Closure Detected` events with local-map overlap scores ranging from
  ~0.39 to ~0.90; one closure (overlap 0.39) was correctly rejected for low overlap
  while the rest were accepted and triggered `Optimize Pose Graph`. This confirms the
  loop-closure and pose-graph-correction pipeline is functioning as intended.
- **Drift/jumps:** No sudden trajectory jumps were observed in RViz during the
  captured segment; the map appeared spatially consistent rather than duplicated
  or offset.

## Problems and Troubleshooting
1. **`externally-managed-environment` pip error** — Ubuntu blocks system-wide `pip
   install`. Solved with `pip install --break-system-packages kiss-slam`.
2. **`ModuleNotFoundError` for `kiss_icp` inside the SLAM node** — occurred when
   `kiss-slam` was installed only inside a Python venv. ROS2 launches nodes using the
   system Python interpreter, not the venv, so the package was invisible to the node.
   Solved by installing `kiss-slam` into system Python instead of a venv.
3. **`colcon build` produced `install/` in the wrong location** — build was
   initially run from inside the package folder (`~/slam_ws/src/kiss-slam-ros2`)
   instead of the workspace root (`~/slam_ws`), so `source ~/slam_ws/install/setup.bash`
   failed with "no such file." Solved by removing the stray `build/`, `install/`,
   `log/` folders from inside the package directory and re-running `colcon build`
   from `~/slam_ws`.
4. **`/livox/lidar` showed no data initially** — root cause was simply that the bag
   player had not been started yet in a separate terminal; confirmed with
   `ros2 bag info` and `ros2 topic hz /livox/lidar`.
5. **RViz showed only a TF axis marker, nothing else** — a fresh RViz session does
   not auto-add displays. Fixed by manually adding `PointCloud2` displays for
   `/deskewed_points` and `/global_voxel_map`, and a `Path`/`Odometry` display for
   `/global_path` and `/odometry`, and setting Fixed Frame to `odom`.
6. **QoS incompatibility silently blocked RViz from receiving the map** — terminal
   showed:
   ```
   [slam_node]: New subscription discovered on topic 'global_voxel_map', requesting
   incompatible QoS. No messages will be sent to it. Last incompatible policy: RELIABILITY
   ```
   The node publishes `/global_voxel_map` and `/global_pose` with **Best Effort**
   reliability, while RViz's display subscriber defaulted to **Reliable**. Because
   these are incompatible, zero messages were delivered — no error, no crash, just
   silence. Fixed by opening the display's **Topic** settings in RViz and manually
   setting **Reliability Policy** to **Best Effort** to match the publisher. This is a
   wrapper-side design choice worth noting for anyone reproducing this project.
7. **`TF_OLD_DATA` warnings when looping the bag** — using `ros2 bag play --loop`
   causes timestamps to jump backward when the bag restarts, which TF interprets as
   data from the past and discards. This is expected behavior with looped playback,
   not a system failure; documented here rather than treated as a bug. For clean
   single-pass evidence, the bag was played once (no `--loop`) for final screenshots.
8. **Bare `kiss_icp_pipeline` (non-ROS) produced no output** — this is not part of
   the required workflow. The standalone pipeline expects raw point-cloud formats
   (e.g. KITTI-style/PCD), not `.db3` ROS bags, so it exited almost immediately with
   nothing processed. Not used further; all results in this report come from the
   ROS2 wrapper (`slam.launch.py`), which is the correct/required path for this bag
   format.

## What I Learned
- The practical difference between LiDAR odometry (frame-to-frame motion estimate)
  and full SLAM: KISS-SLAM builds and maintains a persistent map, detects when the
  sensor revisits a previously seen area, and retroactively corrects the whole
  trajectory/map via pose graph optimization — something pure odometry cannot do.
- How to diagnose a "silent" ROS2 pipeline failure (no data reaching RViz) by
  checking, in order: is the bag playing → is the input topic correct → is the node
  alive and subscribed → is the node publishing → is a QoS mismatch silently
  dropping messages between a live publisher and subscriber.
- The importance of running `colcon build` from the workspace root, and of
  installing Python dependencies into whichever interpreter ROS2 actually launches
  nodes with (system Python, not an isolated venv), when a ROS2 node imports a
  pip-installed library at runtime.
