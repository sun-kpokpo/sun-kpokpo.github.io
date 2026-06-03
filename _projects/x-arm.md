---
layout: page
title: X-Arm
description: Autonomous pick-and-place with a DOFBOT Pi arm, YOLOv8, ArUco calibration, and vision-to-robot integration.
img: assets/img/projects/x-arm-cover.jpg
importance: 3
category: work
github: https://github.com/sun-kpokpo/X-Arm/
---

X-Arm is a team project from the AI Club of the Bénin Excellence-Fondation Vallet AI Lab. The goal is to build an autonomous pick-and-place system where a Yahboom DOFBOT Pi robotic arm detects colored objects, estimates their position in the robot workspace, and moves them to a target location.

The project brings together computer vision, camera-to-robot calibration, inverse kinematics, and robot control. It started as a club project around manipulation robotics: how can a low-cost educational arm perceive an object, convert that perception into usable coordinates, plan a motion, and execute a physical grasp?

## Robot Platform

The physical platform is the Yahboom DOFBOT Pi, a small educational robotic arm controlled from a Raspberry Pi. In this project, the arm is used as a low-cost manipulation platform for pick-and-place experiments with colored cubes.

{% include figure.liquid path="assets/img/projects/x-arm-dofbot.jpg" caption="Yahboom DOFBOT Pi used for the X-Arm pick-and-place setup." width="420" class="img-fluid rounded z-depth-1 mx-auto d-block" %}

## Perception

The perception module uses a custom YOLOv8n detector trained to recognize colored objects, mainly blue, green, and red cubes. The model was trained from a custom dataset and tested for real-time webcam detection.

In the current vision stack, YOLO first detects the target object and returns a bounding box. OpenCV and HSV segmentation are then used to refine the top-face centroid of the object inside the bounding box. This gives a more useful point for manipulation than the raw center of the whole detection box.

The current model results are:

- **mAP50:** 88.4%
- **mAP50-95:** 70.4%
- **Model:** YOLOv8n

{% include figure.liquid path="assets/img/projects/x-arm-detection.jpg" caption="YOLO-based colored object detection used by the perception module." width="520" class="img-fluid rounded z-depth-1 mx-auto d-block" %}

## Calibration

To make the camera output useful for the robot, the project uses ArUco markers and homography calibration. Four ArUco markers are placed in the workspace. Their pixel positions are detected with OpenCV, while their corresponding physical positions are recorded in the robot frame.

From these correspondences, OpenCV computes a homography matrix. Once YOLO and HSV provide the selected object's pixel centroid, the homography maps that point to robot-frame `x, y` coordinates. The `z` value is treated as fixed for the tabletop/object height, which makes the target usable by the control layer.

This calibration step is the bridge between "the camera sees a cube here" and "the robot should move to this physical position."

## Integration

The integrated test pipeline is split into two processes that communicate through ZMQ:

- `detector.py` runs the camera, YOLO detection, HSV centroid refinement, and homography mapping. It replies with the detected object's class and robot-frame coordinates.
- `arm_controller.py` requests detections, computes target joint angles, and sends commands to the DOFBOT Pi arm.
- `main.py` launches both processes and keeps the full test running.

```text
Camera Feed -> YOLOv8 + HSV -> Homography -> ZMQ -> IK + Motion Sequence -> DOFBOT Pi
```

## Control

The robot currently performs pick-and-place demonstrations: it detects an object, receives the mapped robot-frame coordinates, moves above the target, descends to pick it, lifts it, and places it at a predefined location.

For inverse kinematics, the current integrated version uses the Python `ikpy` library with a URDF model of the arm. This has allowed the team to connect the vision and actuation pipeline, but the returned IK solutions are not reliable enough for robust manipulation. The current work is therefore moving toward a custom inverse-kinematics solver better matched to the real DOFBOT Pi geometry and servo behavior.

## Demonstration

{% include video.liquid path="assets/video/x-arm-pick-and-place.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true caption="Current X-Arm pick-and-place demonstration with the DOFBOT Pi." %}

## Status

- Perception: YOLOv8n colored object detection, HSV centroid refinement, and webcam testing.
- Calibration: ArUco marker detection and homography mapping from image points to robot-frame coordinates.
- Integration: ZMQ-based connection between vision and robot control.
- Control: pick-and-place demonstration working, with custom IK solver development in progress.

## Code

The project repository is available on GitHub:

[sun-kpokpo/X-Arm](https://github.com/sun-kpokpo/X-Arm/)
