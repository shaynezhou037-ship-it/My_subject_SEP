# Weekly Progress Log

## Week 0 | 2026-06-14

### Progress
- Created GitHub repository.
- Read the FURP handbook and confirmed repository structure.
- Started project background research.

### Challenges
- Still need to clarify the exact research direction and technical requirements.
- Not fully familiar with GitHub workflow yet.

### Next Steps
- Confirm project scope with supervisor.
- Set up the `/docs` and `/src` folders.
- Begin collecting papers or reference materials.

### Meeting Notes / Evidence
- Weekly meeting attended: Yes
- Key takeaway: Need to update GitHub every week.

## Week 1 | 2026-06-21

### Progress
- Reviewed and analyzed the GitHub project structure related to the research task.
- Set up the basic ROS 2 working environment and learned how to run the project through terminal commands.
- Successfully launched the required ROS 2 / robot simulation-related demo after troubleshooting environment and launch errors.
- Learned the basic workflow of using multiple terminals for ROS 2 tasks, including running one process for the system/simulation and another for commands, nodes, or monitoring.
- Gained an initial understanding of several core ROS 2 concepts, including workspace, package, node, launch file, topic, and terminal environment setup.

### Challenges
- At the beginning, I was not familiar with the ROS 2 workflow and did not understand why multiple terminals were needed.
- Encountered several errors during the launch process, including inactive/error states and terminal processes getting stuck.
- Some technical concepts, such as node, launch, workspace, and UR/robot-related packages, were new to me and required step-by-step clarification.
- Still need more practice to understand the relationship between GitHub source code, ROS 2 packages, and the actual running simulation.

### Next Steps
- Continue reviewing the GitHub repository and summarize the function of each main folder and file.
- Learn the basic ROS 2 concepts more systematically, especially nodes, topics, services, launch files, and package structure.
- Try to reproduce the successful launch process independently and record the key commands used.
- Start building a simple technical note or README to document installation steps, common errors, and solutions.
- Clarify how this GitHub project connects to the broader research direction and future experiment tasks.

This week's progress marks a critical transition from preliminary software simulation to a rigorous systemic analysis of the ROS2, MoveIt, and UR5 architecture, focusing on state feedback mechanisms, kinematic solver configurations, collision-aware planning, and methodological preparation for real-world hardware deployment.

## Week 2 | 2026-06-28
## 1. Overview

In Week 2, the primary objective shifted toward understanding the internal execution and data pipelines of the ROS2 / MoveIt / UR5 system based on ROS2 Humble. Core engineering tasks included developing custom joint state readers, executing programmatic pose goal planning via `pymoveit2`, integrating environmental collision parameters, and mapping the theoretical and practical relationships between system layers (command, planning, execution, and feedback).

## 2. Work Completed

### 2.1 Development of UR5 State Reading Node
A custom ROS2 Python package (`ur5_state_reader`) was engineered to monitor and validate robot joint states.
*   **Implementation:** Developed a node subscribing to the `/joint_states` topic to continuously output the 6-DOF joint positions.
*   **Tracked Joints:** `shoulder_pan_joint`, `shoulder_lift_joint`, `elbow_joint`, `wrist_1_joint`, `wrist_2_joint`, `wrist_3_joint`.
*   **Theoretical Context:** This confirmed that `/joint_states` serves as the primary joint-space feedback mechanism, broadcasting nomenclature, positions, velocities, and efforts essential for downstream processing by MoveIt, RViz, and `robot_state_publisher`.

### 2.2 Kinematic Configuration Validation
A comprehensive audit of the UR5 MoveIt configuration was conducted to verify operational parameters.
*   **Key Parameters Confirmed:**
    *   **Planning Group:** `ur_manipulator`
    *   **Base Frame:** `base_link`
    *   **End-Effector (Tip) Frame:** `tool0`
    *   **IK Solver:** `kdl_kinematics_plugin/KDLKinematicsPlugin` (a numerical Jacobian-based inverse kinematics solver).
    *   **Configuration File:** `kinematics.yaml`
*   **Action Taken:** Isolated and installed `kinematics.yaml` via `setup.py` within the custom package to control and analyze the solver's behavior. 
*   **Observation:** MoveIt relies heavily on this IK solver to translate Cartesian spatial targets for `tool0` into actionable, valid joint configurations for the UR5 manipulator.

### 2.3 Pose Goal Planning via `pymoveit2`
Tested automated pose goal generation and execution to evaluate spatial commanding logic.
*   **Methodology:** Read current `base_link` → `tool0` transforms, calculated a minor coordinate offset, and published the target pose to MoveIt.
*   **Anatomy of a Pose Goal:**
    *   Reference Frame (`base_link`) & Target Frame (`tool0`).
    *   Cartesian Position ($x, y, z$) & Orientation (Quaternion).
    *   Translational Offsets ($dx, dy, dz$) evaluated strictly relative to the designated reference frame.
*   **Safety Adjustment:** Constrained experimental motion to the horizontal plane to prevent theoretical collisions with unmapped ground surfaces.

