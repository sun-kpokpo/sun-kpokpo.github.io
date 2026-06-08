---
layout: page
title: TicTacToeRL-RobotArm
description: Three Men's Morris with a DOFBOT arm, Q-learning, computer vision, and real-world manipulation.
img: assets/img/projects/robotic-arm.png
importance: 2
category: work
github: https://github.com/sun-kpokpo/TicTacToeRL-RobotArm/tree/feature/decision-qlearning
---

TicTacToeRL-RobotArm is a final project my team worked at the end of the [École d'Été sur l'Intelligence Artificielle (EEIA)](https://eeia.bj/) 2025. We built a robotic game-playing pipeline where a Yahboom DOFBOT Pi arm perceives a Three Men's Morris board, decides a move with reinforcement learning, and physically plays against a human opponent.

During the project realization, my main contribution was on the Q-learning decision module. Since then, we have continued developing the project at the Bénin Excellence AI Lab, especially around perception-to-action integration, robot calibration, and the transition from hand-coded arm positions toward inverse kinematics.

## The Game

Three Men's Morris is an abstract strategy game played on a 3-by-3 grid of nine points. It looks close to tic-tac-toe, but it has an important second phase.

In the placement phase, each player places three pieces on empty points, trying to form three in a row. If no one wins after all six pieces have been placed, the game enters a movement phase: players move one of their pieces to an adjacent unoccupied point along the board lines until someone forms a mill. This movement phase makes the game more dynamic than classical tic-tac-toe and gives the agent a richer decision problem than simply choosing an empty square once.

## Robot Platform

The physical platform is the Yahboom DOFBOT Pi, a small educational 6-DOF robotic arm driven by serial bus servos. In this project, the arm is used as a low-cost manipulation platform: it picks or places game pieces on a physical board using vision-derived target coordinates.


Early execution was based on hard-coded joint angles for each board position. That worked for controlled demonstrations, but it does not scale well when the board pose changes or when calibration drifts. The current direction is to move toward inverse kinematics so the robot can execute moves from target coordinates rather than fixed angle scripts.

## Perception

The perception stack has evolved from direct board-state detection toward a more geometric calibration pipeline. At the current stage, we use OpenCV ArUco markers to calibrate the board plane. I recorded the physical positions of the four ArUco markers in the robot frame, with a fixed `z` height for the board plane. OpenCV detects the same markers in the camera frame, and these corresponding points are used to compute a homography.

Once YOLO detects the pawns in the camera frame and the Q-learning agent selects a move, the homography maps the selected board target into robot-frame `x, y` coordinates. The fixed `z` coordinate is then added before passing the target to the execution layer. This matters because the robot does not only need to know _which_ cell is occupied. It also needs a metric target location that the control module can use to move the arm in the physical workspace.

{% include figure.liquid path="assets/img/projects/robotic-arm_visuel.png" caption="YOLO-based detection." width="450" class="img-fluid rounded z-depth-1 mx-auto d-block" %}

## Decision

The decision module uses Q-learning trained in simulation. The agent learns a policy over board states and actions, balancing exploration and exploitation during training. In the original pipeline, the Q-learning agent was trained over 400,000 episodes and used to select the next move from the perceived board state.

This module was my main focus, and it was what sparked my interest in reinforcement learning for embodied systems.

More recently, we also experimented with Minimax as an exact game-theoretic baseline. Because Three Men's Morris has a small enough state space to search exhaustively, Minimax behaves as a perfect player and gives us a mathematical reference point for evaluating learned or approximate agents. I will add more detail on this comparison as the project evolves.

## Control

The control module transforms a selected move into a physical robot action. In the first working demonstrations, target positions were executed with hard-coded joint angles for each pawn spot. This gave us a practical way to validate the full AI-to-robot loop, but it also exposed the limits of fixed scripts: placement errors, occupied-square conflicts, and sensitivity to board alignment.

The current work is moving toward an inverse-kinematics-based execution layer, using the `x, y` coordinates produced by the ArUco/homography pipeline and a fixed `z` coordinate for the board plane.

## Architecture

```text
Camera Feed -> Perception -> Board State -> Decision -> Robot-frame Target -> Control -> Robot Arm
              YOLOv8 + ArUco/H             Q-learning             IK / DOFBOT Pi
```

| Module | Folder | Role |
| --- | --- | --- |
| Perception | `perception/` | YOLO pawn detection, ArUco marker detection, homography calibration, and board-state estimation |
| Decision | `decision/` | Q-learning agent trained in simulation |
| Control | `control/` | DOFBOT Pi execution, calibration, hard-coded angle scripts, and inverse-kinematics work |

## Key Results

- Built an end-to-end prototype connecting perception, Q-learning, and physical arm execution.
- Trained a Q-learning agent over 400,000 simulated episodes.
- Reached 88% board detection accuracy with YOLOv8 in earlier perception experiments under variable lighting.
- Demonstrated physical gameplay with a DOFBOT arm against a human opponent.
- Identified the need for more robust robot-frame calibration, grasp-state feedback, and inverse-kinematics-based execution.

## Demonstrations

The project has been presented in several local AI and robotics settings:

- Final project, EEIA 2025.
- Live demo, Benin Workshop on AI (BWAI) 2025.
- Poster, Deep Learning IndabaX Benin 2025.
- Hands-on demo, TEKBOT Robotics Challenge 2025.

{% include video.liquid path="assets/video/tictactoe-rl-robotarm-demo.mp4" class="img-fluid rounded z-depth-1 mx-auto d-block" controls=true caption="DOFBOT Pi playing Three Men's Morris against a human opponent." %}

## Workshop Presentation

This work was presented at the Benin Workshop on AI as a project synthesis, slide presentation, and live demonstration.

_Hounsinou, Fangnon, Kochoni, Kpokpo. "Intégration de l'apprentissage par renforcement et de la vision par ordinateur pour le jeu de Morpion avec un bras robotique."_

## Status

- Perception: YOLO pawn detection plus OpenCV ArUco markers and homography-based board-plane calibration.
- Decision: Q-learning agent trained and working.
- Control: hard-coded board positions validated; inverse kinematics integration in progress.
- Integration: full pipeline refinement ongoing in feature branches.

## Code

The repository is available on GitHub:

[sun-kpokpo/TicTacToeRL-RobotArm](https://github.com/sun-kpokpo/TicTacToeRL-RobotArm/tree/feature/decision-qlearning)
