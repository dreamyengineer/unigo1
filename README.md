# Unitree Go1 Simulation in ROS2

This repository contains the files and modifications necessary to simulate and visualize a Unitree Go1 robot using **ROS2 (Humble)** and **Gazebo Classic 11**. It integrates:

- **SLAM** via [RTABMap](http://wiki.ros.org/rtabmap_ros)  
- **Navigation** via [Nav2](https://navigation.ros.org/)  
- A workaround for robot control using a **differential drive** plugin and teleoperation  

The project demonstrates how to modify ROS1-specific Unitree Go1 files to function in a ROS2 environment, including configuring URDF/XACRO files, updating Gazebo Classic plugins, and setting up a simple image flipping node.

---

## Table of Contents

1. [Features](#features)  
2. [Requirements](#requirements)  
3. [Installation](#installation)  
4. [Usage](#usage)  
   - [Launching Gazebo with the Robot](#1-launch-gazebo-with-the-robot)  
   - [Teleoperation](#2-teleoperation)  
   - [Optional Image Flip Node](#3-optional-image-flip-node)  
   - [Launching SLAM (RTABMap)](#4-launching-slam-rtabmap)  
   - [Launching Nav2](#5-launching-nav2)  
5. [How It Works](#how-it-works)  
   - [Modified XACRO Files](#modified-xacro-files)  
   - [Camera Frames and Image Flipping](#camera-frames-and-image-flipping)  
   - [Differential Drive Workaround](#differential-drive-workaround)  
   - [Map Creation and Navigation](#map-creation-and-navigation)  
6. [Future Work](#future-work)  
7. [Appendix: Project Structure](#appendix-project-structure)

---

## Features

- **ROS2 Migration**: Adapts the Unitree Go1 XACRO and URDF files (originally for ROS1) to work with ROS2 (Humble).  
- **Gazebo Classic 11 Integration**: Updates and replaces incompatible Gazebo plugins.  
- **SLAM & Mapping**: Uses RTABMap to build a 3D map and perform loop closure.  
- **Path Planning & Navigation**: Integrates Nav2 for global (Dijkstra/Hybrid A*) and local (DWB) path planning.  
- **Image Flipping**: Includes a Python node to flip camera images 180° to accommodate the Go1’s inverted camera orientation.

---

## Requirements

- **Operating System**: Ubuntu 22.04 (Jammy)  
- **ROS2 Distro**: Humble Hawksbill  
- **Gazebo Classic 11**: Installed from ROS2 repositories  
- **colcon**:
```bash
sudo apt install python3-colcon-common-extensions
```
- **RTABMap (ROS2)**:
```bash
sudo apt install ros-humble-rtabmap-ros
```
- **Nav2**:
```bash
sudo apt install ros-humble-navigation2
sudo apt install ros-humble-nav2-bringup
```
- **Teleoperation**:
```bash
sudo apt install ros-humble-teleop-twist-keyboard
```
- **Visual Studio Code** or another text editor (recommended)  
- (Optional) **Python OpenCV** for image flipping:
```bash
sudo apt install python3-opencv
```

> **Note**: Make sure you have a functional ROS2 workspace set up prior to building this package.

---

## Installation

1. **Clone this repository** into your ROS2 workspace (e.g., `~/ros2_ws/src`):
   ```bash
   cd ~/ros2_ws/src
   git clone <this-repo-url> unigo1
   ```
2. **Build the workspace**:
   ```bash
   cd ~/ros2_ws
   colcon build
   source install/setup.bash
   ```
3. **Verify** that ROS2 and Gazebo versions match (ROS2 Humble & Gazebo Classic 11).

---

## Usage

Below are the basic steps to launch the simulation, control the robot, run SLAM (RTABMap), and navigate with Nav2.

### 1. Launch Gazebo with the Robot

Launch Gazebo Classic 11 and spawn the Unitree Go1 robot:

```bash
ros2 launch unigo1 gazebolauncher.launch.py
```

- This will start Gazebo, load the robot model, and publish the robot’s state to ROS topics.  
- Optionally, you can edit the launch file to specify a custom world file.

### 2. Teleoperation

In a **new terminal**, run the teleoperation node to manually control the robot (via keyboard commands):

```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

### 3. Optional Image Flip Node

If you need corrected camera views (due to the robot’s inverted camera orientation), launch the Python script in another terminal:

```bash
ros2 run py_imageflip image_flip_node.py
```

- The flipped images will be published on `/flipped_image_raw`.  
- Adjust your SLAM or RViz settings to use the flipped topic if needed.

### 4. Launching SLAM (RTABMap)

In a **new terminal**, start the RTABMap launch file:

```bash
ros2 launch unigo1 unigo1_rgbd.launch.py
```

- This sets up RTABMap, subscribing to the camera/depth topics and publishing an evolving map of the environment.  
- Make sure your camera topics match what RTABMap expects (remapping may be configured inside the launch file).

### 5. Launching Nav2

Finally, in **new terminals**, launch the Nav2 stack and RViz:

```bash
# Terminal 1: Nav2
ros2 launch nav2_bringup navigation_launch.py use_sim_time:=True

# Terminal 2: RViz with Nav2 config
ros2 launch nav2_bringup rviz_launch.py
```

- Inside RViz, use the **2D Goal Pose** tool to set a navigation target.  
- Nav2 will compute a path using the RTABMap-generated map and send velocity commands to the simulated robot.

---

## How It Works

### Modified XACRO Files

- Original XACRO and URDF files from Unitree’s ROS1 repository were updated to remove or replace incompatible Gazebo plugins.  
- The **Kinect** plugin (`libgazebo_ros_openni_kinect.so`) was replaced by a **depth camera** plugin (`libgazebo_ros_camera.so`).  
- Two incompatible ROS1 plugins for foot contact visualization (`libunitreeFootContactPlugin.so` and `libunitreeDrawForcePlugin.so`) were commented out.

### Camera Frames and Image Flipping

- Because camera frames differ from the robot’s coordinate system, a transform is added:
  - `camera_{name}` → `camera_optical_{name}`  
  - This aligns the camera’s coordinate frame with the ROS standard.  
- **image_flip_node.py** uses OpenCV to flip images 180° when needed.

### Differential Drive Workaround

- Since ROS2-compatible high-level control plugins for the Go1 are not yet available, an **invisible differential drive** plugin was added beneath the robot.  
- This lets you control the robot’s motion via teleoperation or Nav2 despite the quadruped’s legs not being actuated in the simulation.

### Map Creation and Navigation

- **RTABMap** provides a vSLAM solution, creating a map by processing depth data and performing loop closure.  
- **Nav2** converts RTABMap’s data into a **Costmap**, marking obstacles with higher cost.  
- A path is planned using Dijkstra’s or Hybrid A*, and local control is managed by the DWB (Dynamic Window Approach) successor.

---

## Future Work

- **Multi-Camera SLAM**: Integrate additional cameras (e.g., left and right side) to provide more sensor information.  
- **Native Quadruped Control**: Develop or integrate a **ROS2-compatible high-level motion controller** for Unitree Go1 to replace the differential drive hack.  
- **Plugin Updates**: Port the foot contact and force visualization plugins to ROS2 if they become available.

---


## Appendix: Project Structure

A typical folder layout after cloning and building might look like this:

```
unigo1/                   # Main package directory
├── launch/               # Contains launch files
│   ├── gazebolauncher.launch.py
│   ├── robstpub.launch.py
│   └── unigo1_rgbd.launch.py
├── meshes/               # Mesh files for the robot
│   ├── trunk.dae
│   ├── ...
├── urdf/                 # URDF / XACRO files
│   ├── go1.urdf.xacro
│   └── ...
├── scripts/              # Python scripts
│   └── image_flip_node.py
├── CMakeLists.txt
└── package.xml
```

**Questions or Contributions?**  
Feel free to open an issue or submit a pull request if you have improvements or questions about this repository. Happy simulating!
