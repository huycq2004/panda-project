# Panda Robot Pick-and-Place with OBB Detection

ROS 2 workspace for a simulated **Franka Emika Panda** robot that detects objects from a Gazebo camera, lets an operator select an object in a PyQt5 interface, and plans a pick-and-place motion through MoveIt 2.

The workspace supports two detection paths:

- **Bolts** — YOLOv8 oriented-bounding-box (OBB) inference.
- **Cubes** — OpenCV colour segmentation for red, green, and blue cubes.

> This is a simulation-focused project. The supplied launch file starts Gazebo, controller bridges, RViz, MoveItPy, and the Panda robot. It must be reviewed and adapted before commanding physical hardware.
<img width="945" height="529" alt="image" src="https://github.com/user-attachments/assets/f03b4dd1-e747-491d-a450-517ae8d53929" />
<img width="945" height="529" alt="image" src="https://github.com/user-attachments/assets/352ec188-d8be-491a-a2ac-5612bcabf27f" />
<img width="945" height="529" alt="image" src="https://github.com/user-attachments/assets/1b8207be-9a47-4680-ae65-6fe62cc6544a" />

## Features

- Gazebo simulation of a Panda arm, gripper, tabletop, camera, cubes, and bolts.
- MoveIt 2 planning and execution for the `panda_arm` and `hand` groups.
- YOLOv8-OBB inference with an included model checkpoint, `best.pt`.
- Colour-cube detection using HSV masks and oriented rectangles.
- PyQt5 visual selection interface for detected bolts or cubes.
- Automatic pick, lift, transfer, and release after an object selection.

## System overview

```text
Gazebo camera (/image_raw)
          |
          +--> YOLOv8-OBB --> /Yolov8_Inference --> bolt_selector.py
          |
          +--> OpenCV cubes -> /Cube_Inference ----> cube_selector.py
                                                       |
                                                   /target_point
                                                       |
                                             MoveItPy controller
                                                       |
                                           Panda arm + gripper in Gazebo
```

The UI derives an `(x, y, yaw)` target from the selected object centre and publishes it on `/target_point`. The controller interprets a fourth value, when supplied, as the object type: `1` for a cube and `0` for a bolt. It then performs the configured pick-and-place sequence.

## Repository layout

```text
.
├── UI/                              # PyQt5 selection applications and generated UI modules
└── src/
    ├── cube_detection/              # HSV colour cube detector
    ├── panda_moveit_config/         # MoveIt 2, Gazebo worlds, controllers, and arm controller
    ├── robot_description/           # Panda URDF/Xacro and mesh assets
    ├── yolov8_obb/                  # YOLOv8-OBB inference nodes and checkpoint
    └── yolov8_obb_msgs/             # Builds the ROS package named yolov8_msgs
```

## Requirements

### ROS and simulation

Use Ubuntu with a ROS 2 distribution compatible with the installed MoveIt 2 and Gazebo (`ros_gz_*`) packages. The project expects:

- ROS 2, `colcon`, and `rosdep`
- MoveIt 2, `moveit_py`, RViz 2, and OMPL/STOMP planners
- Gazebo Sim, `ros_gz_sim`, and `ros_gz_bridge`
- `ros2_control`, controller manager, and joint-state / robot-state publishers
- `xacro`, `cv_bridge`, `sensor_msgs`, `geometry_msgs`, and `std_msgs`

Install matching packages for your ROS distribution (replace `$ROS_DISTRO` with the sourced distribution name):

```bash
sudo apt update
sudo apt install \
  ros-$ROS_DISTRO-moveit \
  ros-$ROS_DISTRO-ros-gz-sim \
  ros-$ROS_DISTRO-ros-gz-bridge \
  ros-$ROS_DISTRO-ros2-control \
  ros-$ROS_DISTRO-ros2-controllers \
  ros-$ROS_DISTRO-cv-bridge \
  ros-$ROS_DISTRO-xacro \
  python3-colcon-common-extensions \
  python3-rosdep
```

Package names can differ between ROS/Gazebo releases; use `rosdep` below as the source of truth for unresolved dependencies.

### Python packages

The detection nodes and GUIs require Python 3 with Ultralytics, OpenCV, NumPy, and PyQt5:

```bash
python3 -m pip install ultralytics opencv-python numpy PyQt5
```

If ROS provides OpenCV or PyQt5 through system packages, prefer those packages so the interpreter used by ROS can import them.

## Installation

Clone into a ROS 2 workspace and install dependencies:

```bash
git clone https://github.com/huycq2004/panda-project.git ~/moveit2_obb
cd ~/moveit2_obb

source /opt/ros/$ROS_DISTRO/setup.bash
rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
```

### Model checkpoint path

`src/yolov8_obb/scripts/yolov8_obb_publisher.py` currently loads the model from:

```text
~/moveit2_obb/src/yolov8_obb/scripts/best.pt
```

