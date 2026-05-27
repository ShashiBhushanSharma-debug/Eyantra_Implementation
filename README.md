# Eyantra_Implementation
This repository onoly contain a readme.md file that show the efforts I did with my team in the eyantra and what were the observations that we made during this venture. This is a technical documentation of all the techonology used and there trade -off's that we encountered. Also, this contain the reason to why we were not able to be in top 5.

# Eyantra Warehouse Automation — Technical Documentation

> A retrospective and technical record of our team's implementation for the **e-Yantra Robotics Competition (eYRC)** under the **Warehouse Automation** theme.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [Implementation Details](#implementation-details)
   - [Multi-Robot Coordination](#multi-robot-coordination)
   - [Sorting & Pick-and-Place](#sorting--pick-and-place)
   - [Inventory Tracking & Localization](#inventory-tracking--localization)
5. [Trade-offs & Observations](#trade-offs--observations)
6. [Why We Did Not Finish in the Top 5](#why-we-did-not-finish-in-the-top-5)
7. [Team Learnings](#team-learnings)

---

## Project Overview

This repository serves as a technical documentation of our team's work during the **e-Yantra Robotics Competition**, focused on the **Warehouse Automation** theme. The objective was to design and deploy a multi-robot system capable of autonomous sorting, pick-and-place operations, and real-time inventory tracking inside a simulated warehouse environment.

No source code is hosted here. This document captures the architecture decisions, technologies evaluated, trade-offs encountered, and lessons learned throughout the competition.

---

## System Architecture

The overall system was divided into three tightly coupled subsystems:

```
┌─────────────────────────────────────────────────────────┐
│                     Central Controller                  │
│         (ROS Master + Task Allocation + MQTT Broker)    │
└───────────────────┬─────────────────────────────────────┘
                    │
        ┌───────────┴────────────┐
        │                        │
┌───────▼──────┐        ┌────────▼──────┐
│   Robot 1    │        │   Robot 2     │
│  (ESP32 +    │  MQTT  │  (ESP32 +     │
│   ROS Node)  │◄──────►│   ROS Node)   │
└──────┬───────┘        └───────┬───────┘
       │                        │
┌──────▼────────────────────────▼──────┐
│         Perception Layer             │
│  (OpenCV / YOLO — Camera Feeds)      │
└──────────────────────────────────────┘
```

- **Central Controller** manages task assignment and robot scheduling via ROS topics.
- **ESP32 microcontrollers** handle low-level motor control on each robot and communicate with the ROS ecosystem via MQTT.
- **Perception Layer** processes overhead or onboard camera feeds to detect objects, ArUco markers, and warehouse zones.
- **Path Planning** runs on the ROS layer using MPPI (Model Predictive Path Integral) as the primary controller, with PID loops for low-level motor correction.

---

## Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Middleware | ROS (Robot Operating System) | Inter-node communication, task orchestration |
| Microcontroller | ESP32 | On-robot low-level control |
| Communication | MQTT | Wireless messaging between ESP32 and ROS |
| Perception | OpenCV, YOLO | Object detection, ArUco marker tracking |
| Path Planning | MPPI | High-level trajectory optimization |
| Motor Control | PID | Low-level velocity/position correction |

---

## Implementation Details

### Multi-Robot Coordination

Task allocation was handled centrally through a ROS node that maintained a job queue. Each robot subscribed to a task topic and acknowledged task completion before the next instruction was dispatched. A simple priority-based scheduler was used to avoid collisions at shared waypoints.

**Challenge:** Synchronizing two robots in real time over MQTT introduced non-deterministic delays, making collision-free guarantees difficult to enforce without conservative wait buffers.

### Sorting & Pick-and-Place

Objects were classified by color and shape using a YOLO-based detection model running on an overhead camera feed. Detected object coordinates were transformed into the robot's frame using a calibrated homography matrix. The robot then navigated to the object, performed a pick operation, and deposited it in the designated zone.

**Challenge:** Detection confidence dropped under inconsistent lighting conditions and when objects were partially occluded, causing missed picks and incorrect sorting.

### Inventory Tracking & Localization

Localization was achieved using ArUco markers placed on the warehouse floor. OpenCV's ArUco module estimated the robot's pose relative to known marker positions. An inventory map was maintained in memory and updated after each successful pick-and-place cycle.

**Challenge:** Marker detection was sensitive to camera angle and motion blur, especially at higher robot speeds, degrading localization accuracy mid-task.

---

## Trade-offs & Observations

### 1. MPPI — Performance vs. Computational Cost

MPPI provided smooth, dynamically feasible trajectories and handled obstacle avoidance well in simulation. However, on the actual hardware, the computational overhead was significant. Running MPPI at the required frequency was not reliably achievable on the available onboard compute, leading to delayed control updates and jerky motion.

**Trade-off:** MPPI offers superior path quality but demands compute resources that were beyond our deployment hardware. A lighter planner (e.g., pure pursuit or DWA) would have been more practical for real-time execution.

### 2. MQTT Communication Latency

MQTT worked well for low-frequency command exchange but introduced variable latency (occasionally 80–200ms spikes) in time-critical coordination scenarios. This made tight multi-robot synchronization unreliable.

**Trade-off:** MQTT is easy to integrate with ESP32 but is not suited for hard real-time robotics communication. ROS 2's DDS-based transport or a direct serial/UDP link would have been more deterministic.

### 3. Vision Reliability

The YOLO model performed well in controlled lighting but was brittle in the competition environment. We did not have sufficient time to retrain or fine-tune the model on domain-specific data, which hurt detection precision.

**Trade-off:** Pretrained YOLO models offer fast integration but require domain-specific fine-tuning to be robust in novel environments.

### 4. Subsystem Integration

Each subsystem (perception, planning, communication, control) was developed somewhat independently and integrated late in the timeline. This revealed interface mismatches and timing issues that consumed significant debugging time close to the final evaluation.

**Trade-off:** Parallel development accelerates individual progress but increases integration risk if interface contracts are not strictly defined upfront.

---

## Why We Did Not Finish in the Top 5

The following factors collectively contributed to our ranking:

- **MPPI computation bottleneck** caused slow and inconsistent robot motion during the live evaluation, directly affecting task completion time.
- **Vision failures** during the evaluation led to missed picks, which penalized our score significantly.
- **MQTT latency spikes** caused desynchronization between robots on two occasions, resulting in a near-collision recovery that cost time.
- **Late integration** meant we had limited time to stress-test the full pipeline end-to-end under evaluation conditions. Edge cases that appeared only under system-wide load were not adequately addressed.

Despite these setbacks, the core architecture was sound and all individual subsystems functioned correctly in isolation.

---

## Team Learnings

- **Define subsystem interfaces early.** A shared message contract (ROS topic schemas, coordinate frame conventions) from day one would have drastically reduced integration friction.
- **Benchmark your planner on target hardware, not just simulation.** MPPI's real-time feasibility on constrained hardware should have been validated before committing to it.
- **Invest in domain adaptation for vision models.** Even a small fine-tuning dataset collected in the actual environment would have meaningfully improved detection robustness.
- **Use deterministic communication where timing matters.** MQTT is appropriate for telemetry; it is not appropriate for synchronized multi-robot control loops.
- **End-to-end testing must happen early and often.** Full-pipeline stress tests should be a regular part of the development cycle, not a final-week activity.

---

*This documentation was written post-competition as a retrospective record for future reference and learning.*