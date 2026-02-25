# Perception stack for YOLOv9 + ROS2 cone detection (Camera only)

Camera-based cone detection pipeline for Formula Student Autonomous
using **YOLOv9 (ONNX)** integrated with **ROS2**.

This repository provides:

-   YOLOv9 training workflow
-   ONNX export pipeline
-   ROS2 integration
-   Real-time cone detection for FS environments

------------------------------------------------------------------------

# System Overview

The perception pipeline is structured as:

usb_cam (camera) ↓ YOLO ROS2 Node (image resize → ONNX inference) ↓
Detection messages ↓ RViz / downstream planning modules

The YOLO node resizes incoming frames to the model input size before
inference.

------------------------------------------------------------------------

# Requirements

## Hardware & Software
* **Framework:** ROS 2 (Humble/Iron/Jazzy)
* **Camera:** Luxonis OAK-D LR (DepthAI)
* **GPU:** NVIDIA (CUDA enabled)
* **Model:** Custom trained YOLOv9 (`.onnx` format for TensorRT acceleration)

Install YOLO:

``` bash
pip install ultralytics
```

Install ONNX Runtime:

``` bash
pip install onnx onnxruntime
```

------------------------------------------------------------------------

# Export to ONNX

Export the trained model to ONNX format:

``` bash
yolo export model=best.pt format=onnx imgsz=[640,640]
```

This produces a static 640×640 ONNX model.

------------------------------------------------------------------------

# ROS2 Setup

## Start Camera

**For webcam:**
``` bash
ros2 run usb_cam usb_cam_node_exe
```

**For Physical OAK-D Camera:**
```bash
ros2 launch depthai_ros_driver camera.launch.py
```

**For EUFS Simulator:**
Launch the simulator environment as per the EUFS documentation. The camera topic will typically be `/zed/left/image_rect_color`.

*Note: The EUFS Simulator and OAK-D publish images using **Best Effort (Reliability=2)** QoS. The YOLO node must match this, or the nodes will not communicate.*

```bash
ros2 launch yolo_bringup yolov9.launch.py \
  model:=/path/to/your/best.onnx \
  device:=cuda:0 \
  input_image_topic:=/oak/rgb/preview/image_raw \
  reliability:=2
```


**Always ensure `/image_raw` is publishing:**

``` bash
ros2 topic list
ros2 topic hz /image_raw
```

------------------------------------------------------------------------

## Launch YOLO Node

``` bash
ros2 launch yolo_bringup yolov9.launch.py   imgsz_height:=640   imgsz_width:=640  model:=/path/to/your/best.onnx device:=cuda:0
```

### Parameters

  Parameter        Description
  ---------------- ------------------------------
  `imgsz_height`   Model input height
  `imgsz_width`    Model input width
  `device`         CPU or CUDA usage (GPU first)
  `model`          weights file used
  `threshold`      Confidence threshold (adjustable if necessary)

------------------------------------------------------------------------

# Viewing in RViz

Launch RViz:

``` bash
rviz2
```

Add: - Image display (`/image_raw`) - Detection topic
(e.g. `/yolo/detections`)

Ensure the correct image topic is selected.

------------------------------------------------------------------------

# Configuration Notes

-   The ONNX model is exported with a fixed input size (640×640).
-   The ROS2 node resizes incoming images before inference.
-   Adjust `threshold` and `iou` to balance precision and recall.
-   Ensure camera resolution is compatible with model input settings.

------------------------------------------------------------------------

# Extending the Pipeline

The detection output can be integrated with:

-   LiDAR clustering
-   Sensor fusion modules
-   Path planning systems
-   Autonomous driving logic

The camera-only pipeline operates independently and can be used as a
standalone perception module.

------------------------------------------------------------------------

# Best Practices

-   Always export from `best.pt`
-   Keep a backup of trained weights
-   Verify ONNX model loads correctly before deployment
-   Confirm ROS topics are publishing before debugging inference

# Known Issues

### 1. Camera Permissions (udev rules)
If you get an `X_LINK_INSUFFICIENT_PERMISSIONS` error when launching the OAK-D, Linux is blocking USB access to the Movidius chip. Run this once:
` ` `bash
echo 'SUBSYSTEM=="usb", ATTRS{idVendor}=="03e7", MODE="0666"' | sudo tee /etc/udev/rules.d/80-movidius.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
` ` `
*(Unplug and re-plug the camera after running this).*

### 2. Python Dependencies (The NumPy Trap)
ROS 2 requires an older version of NumPy, but Ultralytics (YOLO) requires a newer one. To prevent `AttributeError` crashes, force this specific version range:
` ` `bash
pip3 install "numpy>=1.26.4,<2.0.0"
` ` `
