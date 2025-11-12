# Introduction
This article primarily introduces how to install Intel® RealSense™ ROS2 on the AIB-NW01 Ubuntu 20.04 environment provided by Avalue Technology Inc.
These are packages for using Intel® RealSense™ cameras D400 and L500 series, SR300 camera and T265 Tracking Module with ROS2.

# Install GLU
```bash
sudo apt-get update
sudo apt-get install -y libgl1-mesa-dev libglu1-mesa-dev
```

# Install CUDA Toolkit
```bash
apt list 'cuda-toolkit*'
sudo apt-get install -y cuda-toolkit-11-4

echo 'export PATH=/usr/local/cuda/bin:$PATH' | sudo tee -a /etc/profile.d/cuda.sh
source /etc/profile.d/cuda.sh
export CUDACXX=/usr/local/cuda/bin/nvcc
nvcc --version
```
## Step 1: Install ROS2 foxy, on Ubuntu 20.04
If your target machine does not yet have ROS2 Foxy installed, please refer to the following documentation for installation instructions.
Reference - [ros2.foxy.aib-nw01](https://github.com/AlexChang633/ros2.foxy.aib-nw01 "ros2.foxy.aib-nw01")

## Step 2: Build librealsense2, Install Intel® RealSense™ ROS2 wrapper from Sources
```bash
sudo apt-get update
sudo apt-get install -y git cmake build-essential libssl-dev libusb-1.0-0-dev libgtk-3-dev libglfw3-dev pkg-config udev
# Intel Deprecated
# git clone https://github.com/IntelRealSense/realsense-ros.git -b foxy
git clone https://github.com/IntelRealSense/librealsense.git -b v2.50.0
cd librealsensecd
rm -rf build
mkdir build
cd build
# Jetson recommends starting with RSUSB (no kernel modules/patches required). To enable GPU acceleration, add CUDA. If CUDA is not installed, change it to OFF
cmake .. -DCMAKE_BUILD_TYPE=Release -DFORCE_RSUSB_BACKEND=ON -DBUILD_WITH_CUDA=ON
cmake --build . -j"$(nproc)"
sudo make install
sudo ldconfig

# Check
ls /usr/local/lib/cmake/realsense2/realsense2Config.cmake
sudo ldconfig
ldconfig -p | grep librealsense2

mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/IntelRealSense/realsense-ros.git
cd realsense-ros
# The version 4.0.4 supports Foxy and corresponds to libRS v2.50.0
git checkout 4.0.4
cd ~/ros2_ws
```

## Step 3: Install Dependencies
```bash
cd ~/ros2_ws

source /opt/ros/foxy/setup.bash

sudo apt-get update
sudo apt-get install -y python3-rosdep
sudo apt-get install -y ros-foxy-cv-bridge
sudo apt-get install -y ros-foxy-image-transport
sudo apt-get install -y ros-foxy-camera-info-manager
sudo apt-get install -y ros-foxy-diagnostic-updater
sudo apt-get install -y ros-foxy-xacro

sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -r -y --rosdistro foxy --skip-keys "librealsense2"
```

## Step 4: Build Install Intel® RealSense™ ROS2
```bash
# colcon build
colcon build --symlink-install --packages-select realsense2_description realsense2_camera realsense2_camera_msgs

# Create Intel® RealSense™ ROS2 UDEV Rule
# Reference - [99-realsense-libusb.rules] (https://raw.githubusercontent.com/IntelRealSense/librealsense/master/config/99-realsense-libusb.rules "99-realsense-libusb.rules")
sudo nano /etc/udev/rules.d/99-realsense-libusb.rules

# Reload UDEV Rule
sudo udevadm control --reload-rules && sudo udevadm trigger

# Check Intel RealSense
rs-enumerate-devices

# View Intel RealSense
realsense-viewer
```

## Setp 5: Configure ROS2 Foxy, Intel® RealSense™ ROS2 Environment
```bash
# Setup ROS2 Foxy Environment
source /opt/ros/foxy/setup.bash
# Setup Intel® RealSense™ ROS2 Environment
cd ~/ros2_ws
source install/setup.bash
```

# Usage
## Start Camera Node in ROS2
```bash
ros2 launch realsense2_camera rs_launch.py
```

## Start Camera Node in ROS2 with parameters specified in rs_launch.py, for example - pointcloud enabled
```bash
ros2 launch realsense2_camera rs_launch.py enable_pointcloud:=true device_type:=d435
```

## Start Camera Node in ROS2 without using the supplement launch files
```bash
ros2 run realsense2_camera realsense2_camera_node --ros-args -p filters:=colorizer
```

## Start Camera Node in RGB Topic
```bash
ros2 launch realsense2_camera rs_launch.py enable_rgbd:=true enable_sync:=true align_depth.enable:=true enable_color:=true enable_depth:=true 
```

# Optional Installation - ROS2 Image Transport Node
If you need to compress Intel® RealSense™ ROS2 Camera View, consider to install as follows packages.

```bash
sudo apt install ros-foxy-image-transport 
sudo apt install ros-foxy-image-transport-plugins

# Configure ROS2 Foxy Environment
source /opt/ros/foxy/setup.bash

# Launch Image Transport Node
ros2 run image_transport republish raw in:=/camera/color/image_raw compressed out:=/camera/color/image_raw/compressed
```

# Reference.
[ROS2 Wrapper for Intel® RealSense™ Devices](https://github.com/vanderbiltrobotics/realsense-ros "ROS2 Wrapper for Intel® RealSense™ Devices")