The clone path above matches that expectation. If you use another workspace directory, either update the model path in the script or create an equivalent symbolic link. The 6.4 MB checkpoint is already tracked in this repository.

## Running the simulation

Use a terminal per process and source the workspace in each one:

```bash
source /opt/ros/$ROS_DISTRO/setup.bash
source ~/moveit2_obb/install/setup.bash
```

### 1. Start Panda, Gazebo, MoveIt, and RViz

```bash
ros2 launch panda_moveit_config moveit_gazebo_obb.py
```

The default world is `arm_on_the_table`. To request another SDF world from `src/panda_moveit_config/worlds`, pass its basename:

```bash
ros2 launch panda_moveit_config moveit_gazebo_obb.py world:=arm_on_the_table
```

### 2A. Detect and select a bolt

Start YOLOv8-OBB inference:

```bash
ros2 launch yolov8_obb yolov8_obb.launch.py
```

In another terminal, run the bolt selector from the `UI` directory so its generated module can be imported:

```bash
cd ~/moveit2_obb/UI
python3 bolt_selector.py
```

Move the pointer near an outlined bolt and click its centre. The GUI publishes the target and the MoveIt controller executes the configured pick-and-place movement.

### 2B. Detect and select a coloured cube

Start the OpenCV cube detector:

```bash
ros2 launch cube_detection cube_detection.launch.py
```

Then launch the cube selector:

```bash
cd ~/moveit2_obb/UI
python3 cube_selector.py
```

The cube selector highlights red, green, and blue cube candidates. Click an object centre to send a cube pick request.

Run one detector/UI pair at a time unless you intentionally want both systems to publish targets.

## ROS interfaces

| Topic | Type | Publisher | Consumer | Purpose |
| --- | --- | --- | --- | --- |
| `/image_raw` | `sensor_msgs/msg/Image` | Gazebo bridge | YOLO node, cube node, GUIs | Camera image |
| `/Yolov8_Inference` | `yolov8_msgs/msg/Yolov8Inference` | YOLO node | Bolt selector | Bolt OBB detections |
| `/Cube_Inference` | `yolov8_msgs/msg/Yolov8Inference` | Cube detector | Cube selector | Colour-cube OBB detections |
| `/inference_result` | `sensor_msgs/msg/Image` | YOLO node | Optional visualisation | Annotated YOLO image |
| `/target_point` | `std_msgs/msg/Float64MultiArray` | Selection UI | MoveItPy controller | `[x, y, yaw, object_type?]` pick request |

### Custom messages

The folder `src/yolov8_obb_msgs` builds a package named **`yolov8_msgs`**. It defines:

```text
# InferenceResult
string class_name
float64[] coordinates   # Four OBB corners, flattened: x1, y1, ..., x4, y4

# Yolov8Inference
InferenceResult[] yolov8_inference
std_msgs/Header header
```

## Configuration notes

- Camera intrinsics, camera height, and camera-to-arm offset are calibrated directly in `UI/bolt_selector.py` and `UI/cube_selector.py`. Recalibrate them after changing the camera pose or scene geometry.
- Pick heights, carrying height, drop location, and gripper opening are set in `src/panda_moveit_config/scripts/arm_control_from_UI.py`.
- The MoveIt controller plans with OMPL by default; its planning parameters are in `src/panda_moveit_config/config/controller_setting.yaml`.
- YOLO confidence is fixed at `0.90` in `yolov8_obb_publisher.py`.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `Package ... not found` | Re-run `colcon build --symlink-install` and `source install/setup.bash` in the current terminal. |
| YOLO node cannot find `best.pt` | Use the expected `~/moveit2_obb` workspace path or update the hard-coded path in `yolov8_obb_publisher.py`. |
| No objects appear in the selector | Confirm `/image_raw` exists with `ros2 topic echo /image_raw --once`, then start the corresponding detector. |
| Robot does not move after click | Check `ros2 topic echo /target_point`, confirm `arm_control_from_UI.py` is running, and review the MoveIt/Gazebo terminal output. |
| Planning fails | Verify controller services and current joint states, then check target calibration and bounds in the controller script. |
| Import errors for `ultralytics`, `cv2`, or `PyQt5` | Install dependencies into the same Python environment used by ROS 2. |

## Development workflow

After editing Python scripts, rebuild and re-source the workspace:

```bash
cd ~/moveit2_obb
colcon build --symlink-install
source install/setup.bash
```

Useful inspection commands:

```bash
ros2 topic list
ros2 topic echo /Yolov8_Inference
ros2 topic echo /Cube_Inference
ros2 topic echo /target_point
ros2 node list
```

## Safety

The controller contains fixed positions and heights intended for the supplied simulation scene. Do not reuse these values on a physical robot without collision checking, hardware-specific limits, an emergency-stop plan, and a full safety review.

## License

No license has been specified for this repository. Add an explicit license before redistributing or using the project in a context that requires one.
