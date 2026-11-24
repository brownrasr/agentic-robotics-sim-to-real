# Evidence Gates for Agentic Robotics: Simulation to Real Hardware


> **Decision:** Use these evidence gates when moving an agentic policy from data and simulation through shadow mode into bounded hardware trials; skip them if simulation success is being treated as deployment proof.
>
> **For:** robotics researchers, safety reviewers, and deployment engineers  ·  **Not for:** unsupervised real-world rollout or teams without stop and recovery controls

A policy that succeeds in simulation is a hypothesis about a physical system. Moving it to a robot requires evidence about the task, data, coordinate frames, dynamics, perception, control rate, latency, uncertainty, operating envelope, human supervision, and recovery. “Sim-to-real” is not one conversion step.

This guide defines a stage-gated dossier. It helps teams decide whether a learned or agentic policy should remain in data review, simulation, replay, shadow mode, bounded hardware testing, or retirement. It does not certify safety and does not authorize unattended actuation.

The main artifact is [`transfer-evidence.yaml`](assets/transfer-evidence.yaml). Every gate starts closed. A human with independent stop authority owns each transition.

This repository is deliberately scoped to non-certified, low-energy laboratory research. It does not provide a conformity assessment, machinery risk assessment, or functional-safety design. Industrial, collaborative, medical, transport, public-space, human-contact, and other regulated use requires qualified safety engineering and the standards and authorities applicable to that robot and jurisdiction.

## Stack roles before the gates

| Role | Candidates | Object or interface | Choose when | Boundary to verify |
|---|---|---|---|---|
| Middleware graph | ROS 2 | nodes, topics, services, actions, quality-of-service concepts | a distributed robot software graph and ecosystem are required | timing, discovery, security, message definitions, and real-time expectations |
| Hardware control | ros2_control | hardware interfaces, controllers, controller manager | hardware abstraction and controller lifecycle fit the platform | driver quality, rates, limits, watchdogs, and controller/hardware support |
| Motion planning | MoveIt 2 | robot models, planning scene, trajectories | manipulation/path planning is part of the task | collision model, controllers, execution monitoring, and robot integration |
| General robot simulation | Gazebo | worlds, models, sensors, plugins, physics | a system-level robot/world simulator fits the test | engine/plugin choice, assets, sensor fidelity, timing, and ROS integration |
| Physics simulation | MuJoCo | articulated models, contacts, dynamics | contact-rich dynamics and its model format fit the experiment | asset conversion, actuator/sensor mapping, rendering and control loop |
| Robot-learning platform | Isaac Lab | simulation environments and learning workflows | its simulator/toolchain and task abstractions fit the lab | hardware/software requirements, asset fidelity, deployment route and license |
| Manipulation benchmark environment | robosuite | robot manipulation environments/tasks | standardized manipulation experiments are useful | task/robot coverage, controller choice, real-hardware mapping |
| Learning/data stack | LeRobot | datasets, models, policies and robot-learning workflows | data-driven policy work needs its supported structures | supported robots, data schema, policy/runtime and deployment constraints |
| Dataset family | Open X-Embodiment | cross-embodiment datasets and models | dataset discovery or research comparison fits the claim | licenses, schemas, embodiments, task fit and preprocessing |
| Environment API | Gymnasium | environment/reset/step conventions | an RL environment API is useful | physical meaning, timebase, termination, observation/action validation |

This is a role map, not a recommended fixed stack. No source establishes that two rows interoperate, that a simulated policy transfers, or that a deployment is safe.

## Gate 0: task and robot contract

Write an observable task before choosing a simulator or policy:

- robot model, end effector, sensors, payload, firmware, and control interface;
- workspace geometry, people/animals, surfaces, lighting, network, and environmental limits;
- coordinate frames, units, timestamps, calibration, and synchronization;
- initial and terminal states;
- permitted contacts, force/torque, speed, acceleration, joint, thermal, and power bounds;
- success, acceptable degradation, prohibited states, timeout, and recovery;
- human supervisor position, physical E-stop, remote stop, and who may reset.

If a success metric ignores unsafe near misses, it is not a sufficient task metric. Separate task completion from constraint violations and intervention rate.

