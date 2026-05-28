---
layout: page
title: MUTO-RS
description: Sim-to-Real RL Locomotion for an 18-DOF Hexapod
img: assets/img/projects/muto_thumbnail1.jpg
importance: 1
category: work
---

## How It Started

At Arduino Days 2025, Diren presented something genuinely impressive: a multi-robot choreography system where six MUTO hexapods danced
in synchronized formation. Beat-detected from music, coordinated in
real-time over ROS2 with a leader-follower architecture
([see the project](https://github.com/dirennoukpo/MUTO-RS-CHOREGRAGPHY)).

Watching that, I asked him: _what if the robot just...
learned to do that on its own? without anyone writing the
choreography?_

That single question became the whole MUTO-RS project.

## The Robot

MUTO-RS is a Yahboom hexapod with six legs and three joints per leg,
giving it 18 degrees of freedom in total. The frame is machined aluminum,
driven by 35 kg/cm serial bus servos and powered by a 9900 mAh battery,
with an onboard IMU for orientation sensing. It runs on a Jetson Nano
under ROS2.

{% include figure.liquid path="assets/img/projects/muto_thumbnail.jpg"
   caption="MUTO-RS" width="400" class="img-fluid rounded z-depth-1 mx-auto d-block" %}

## The Hard Part

The servos — Yahboom YB-SD35M — have a fixed internal PID baked into
the firmware. Stiffness, damping coefficients — none of it is documented.
On top of that: servo bus reads at 50 Hz, IMU at 20 Hz. So we match these
frequencies in simulation so training and inference stay synchronized.
But accurately modeling the PID in Isaac Lab is the core
of our sim-to-real gap. Domain randomization is probably part of the
answer, but we'll see.

## What We Tried First

We started with push recovery, our first time touching Isaac Lab.
After roughly 15 hours of training on an RTX A6000 with 4096 parallel
environments, the result was memorable: robot pinned to the ground,
legs moving in every direction. Very Orochimaru energy.😅

<div class="text-center">
{% include video.liquid path="assets/video/muto-push-recovery.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true %}
<div class="caption">
    Push recovery attempt: 15h of training, Orochimaru edition.🥲
</div>
</div>

In hindsight, we suspect the issue was a fixed base link during the
URDF import in Isaac Sim, a low-key setup mistake that would have corrupted
the dynamics from the start. Before launching any new training run,
we're taking the time to properly set up the digital twin and make sure
no problem comes from the simulation itself.

<div class="text-center">
{% include video.liquid path="assets/video/muto-spawning&setup.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true %}
<div class="caption">
    MUTO-RS spawning and setup in IsaacSim
</div>
</div>

## Where We Are

We are now preparing the locomotion training. The setup:

- **Simulator**: Isaac Lab (Isaac Sim)
- **Algorithm**: PPO via SKRL, 4096 parallel environments, RTX A6000
- **Observations**: 70 dimensions: base orientation, angular and linear
  velocity, projected gravity, joint positions and velocities, last actions
- **Hardware target**: Yahboom MUTO-RS, Jetson Nano, ROS2

## The Goal

Walk one meter. Stable forward locomotion before July 2026.

If that works, the next question is obvious — can we train it to dance?
A learned dance policy would be a natural follow-up. Designing a reward
function for "dancing well" though... that's a problem for future us.

## Team

- **Sunday B. KPOKPO**: simulation, training pipeline
- **Edwin D. NOUKPO**: hardware deployment, ROS2

## Code

Active development. Check the branches on
[GitHub](https://github.com/dirennoukpo/MUTO_RL/).
