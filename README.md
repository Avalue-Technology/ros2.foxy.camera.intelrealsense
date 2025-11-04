# Introduction
This article primarily introduces how to install ROS2 Foxy and Intel® RealSense™ ROS2 on the AIB-NW01 Ubuntu 20.04 environment provided by Avalue Technology Inc.

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

# Fix SSH Issue
```bash
sudo dpkg-reconfigure openssh-server

sudo mkdir -p /etc/ssh
sudo ssh-keygen -A

sudo chown -R root:root /etc/ssh
sudo chmod 755 /etc/ssh
sudo chmod 600 /etc/ssh/ssh_host_*_key
sudo chmod 644 /etc/ssh/ssh_host_*_key.pub

sudo mkdir -p /run/sshd
sudo chown root:root /run/sshd
sudo chmod 755 /run/sshd

sudo /usr/sbin/sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager

sudo dpkg --configure -a
sudo apt-get -f install
```
## Step 1: Install the ROS2 distribution (Install ROS2 foxy - Ubuntu 20.04)
```bash
ROS_DISTRO=foxy
sudo apt update && sudo apt install curl gnupg2 lsb-release
curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
sudo sh -c 'echo "deb [arch=$(dpkg --print-architecture)] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" > /etc/apt/sources.list.d/ros2-latest.list'
sudo apt update
sudo apt install ros-$ROS_DISTRO-ros-base

# Environment setup
echo "source /opt/ros/$ROS_DISTRO/setup.bash" >> ~/.bashrc
source ~/.bashrc

sudo apt install python3-colcon-common-extensions -y

sudo apt install python3-colcon-common-extensions -y

# Install argcomplete (optional)
sudo apt install python3-argcomplete
```

## Step 2: Install librealsense2
```bash
# Install the latest Intel® RealSense™ SDK 2.0
sudo mkdir -p /etc/apt/keyrings
curl -sSf https://librealsense.intel.com/Debian/librealsense.pgp | sudo tee /etc/apt/keyrings/librealsense.pgp > /dev/null

sudo apt-get install apt-transport-https

echo "deb [signed-by=/etc/apt/keyrings/librealsense.pgp] https://librealsense.intel.com/Debian/apt-repo `lsb_release -cs` main" | \
sudo tee /etc/apt/sources.list.d/librealsense.list
sudo apt-get update

sudo apt-get install librealsense2-dkms
sudo apt-get install librealsense2-utils

sudo apt-get install librealsense2-dev
sudo apt-get install librealsense2-dbg

sudo apt-get update
sudo apt-get upgrade
```

## Step 3: Install Intel® RealSense™ ROS2 wrapper from Sources
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
cd ~/ros2_ws/src
git clone https://github.com/IntelRealSense/realsense-ros.git
cd realsense-ros
# The version 4.0.4 supports Foxy and corresponds to libRS v2.50.0
git checkout 4.0.4
cd ~/ros2_ws
```

## Step 4: Install Dependencies
```bash
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
# rosdep install -i --from-path src --rosdistro $ROS_DISTRO -y
# rosdep install --from-path src --ignore-src -y
rosdep install --from-paths src --ignore-src -r -y --rosdistro foxy --skip-keys "librealsense2"
```

## Step 5: Build Install Intel® RealSense™ ROS2
```bash
# colcon build
colcon build --symlink-install --packages-select realsense2_description realsense2_camera realsense2_camera_msgs

# Step 6: Source (on each new terminal)
source install/setup.bash

# Verify Intel RealSense
realsense-viewer

# Launch Intel RealSense
source /opt/ros/foxy/setup.bash
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

# Launch Image Transport  Node
ros2 run image_transport republish raw in:=/camera/color/image_raw compressed out:=/camera/color/image_raw/compressed
```

# Reference.
[ROS2 Wrapper for Intel® RealSense™ Devices](https://github.com/vanderbiltrobotics/realsense-ros "ROS2 Wrapper for Intel® RealSense™ Devices")