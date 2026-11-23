# NAVIGATION-AND-MAP-GENERATION-FOR-SEARCH-AND-RESCUE-OPERATION-USING-UNMANNED-AERIAL-VEHICLE

A simulated UAV project exploring indoor search-and-rescue navigation and map generation in a warehouse environment. An Iris quadrotor is simulated in Gazebo, while ROS 2 connects sensor data, mapping, visualization and movement-control components. The project includes a Deep Q-Network (DQN) reinforcement-learning prototype for selecting discrete movement actions based on map information.


> **Portfolio overview:** This public portfolio repository presents the project architecture, workflow and simulation evidence. The implementation repository can remain private. The DQN implementation is a prototype, and its limitations and unfinished areas are documented below.


## Project Results

### RViz point-cloud visualization

The provided RViz configuration visualizes the point cloud/map data in the map frame.

![RViz point-cloud view](output/Screenshot%20from%202025-04-30%2000-23-07.png)

### DQN training-loss plot

The training process plots loss during optimization. The academic project report records a Mean Squared Error (MSE) of **0.09**. This is a reported Q-value error/loss metric; it should not be interpreted as navigation accuracy, map accuracy, or a measured success rate.

![DQN training loss](output/Screenshot%20from%202025-05-08%2019-19-58.png)

### Gazebo simulation recording

[View the Gazebo simulation recording](output/Screencast%20from%202025-04-15%2015-53-31.mp4)


## Project Overview

The project is organized into three ROS 2 Python packages, along with the simulation assets.

| Component | Responsibility |
|---|---|
| `launch_world` | Launches Gazebo, spawns the Iris model, starts ROS–Gazebo bridges, RViz, RTAB-Map and the robot-state publisher. |
| `drone_controller` | Converts discrete movement commands into timed `Twist` velocity commands and publishes them to the simulated UAV. |
| `rl_for_drone` | Contains the DQN model, training loop, map-based environment interface and a separate inference script. |
| `working_model_5` | Contains the warehouse world, Iris URDF/meshes and RViz configurations. |
| `output` | Contains the screenshots and simulation recording shown in this README. |


### Runtime Data Flow

```text
Gazebo warehouse + Iris UAV
        │
        ├── ROS–Gazebo bridge ── camera, depth, point cloud, IMU, odometry, TF
        │
        ├── RTAB-Map ── SLAM map / point-cloud mapping
        │                    └── RViz visualization
        │
        └── DQN node ── map state ── discrete movement action
                                  │
                                  ▼
                         "iris/movement" command
                                  │
                                  ▼
                           drone_controller
                                  │
                                  ▼
                             /iris/cmd_vel
```


## Mapping and Sensor Pipeline

The simulated UAV uses an RGB-D camera and an IMU. Sensor data is bridged from Gazebo into ROS 2 and consumed by RTAB-Map to build a 3D SLAM map and a corresponding 2D grid-map representation. RViz is used to visualize mapping and point-cloud data.

- **RGB-D camera:** provides color images and depth information for perception and mapping.
- **IMU:** provides inertial measurements; the project report identifies issues with IMU readings that affected map quality.
- **RTAB-Map:** performs SLAM and produces the map/point-cloud representation used for visualization.
- **RViz:** visualizes the map and related sensor/robot data.


## DQN Reinforcement-Learning Prototype

The DQN component is intended to use local map information to select movement actions for the simulated UAV.

- **State input:** a 128 × 128 map region centered around the UAV's position (16,384 values) is the intended DQN input.
- **Network:** a fully connected PyTorch network with ReLU hidden layers and five Q-value outputs.
- **Discrete actions:** `up`, `down`, `turn_right`, `turn_left` and `forward`.
- **Learning components:** epsilon-greedy action selection, replay buffer (up to 5,000 transitions), mini-batches of 64, discount factor 0.99, Adam optimizer and a target network synchronized at episode boundaries.
- **Training configuration:** up to 1,000 episodes and 200 steps per episode; loss is plotted during training.
- **Movement control:** movement commands are converted to short velocity commands, followed by a zero-velocity command.


The launch setup also bridges IMU, odometry, TF, camera images/depth and point-cloud data. Although the IMU topic is subscribed to, the current DQN state is map-based and does not combine IMU features with the map.


