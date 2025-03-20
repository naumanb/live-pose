
# FoundationPose / LivePose Setup Guide

This comprehensive guide will walk you through setting up and running FoundationPose, a package for live 6 DOF pose estimation using Intel RealSense depth cameras.

## Prerequisites

- Windows 10/11 with WSL2 installed
- Ubuntu 24.04 in WSL2
- Intel RealSense D435 camera (or compatible model)
- NVIDIA GPU (for optimal performance)


## Setting Up WSL2 with RealSense Camera Support

### 1. Update WSL2 Kernel

Ensure you have the latest WSL2 kernel for proper camera support:

```powershell
# In Windows PowerShell (as Administrator)
wsl --update
# For latest pre-release kernel (recommended)
wsl --update --pre-release
# Restart WSL
wsl --shutdown
```

Verify your kernel version (should be 6.6.x or newer):

```bash
uname -r
```


### 2. Connect RealSense Camera to WSL2

```powershell
# In Windows PowerShell (as Administrator)
# List USB devices
usbipd list
# Find your RealSense camera and note its bus ID
usbipd attach --busid <BUSID> --wsl
```


### 3. Install RealSense SDK in WSL2

```bash
# Install dependencies
sudo apt update
sudo apt install -y libssl-dev libusb-1.0-0-dev libudev-dev pkg-config libgtk-3-dev
sudo apt install -y git cmake build-essential

# Clone and build librealsense
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense
mkdir build && cd build

# If Git ownership issues occur, run:
git config --global --add safe.directory '*'

# Build with RSUSB backend (recommended for WSL2)
cmake ../ -DBUILD_EXAMPLES=true -DCMAKE_BUILD_TYPE=Release -DFORCE_RSUSB_BACKEND=TRUE
make -j$(nproc)
sudo make install

# Set up udev rules
sudo mkdir -p /etc/udev/rules.d/
wget -O ~/99-realsense-libusb.rules https://raw.githubusercontent.com/IntelRealSense/librealsense/master/config/99-realsense-libusb.rules
sudo cp ~/99-realsense-libusb.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger
```

Verify camera detection:

```bash
rs-enumerate-devices
```


## Setting Up FoundationPose

### 1. Clone the Repository

```bash
git clone https://github.com/Kaivalya192/live-pose.git
cd live-pose
```


### 2. Build Docker Container

```bash
cd docker
docker build --network host -t foundationpose .
cd ..
```


### 3. Download Model Weights

Download the required model weights (links not provided in the README) and place them in:

```
live-pose/FoundationPose/weights
```


## Installing RealSense SDK in Docker Container

After starting your Docker container (but before running the application), you need to install the RealSense SDK inside the container:

```bash
# Inside the Docker container
apt update
apt install -y libssl-dev libusb-1.0-0-dev libudev-dev pkg-config libgtk-3-dev
apt install -y git cmake build-essential

# Clone and build librealsense
git clone https://github.com/IntelRealSense/librealsense.git
cd librealsense
mkdir build && cd build

# If Git ownership issues occur, run:
git config --global --add safe.directory '*'

# Build with RSUSB backend
cmake ../ -DBUILD_EXAMPLES=true -DCMAKE_BUILD_TYPE=Release -DFORCE_RSUSB_BACKEND=TRUE
make -j$(nproc)
make install

# Create udev rules directory and install rules
mkdir -p /etc/udev/rules.d/
wget -O ~/99-realsense-libusb.rules https://raw.githubusercontent.com/IntelRealSense/librealsense/master/config/99-realsense-libusb.rules
cp ~/99-realsense-libusb.rules /etc/udev/rules.d/
```

Verify camera detection inside the container:

```bash
rs-enumerate-devices
```


## Running FoundationPose

### 1. Start the Docker Container

```bash
# For Linux/WSL2
bash docker/run_container.sh

# For Windows (using Cygwin)
./docker/run_container_win.sh
```

The container is configured with:

- GPU access
- USB device passthrough
- X11 forwarding for GUI
- Host network access


### 2. Build the Package Inside Container

Once inside the container:

```bash
CMAKE_PREFIX_PATH=$CONDA_PREFIX/lib/python3.9/site-packages/pybind11/share/cmake/pybind11 bash build.bash
```


### 3. Run Live Pose Estimation

```bash
bash run_live.sh
```


## Using with Custom Objects

1. Locate or create an .obj file for your target object
2. For novel objects, use the Object Reconstruction Framework (details not provided in README)
3. When running the application, select boundary points of the object in the first frame for masking

## Troubleshooting

### Camera Not Detected

- Ensure the camera is properly attached to WSL2 before starting the Docker container
- Verify the camera works in WSL2 with `rs-enumerate-devices` before trying in Docker
- If the camera disconnects, you'll need to reattach it to WSL2 and restart the container


### Permission Issues

- You may need to run commands with sudo in WSL2
- In Docker, you're likely already running as root, so sudo isn't necessary


### Git Ownership Errors

- If you encounter "dubious ownership" errors with Git, use:

```bash
git config --global --add safe.directory '*'
```


### X11 Display Issues

- Ensure xhost is properly configured with `xhost +` before starting the container
- Check that DISPLAY environment variable is correctly set


## Notes for Windows Users

Windows users should install Cygwin to run the shell scripts properly. The Windows-specific scripts are provided in the repository.

## System Requirements

- NVIDIA GPU with CUDA support
- At least 8GB RAM
- Intel RealSense D435 or compatible depth camera


## Important Notes

- Installing the RealSense SDK in both WSL2 and Docker is necessary for proper functionality
- The camera must be attached to WSL2 before starting the Docker container
- If you disconnect and reconnect the camera, you'll need to reattach it to WSL2 using usbipd