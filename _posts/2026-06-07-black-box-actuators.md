---
layout: post
title: "Black-box actuators in sim-to-real RL: the hidden constraint"
date: 2026-06-07
description: "Most sim-to-real RL tutorials assume you can read torque, current, or at least velocity from your actuators. What happens when you can't?"
tags: [robotics, sim-to-real, reinforcement-learning, locomotion, hexapod]
categories: research
related_posts: false
---

When people talk about the sim-to-real gap in robot learning, the usual suspects come up: inaccurate dynamics models, sensor noise, unmodeled contacts, latency. These are real problems, and there is good work addressing each of them.

But there is a constraint that gets much less attention, one that does not show up in benchmark papers because benchmark papers use well-instrumented hardware. What happens when your actuators are black boxes?

## The problem

Most sim-to-real RL pipelines, implicitly or explicitly, assume something about actuator feedback. Not necessarily full torque sensing but at least current feedback, or velocity estimates, or a model of how the actuator responds to a given command. Frameworks like Isaac Lab expose joint velocities and efforts as standard observations. Papers on legged locomotion routinely include joint torques in the observation vector.

Now suppose you are working with position-controlled servos that expose exactly one thing: the commanded position you sent them. No torque. No current. No velocity. No internal PID state. You send a setpoint; the actuator does something; you have no direct window into what.

This is the situation with the YB-SD35M servos on MUTO-RS, our 18-DOF hexapod. The internal PID is fixed and inaccessible. The actuator is a black box in the strict sense. Inputs go in, the physical world responds, and the feedback channel is silent.

## Why it matters for sim-to-real

In simulation, this is easy to paper over. Isaac Lab will faithfully simulate whatever actuator model you give it. The danger is giving it a model that is too clean, one where the simulated joint tracks commands perfectly, with no lag, no velocity-dependent stiffness, no load-dependent droop.

On real hardware, none of that is true. The gap between what you commanded and what the joint actually did is invisible to your policy unless you explicitly measure and model it. And if your simulated actuator is too ideal, your policy will learn to exploit that ideality, issuing commands that only work in a world where servos are perfect springs.

This is not a new insight. Actuator modeling is a known component of sim-to-real transfer. What is less discussed is what to do when you cannot identify the actuator from the inside, when you have no access to current draw, winding resistance, or back-EMF, and must infer everything from external observation.

## What we are doing about it

MUTO-RS is an ongoing project. We are currently working through a pre-training checklist that includes actuator system identification from external joint position measurements, mass and inertia verification, and action delay characterization. The goal is to build the most faithful actuator model we can from the outside, then close the remaining gap with domain randomization during training.

Whether that is enough to produce a walking policy on Jetson Nano hardware within a 50 Hz control loop, with TensorRT inference, is an open question. We will find out.

More on this as the project develops.