## ROS Interfaces

| Interface | Use |
|---|---|
| `/map` (`nav_msgs/OccupancyGrid`) | Map input intended for the DQN environment. |
| `/imu` (`sensor_msgs/Imu`) | Bridged/subscribed sensor topic; not currently part of the DQN state. |
| `/world/world_demo/pose/info` (`geometry_msgs/PoseArray`) | Pose data consumed by the environment code. |
| `iris/movement` (`std_msgs/String`) | Discrete movement command from the DQN node to the controller. |
| `/iris/cmd_vel` (`geometry_msgs/Twist`) | Velocity command sent by the controller to Gazebo. |
| `/camera1/*`, `/camera2/*`, `/tf`, `/odom` | Sensor and state topics bridged from Gazebo for mapping and visualization. |
| `/cloud_map` | Point-cloud display topic configured in the RViz view. |


## Running the Simulation

The original workspace was developed for a Linux ROS 2 environment. A compatible ROS 2 distribution, Gazebo Sim with `ros_gz`, RViz 2, RTAB-Map ROS packages and the Python dependencies used by the nodes (`torch`, `numpy` and `matplotlib`) are required. The package manifests do not fully declare the runtime dependencies, so install them in the ROS environment before launching.


### 1. Update local asset paths

The launch and model files contain machine-specific absolute paths. Update them to match your local checkout in:


- `new_ros_packages/src/launch_world/launch/world_launch.py`
- `working_model_5/drone_simulation.world`
- `working_model_5/iris_drone/iris.urdf`


The world file also loads the warehouse model using a file URI. Make sure that URI points to your local `working_model_5/warehouse` directory.


### 2. Build and launch Gazebo, bridges, mapping and RViz

From a terminal in `new_ros_packages`:


```bash
source /opt/ros/$ROS_DISTRO/setup.bash
colcon build
source install/setup.bash
ros2 launch launch_world world_launch.py
```


Press **Play** in Gazebo if the simulation is paused. The launch file starts the movement controller, bridges, robot-state publisher, RViz and RTAB-Map. The configured RViz view uses the `map` fixed frame and displays `/cloud_map`.


### 3. Start DQN training

Keep the simulation terminal running. In a second terminal:


```bash
source /opt/ros/$ROS_DISTRO/setup.bash
cd /path/to/your/project/new_ros_packages
source install/setup.bash
ros2 run rl_for_drone rl_for_drone_1
```


The `rl_for_drone_1` executable starts the training entry point. The separate `DQN_Prediction.py` file contains inference code but is not currently registered as a ROS 2 executable in the package setup.


## Limitations and Future Work

This project is a simulation and learning prototype—not a validated, ready-to-fly autonomy system. The academic report describes simulation runs and DQN training, but also notes mapping inaccuracies and occasional collisions. A review of the current code snapshot identifies issues that need addressing before the DQN workflow can be considered reliable or reproducible.

- The environment's `reset()` is not implemented, so episodes do not currently reset the Gazebo world or drone.
- The contact-sensor subscription is commented out. The `contact` flag therefore does not currently provide collision detection or a meaningful terminal condition.
- The training/inference code needs ROS callback processing while it runs; the current main loops do not spin the node, so callback servicing must be verified and corrected before relying on live map or pose updates.
- The DQN transition target does not mask terminal states, and the current reward is based on changes in map cells rather than a clearly defined goal or obstacle-avoidance objective.
- The map callback currently stores only the first cell value, while the state conversion method expects a full occupancy-grid message; the map-state path needs correction.
- Model weights use a hard-coded local path. Configure a portable model directory before running on another machine.
- Mapping quality was limited in the report, with IMU-related issues and occasional obstacle collisions. The reported MSE does not establish navigation or map accuracy.
- Validate learned behavior, reward design, map-state alignment and safety constraints in simulation before drawing conclusions about navigation performance.


Potential next steps include parameterizing paths, implementing world/drone reset and collision handling, reliably processing ROS callbacks, correcting map-state extraction, defining a goal-based reward, registering an inference executable, collecting quantitative evaluation metrics and documenting exact ROS/Gazebo versions.