[ROS 2 concepts](https://docs.ros.org/en/jazzy/Concepts/Basic.html) document foundational ROS concepts. [ros2_control](https://control.ros.org/jazzy/index.html) documents a control framework, and [MoveIt 2](https://moveit.picknik.ai/main/index.html) documents motion-planning/manipulation software. These roles are related but not interchangeable.

## Gate 1: data and policy provenance

Record who collected demonstrations, the robot/simulator and sensors, calibration, task distribution, failed examples, interventions, exclusions, transformations, and licenses/consent. [LeRobot](https://github.com/huggingface/lerobot) is a robot-learning project reference. [Open X-Embodiment](https://robotics-transformer-x.github.io/) represents a multi-dataset robotics effort. Neither source guarantees that a dataset fits a new robot or task.

For AI-generated plans, labels, rewards, or code, retain model/runtime identity, instruction/config hash, tool access, reviewer, and resulting diff. Treat generated labels and synthetic trajectories as inputs requiring validation, not ground truth.

Do not use data without a valid right and governance basis. Blur/removal is not automatically sufficient de-identification when homes, voices, faces, screens, routines, or locations remain inferable.

## Gate 2: simulation model disclosure

[Gazebo Harmonic](https://gazebosim.org/docs/harmonic/getstarted/), [MuJoCo](https://mujoco.readthedocs.io/en/stable/overview.html), [Isaac Lab](https://isaac-sim.github.io/IsaacLab/main/source/overview/core-concepts/index.html), and [robosuite](https://robosuite.ai/docs/overview.html) are references for different simulation/robot-learning environments. [Gymnasium’s environment guide](https://gymnasium.farama.org/tutorials/gymnasium_basics/environment_creation/) documents environment construction concepts. A common API does not imply common dynamics, sensors, timing, contact, or rendering.

Freeze and disclose:

- simulator and physics version/build;
- robot/scene assets and hashes;
- solver, timestep, substeps, control rate, and actuator model;
- mass, inertia, friction, compliance, damping, backlash, saturation, and delay;
- camera/lidar/force sensor intrinsics, extrinsics, noise, occlusion, and update timing;
- reset distribution, randomization ranges, seeds, reward/termination code;
- known missing physics and why they may affect the task.

Domain randomization can test sensitivity; it does not prove the real system lies inside the randomized distribution.

## Gate 3: simulation evidence

Evaluate on held-out task instances and declared perturbations. Keep success, constraint violations, minimum margin to limits, collisions/contacts, interventions, timeout, energy, and uncertainty separately. Report distributions, not only a highlight video.

Required negative tests may include sensor dropout, delayed observations/actions, calibration offset, friction/payload shift, occlusion, unexpected but bounded obstacle placement, and invalid agent output. Use a deterministic safety layer to reject actions outside the envelope; the policy cannot override it.

No simulator result authorizes hardware. A policy that exploits reset behavior, numerical contact, privileged state, or simulator-specific rendering must return to design.

## Gate 4: replay and hardware-in-the-loop

Before motors move, replay recorded sensor streams and inspect proposed commands. Then use hardware-in-the-loop where feasible: real controller/interface, fake or physically disconnected actuators, actual rates, and bounded fault injection.

Verify frame/unit conversion, timestamps, queue growth, deadline misses, saturation, watchdog, stale commands, network loss, process crash, and restart. A command should expire; reconnect must not replay a stale trajectory.

Record discrepancy between simulated and observed sensors/control paths. Unknown timing is a blocked gate.

## Gate 5: shadow mode

Run the policy on the real sensor stream without allowing it to control the robot. Compare proposed actions with a trusted controller or human decision where that comparison is meaningful. Measure out-of-distribution flags, safety-layer rejects, disagreements, latency, and action smoothness.

Shadow agreement is not proof of safe control: the policy may fail only after its own actions change future observations. Still, a high reject or disagreement rate is a clear reason not to proceed.

## Gate 6: bounded real-world trial

The first actuation trial uses the smallest energy, speed, workspace, payload, duration, and action set that tests the specific transfer assumption. Establish barriers and clearance. A trained observer holds a tested physical E-stop. Use independent joint/workspace limits and a watchdog outside the policy process.

Start with one-step or short-horizon actions, return to a known safe state, inspect logs and hardware, then decide whether to repeat. Never broaden several variables at once. Any unexplained contact, limit approach, stale command, oscillation, thermal/power anomaly, or safety-layer disagreement closes the gate.

For mobile or outdoor robots, account for public space, communications loss, geofencing, bystanders, weather, and local regulation. For medical, industrial, transport, weapons-adjacent, or human-contact applications, this guide is not an adequate assurance process; obtain qualified domain and regulatory review.

## Gate 7: monitored operation or retirement

A bounded pilot requires versioned robot/policy/config identity, operating limits, preflight, trained operators, maintenance/calibration, incident ownership, telemetry retention, rollback, and a defined safe state. Monitor drift in sensors, environment, payload, task mix, and intervention rate.

Retire or return to an earlier gate when assumptions change, telemetry is missing, incidents are unexplained, hardware ages beyond calibration, or upstream/model versions change. Preserve failed evidence; do not overwrite it with a later successful run.

## Transfer discrepancy register

For each assumption, record simulated value/range, measured real value/range, measurement method, uncertainty, impact, mitigation, and gate owner. High-priority discrepancies commonly include:

- actuator latency, saturation, backlash, and thermal behavior;
- friction/compliance/contact and payload center of mass;
- camera exposure, rolling shutter, depth artifacts, occlusion, and calibration;
- clock synchronization and transport jitter;
- reset/initial state and object placement;
- human behavior and obstacles not represented in simulation;
- policy compute latency and resource contention;
- recovery after partial execution.

A weighted aggregate “reality score” can hide one fatal discrepancy. Keep safety-critical assumptions individually gated.

## Physical and legal safety boundary

The [OSHA robotics topic page](https://www.osha.gov/robotics) collects industrial-robotics hazard and safety resources for a US workplace context. The [ISO 10218-1:2025 catalogue record](https://www.iso.org/standard/73933.html) identifies an international industrial robot safety standard and its stated scope; the catalogue page is not the standard text and does not demonstrate conformity. These references explain why a generic research checklist cannot substitute for a qualified, application-specific risk assessment.

- Use only authorized robots, datasets, environments, networks, and tasks.
- No unattended real-world actuation based on this guide.
- A human with independent physical stop authority is mandatory for initial trials.
- Safety limits and watchdogs must execute outside the learned/agent policy and fail toward a defined safe state.
- Do not test around uninformed bystanders, animals, shared public space, or valuable/fragile property.
- Do not let a language model generate and immediately execute arbitrary actuator commands.
- Respect dataset licenses, participant consent, privacy, workplace safety, spectrum/aviation/road rules, and application-specific regulation.
- Simulation success, benchmark score, model confidence, or agent explanation is not a safety case.

## Source boundary

The ten upstream records in [`sources/sources.json`](sources/sources.json) support middleware, simulation, control, learning, data, or environment roles. They do not endorse this combined gate model or prove interoperability. Gate ordering, discrepancy register, shadow protocol, and stop rules are author recommendations.

## Maintenance

Reopen the dossier when robot hardware/firmware, calibration, asset, simulator, control rate, policy/model, dataset, sensor, task, workspace, payload, safety layer, network, or operator procedure changes. Version every transition and preserve the evidence used by the approving human.

## Adjacent guides

- [ai-research-reproducibility](https://github.com/Lindenberg668/ai-research-reproducibility) — Use it to package the claims, immutable inputs, environment, variance, and negative results behind each gate.
- [agent-execution-control-sandbox](https://github.com/MargaritaOfficial/agent-execution-control-sandbox) — Use it when simulation or policy tooling also executes untrusted code that needs isolation.
- [agent-observability-field-guide](https://github.com/duthdoll45/agent-observability-field-guide) — Use it when policy decisions, tool calls, and intervention points need structured traces.

## License

Original guide and transfer-evidence dossier: CC BY 4.0. Linked projects and documentation retain their terms.

If this saved you a decision, star the repo — it is how the next reader finds it.
