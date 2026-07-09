# LiDAR SLAM Project - KISS-SLAM

## What system did I use?
KISS-SLAM. It's a LiDAR SLAM system made by the University of Bonn.

- Main code: https://github.com/PRBonn/kiss-slam
- ROS 2 version I actually ran: https://github.com/abwerby/kiss-slam-ros2 (this is a
  wrapper someone else built, not the official one, so it's a bit rough. More on
  that in the Problems section below.)

## What does it actually do?
Normal LiDAR odometry just compares one scan to the next and guesses how far the
sensor moved. KISS-SLAM does that too, but it also:
- builds and keeps an actual map while it runs,
- notices when it comes back to a place it already saw (this is called a "loop
  closure"),
- and fixes the map/path when that happens, instead of letting small errors pile up
  forever.

So it's a step up from plain odometry. It can catch and fix its own mistakes.

## What data did I use?
- Bag file: `lidar_bags/lidar_bag`
- LiDAR topic: `/livox/lidar`
- IMU topic in the bag: `/livox/imu` (I did not end up needing this one. This run
  only uses LiDAR data. No topics needed to be renamed.)

## How I installed it
```bash
sudo apt update
sudo apt install -y git python3-pip python3-colcon-common-extensions \
  libeigen3-dev libsuitesparse-dev

pip install --break-system-packages kiss-slam
```
(The `--break-system-packages` part is there because newer Ubuntu blocks normal
`pip install` by default. More on why I needed this in the Problems section.)

## How I built the ROS 2 wrapper
```bash
mkdir -p ~/slam_ws/src
cd ~/slam_ws/src
git clone https://github.com/abwerby/kiss-slam-ros2.git

cd ~/slam_ws
colcon build --symlink-install
source ~/slam_ws/install/setup.bash
```

## How I ran it
I used 3 terminal windows at the same time.

**Terminal 1: starts the SLAM node and opens RViz**
```bash
cd ~/slam_ws
source install/setup.bash
ros2 launch kiss_slam_ros slam.launch.py \
  topic:=/livox/lidar \
  visualize:=true \
  use_sim_time:=true
```

**Terminal 2: used to check that things were working**
```bash
ros2 topic list -t
ros2 topic hz /global_voxel_map
```

## What topics showed up
Once everything was running, these were the important ones:
- `/deskewed_points`, the cleaned up point cloud
- `/global_path`, the trajectory (the path the sensor took)
- `/global_voxel_map`, the map being built
- `/odometry`, the position and motion estimate
- `/tf`, the frame tree that connects everything together

In RViz I set the Fixed Frame to `odom`. I also had to manually add the map, path,
and points displays myself, since RViz does not show anything by default except the
TF axes.

## What I changed and why
| Setting | Default | What I set it to | Why |
|---|---|---|---|
| `topic` | `/ouster/points` | `/livox/lidar` | The wrapper assumes a different LiDAR brand by default, so I had to point it at the real topic in the Livox bag |
| `use_sim_time` | off | `true` | So the node uses the bag's recorded timestamps instead of my computer's real clock |

## Did it work?
Yes. Here is what I saw:

- The map (a white grid) and the point cloud (red dots) built up over time in RViz.
  Screenshots are below.
- `ros2 topic hz /global_voxel_map` showed the map topic updating steadily, about
  once every 1 to 1.3 seconds. That makes sense since the map does not need to
  update as often as raw scans do.
- On an earlier full run, the terminal log showed several loop closures. Most were
  accepted, with overlap scores between about 0.5 and 0.9. One was correctly turned
  down for being too low (0.39). So the system is not just accepting every closure,
  it is actually checking quality first.
- I did not see any sudden jumps, and the map did not look broken or doubled up. It
  looked steady and consistent.

![Early in the run](images/rviz_early.png)
![Later in the run, map has grown](images/rviz_result.png)

## Problems I ran into (and how I fixed them)
1. **`pip install` refused to run.** Newer Ubuntu blocks installing Python packages
   system-wide by default. Fixed it with `pip install --break-system-packages
   kiss-slam`.

2. **The SLAM node could not find the `kiss_icp` module**, even though I had it
   installed. It turned out I had installed it inside a virtual environment, but
   ROS 2 runs nodes using the normal system Python, not my virtual environment.
   Fixed it by installing the package into system Python instead (same command as
   above).

3. **`colcon build` created an `install/` folder in the wrong spot.** I had run the
   build command from inside the package folder instead of the main workspace
   folder. I deleted the extra `build`, `install`, and `log` folders and rebuilt
   from the correct spot (`~/slam_ws`).

4. **`/livox/lidar` was not publishing any data at first.** It turned out I just had
   not started playing the bag yet in another terminal. Once I did, data started
   showing up.

5. **RViz was empty except for a small axis marker.** RViz does not add anything on
   its own. You have to click "Add" and pick the topics you want to see (map, path,
   points).

6. **RViz still showed nothing even after I added the displays**, and the terminal
   printed a warning about "incompatible QoS." Basically, the SLAM node was sending
   data one way ("Best Effort"), and RViz was trying to listen a different way
   ("Reliable"). They just were not matching up, and no error showed why. I fixed it
   by opening each display's settings in RViz and changing its Reliability setting
   to "Best Effort" to match the node.

7. **Strange timestamp warnings showed up when I looped the bag on repeat.** When a
   looped bag restarts, its timestamps jump backward, and ROS treats that as old
   data and prints warnings. This is not really a bug, just a side effect of
   looping. For my final screenshots, I played the bag once through instead of
   looping it.

8. **I tried running the plain `kiss_icp_pipeline` by itself (without ROS)** and got
   nothing. It expects a different file format, not a ROS bag, so it just closes
   right away. I did not need this anyway. The ROS 2 launch file is the correct way
   to run this on the provided bags.

## What I learned
- The real difference between odometry and SLAM: odometry just estimates motion one
  step at a time, and small errors add up over time. SLAM (like this system) keeps
  an actual map and can go back and fix earlier mistakes once it recognizes a place
  it has already visited.
- When something in ROS 2 looks broken but there is no clear error, it is often a
  quiet mismatch somewhere (wrong topic name, wrong Python environment, or QoS
  settings not matching), not the code actually being wrong.
- Always build ROS 2 workspaces from the top level folder, not from inside a
  package. It saves a lot of confusion later.
