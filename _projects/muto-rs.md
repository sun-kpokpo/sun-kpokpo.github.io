---
layout: page
title: MUTO-RS
description: Sim-to-Real RL Locomotion for an 18-DOF Hexapod
img: assets/img/projects/muto_thumbnail1.jpg
importance: 1
category: work
github: https://github.com/dirennoukpo/MUTO_RL/tree/sun/rl_training
---

## How It Started

At Arduino Days 2026, Diren presented something genuinely impressive: a multi-robot choreography system where six MUTO hexapods danced
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




## Fixing the Digital Twin
The goal was to make sure the 
simulation actually reflects the real robot. And it turned out there 
was a problem hiding in plain sight.

One of the legs — the middle-left tibia, `yz3_Link` — was visually 
disconnected from the rest of the robot. Not in a subtle way either. 
The mesh was just floating somewhere else while the joint moved 
correctly underneath it.

The machine running Isaac Sim wasn't accessible at the time, so rather 
than waiting, we took the opportunity to do something useful: verify 
whether the problem was coming from Isaac Sim itself, or from the URDF. 
Since MUTO-RS lives in the ROS2 ecosystem, we loaded the URDF directly 
in RViz on a separate machine. If the problem showed up there too, it 
was a URDF problem — full stop, nothing to do with Isaac Sim.

It showed up there too.

{% include figure.liquid path="assets/img/projects/yz3_floating_rviz.png"
   caption="Same problem in RViz: the TF frame sits correctly on the robot, but the mesh floats elsewhere" 
   width="800" class="img-fluid rounded z-depth-1 mx-auto d-block" %}

We ran `tf2_echo yz2_Link yz3_Link` to measure the transform between 
the parent joint and the child link. The translation came back as 
`[0.196, -0.025, 0.016]` — and that `0.196` in X was immediately 
suspicious. Looking at the URDF, `yz3_Joint` had its origin set to 
`xyz="0.195684..."`, placing the joint nearly 20 cm away from where 
it should be. Every other tibia in the robot had a sensible origin. 
This one had clearly been exported from SolidWorks with a wrong 
reference point.

We fixed the joint origin. The frame snapped to the right place. But 
the mesh was still offset — because the STL file itself had been 
exported with its geometric origin not at the joint frame. Every other 
`*3_Link` in the robot has `visual origin xyz="0 0 0"`, meaning the 
mesh origin coincides with the joint frame by construction. `yz3_Link.STL` 
didn't — its origin was sitting roughly 20 cm away inside the file. 
We compensated directly in the URDF visual tag:

```xml
<visual>
  <origin xyz="0.1965 0.052 0.0" rpy="0 0 0" />
  ...
</visual>
```

That attached the mesh to the robot. But when we moved the joint with 
the sliders, it was rotating obliquely — clearly wrong. The issue was 
in the rotation axis.

In a URDF, `<axis xyz="a b c" />` defines the direction around which 
the joint rotates — literally a vector in 3D space. `yz3_Joint` had 
inherited its axis from `yq3_Joint` (the front-left tibia): 
`xyz="0.711 0.702 0"`, a diagonal vector at ~45° in the XY plane. 
That makes sense for `yq3` because that leg attaches to the chassis 
at an angle. But `yz3` is the middle-left leg — it attaches 
perpendicularly. Its tibia should rotate around a pure X axis, just 
like its symmetric counterpart `zz3_Joint` on the right side.

We changed the axis to `xyz="-1 0 0"`. The negative sign matters: 
`yz3` is on the left side of the robot (negative Y in the base frame), 
so its X axis points in the opposite direction relative to `zz3`. 
Think of it as two legs facing each other — the same physical rotation 
looks like opposite signs depending on which side you're standing on. 
As a consequence, the joint limits had to be flipped too: 
`lower="-0.7" upper="1.57"` instead of `lower="-1.57" upper="0.7"`.

<div class="text-center">
{% include video.liquid path="assets/video/yz3_fixed_rviz.webm" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true %}
<div class="caption">
    All six legs correctly attached and rotating after URDF corrections.
</div>
</div>

Three fixes in the URDF, one properly behaved digital twin. We can 
now move on to training with confidence that what happens in simulation 
is at least geometrically honest about the real robot.



## Where We Are

The URDF is now clean. Both Gazebo and Isaac Sim confirm it — all six
legs attached, all joints responding correctly.

<div class="text-center">
{% include video.liquid path="assets/video/test_gz.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true %}
<div class="caption">
    MUTO-RS after URDF fix — Gazebo.
</div>
</div>

<div class="text-center">
{% include video.liquid path="assets/video/test_is.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true %}
<div class="caption">
    MUTO-RS after URDF fix — Isaac Sim. One remaining issue visible:
    yz3 (middle-left tibia) responds in the opposite direction compared
    to the other legs — a known consequence of the axis correction, 
    addressed below.
</div>
</div>

There is one remaining issue. Because we corrected `yz3_Joint`'s
rotation axis from `xyz="0.711 0.702 0"` to `xyz="-1 0 0"`, the
sign convention for that joint is now inverted relative to the other
five tibias. A positive command makes it go the wrong way. The joint
limits were flipped accordingly (`lower="-0.7" upper="1.57"` instead
of `lower="-1.57" upper="0.7"`), so the range of motion is physically
correct — but the direction is a mirror of what the policy would expect.

The fix we're going with is a sign wrapper in `muto_rl_env_cfg.py`.
Rather than hoping the policy figures out on its own that this one joint
is different, we intercept the signal at the observation/action boundary:
joint position, velocity, and torque for `yz3` get multiplied by `-1`
before the policy ever sees them, and again when the action goes back
out. From the policy's perspective, all eighteen joints speak the same
language. The wrapper handles the translation silently, and the whole
thing is documented so no one trips over it later.

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