### 2.4 Environmental Collision Modeling
Integrated physical constraints into the software planning space via the `/apply_planning_scene` service.
*   **Implementation:** Injected a definitive ground plane collision object.
*   **Academic Distinction:** RViz visualization grids are strictly aesthetic; MoveIt requires explicitly defined collision objects (`moveit_msgs/msg/CollisionObject`) to calculate obstacle-avoidance trajectories. Failure to define these results in mathematically valid but physically catastrophic motion plans.

## 3. System Architecture & Diagrammatic Modeling

To formalize the operational logic of the robotics stack, system behaviors were codified into five distinct structural models.

*   **Figure 1: Knowledge Mind Map:** Categorized elements into environment, robot models, ROS2 communications, kinematics, execution, and feedback.
*   **Figure 2: Layered Architecture:** Dissected the ecosystem into hierarchical strata: Environment, Model/Config, Planning/Command, Scene/Collision, Control/Execution, and Feedback/Visualization. This clarifies that MoveIt is a trajectory generator, not a direct motor controller.
*   **Figure 3: Computation Graph:** Mapped the data lifecycle from `pymoveit2` to fake hardware execution. 
    *   *Data flow:* TF Read → `/move_group` target formulation → URDF/SRDF/IK resolution → `JointTrajectory` generation → `scaled_joint_trajectory_controller` execution → State update → RViz reflection.
*   **Figure 4: Troubleshooting Matrix (Plan & Execute):** Developed a diagnostic framework classifying failures across Command, Planning, Execution, and Feedback layers to optimize debugging efficiency.
*   **Figure 5: Hardware Readiness Checklist:** Established strict protocols for transitioning to physical hardware, isolating safety procedures, operational sequences, and emergency mitigation.

## 4. Key Understandings Gained

The defining operational paradigm of the UR5 software stack operates on a strict four-stage pipeline: **Command → Planning → Execution → Feedback**.

1.  **Command Layer:** Defines the spatial objective (target pose of `tool0` relative to `base_link`).
2.  **Planning Layer:** MoveIt evaluates the spatial objective against kinematics, current TF, and scene geometry to generate a mathematically viable joint trajectory.
3.  **Execution Layer:** Trajectory controllers interpret the data and actuate the hardware (or fake interface).
4.  **Feedback Layer:** `/joint_states` (joint space) and TF (Cartesian space) confirm kinematic execution.

*Diagnostic Principle:* Debugging must remain strictly directional along this data flow. Verification of target viability, scene accuracy, and state telemetry must precede any structural parameter modifications.

## 5. Problems Encountered and Lessons Learned

*   **WSL Daemon Instability:** Certain ROS2 action/daemon commands hung indefinitely in the WSL environment. 
    *   *Resolution:* Enforced the `--no-daemon` flag for node/topic inspection to bypass daemon synchronization issues.
*   **Planning Failures for Proximal Targets:** Discovered that proximity does not guarantee reachability. Failures stem from quaternion singularities, IK solver limitations (local minima in numerical KDL), or implicit collision flags.
*   **TF Discrepancies:** Learned that TF output mismatches frequently result from referential misunderstandings (e.g., misaligned `base_link` vectors) rather than core calculation errors.
*   **Configuration Conservatism:** Established a strict rule against prematurely editing foundational files (URDF, SRDF, limits). Systemic data flow must be fully analyzed before adjusting core constraints.

## 6. Current Progress

*   **Completed:** ROS2 package `ur5_state_reader` creation.
*   **Completed:** `/joint_states` programmatic subscription.
*   **Verified:** Planning group (`ur_manipulator`), base (`base_link`), and tip (`tool0`) frames.
*   **Completed:** Kinematic solver isolation and validation.
*   **Executed:** `pymoveit2` procedural pose goals on fake hardware.
*   **Modeled:** Integrated physical collision boundaries into the planning scene.
*   **Synthesized:** 5 architectural and diagnostic diagrams governing ROS2/UR5 logic.

## 7. Plan for Next Week

The core objective for Week 3 is rigorous, controlled physical hardware validation, transitioning from abstract simulation to physical actuation.

**Phase 1: Pre-Experiment Hardware Checks**
*   Verify physical emergency stop (E-stop) accessibility and protocols.
*   Secure the physical workspace radius.
*   Validate network topography (UR5 IP connection).
*   Confirm controller state transitions from fake interface to physical `ur_robot_driver`.
*   Audit `/joint_states`, Cartesian TFs, and the physical congruence of the RViz planning scene.

**Phase 2: Conservative Actuation Testing**
*   **Action:** Execute a low-velocity, highly constrained planar translation (1–2 cm along a single Cartesian axis). Orientation must remain locked.
*   **Data Capture Requirements:** 
    *   Pre-execution pose.
    *   Target pose coordinates.
    *   Post-execution state telemetry.
    *   Controller deltas (Desired vs. Actual vs. Error).
    *   TF broadcast latency.
*   **Goal:** Prove hardware-software parity without inducing kinematic risk, establishing a secure baseline for future complex trajectory generation.
