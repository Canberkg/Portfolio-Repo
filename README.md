# Portfolio-Repo
**Iso Crowd Combat — Technical Portfolio**


https://github.com/user-attachments/assets/53e766b1-7d2b-482c-b5b4-4b7d1db53104


**Project Overview**

This is a Unity-based isometric action combat system built with a strong emphasis on systems architecture, AI design, and scalable gameplay logic. The project demonstrates several industry-standard patterns — state machines, behavior trees, blackboards, spatial query systems, and event-driven communication — working together to create fluid player control and coordinated multi-enemy AI.

**- Core Foundation — Interfaces & Types**

The entire architecture is built on a clean interface contract layer defined in Interfaces.cs. Every major system — input, movement, animation, vision, health, weapons, state machines — is expressed as an interface rather than a concrete type. This decouples the player and AI systems while allowing them to share the same logic and behaviors. Also Data-Driven Approach is fully emphasized throughout the project.

Types.cs defines the project's shared vocabulary: enumerations for attack types, combat token types, spatial generation configurations, and condition/weight modifiers used in the spatial query system. A key structure here is CandidatePoint, which represents a world-space position with ownership and claim metadata used by the positioning system.

**- State Machine Architecture**

Base Layer
BaseState.cs and BaseStateMachine.cs form a generic, reusable state machine framework shared by both the player and AI. States are context-typed, meaning each state has typed access to its owner's full context object (player or AI), preventing unsafe casting at runtime.

Transitions are handled through two guard conditions: CanExitState on the current state and CanEnterState on the target state. This two-phase check gives individual states full authority over when transitions occur, making the system highly self-contained.

States are cached by type in a dictionary, so each state instance is created once and reused — avoiding per-frame allocations during gameplay.

Animation Binding
AnimationController.cs wraps Unity's Animator and implements the IAnimator interface. It handles crossfade transitions, root motion delegation, and animation duration querying. All states interact with animation through this interface, keeping Unity's Animator entirely behind an abstraction.

**- Player System**

Input
InputHandler.cs uses Unity's new Input System and implements input buffering — a critical feature for action games. Attack and dodge inputs are stored with timestamps and remain valid within a short window. This means a player pressing attack a few frames before a state can receive it will still have that input registered, creating a responsive feel without requiring frame-perfect timing.

A dodge cooldown system prevents consecutive dodges within a tight window. Camera-relative movement remapping is also handled here, translating raw stick/keyboard input into world-space directional intent.

Movement
MovementController.cs handles all player physics: smooth acceleration/deceleration curves, knockback application, and — importantly — a snap-to-target system. When the player enters combat, the movement direction snaps toward the nearest valid enemy within range, with sphere-cast obstacle detection to avoid targeting enemies through walls. This gives the combat a locked-on feel without a hard lock-on system.

Player Controller & Context
PlayerController.cs acts as the root context object for the player. It aggregates Input, Movement, Animation, Weapon, and Health systems, exposes them through the IPlayerContext interface, and constructs the state machine with all player states registered. It also wires up health UI feedback.

Player States
Locomotion states — Idle, Walk, Run, Sprint — manage blend tree parameters and transition logic. The idle-to-run and run-to-idle transitions are implemented as dedicated transition states (rather than direct transitions), which play a brief crossfade animation before landing in the target state. This is a professional technique for avoiding animation pops.

Combat states form a three-hit combo chain: LightAttack → LightAttackCont → LightAttackEnd. Each state manages its own frame windows — ranges of animation frames during which the hitbox is active, and during which a buffered input can continue the combo. This frame-data system ties hitbox lifetime directly to animation, making timing adjustments purely data-driven.

DodgeState uses the input buffer's dodge flag and applies a timed movement impulse, providing invincibility-frame-style evasion.

**- AI System**

AI Controller & Context
AIController.cs mirrors the player's architecture as the root context for each AI agent. It holds references to all AI subsystems — Vision, Movement, Health, Animation, Weapon, Blackboard, Behavior Tree — and initializes them. It also manages the agent's participation in the token system, handling timeout logic when a combat slot is held too long without engagement.

Movement
Movement.cs wraps Unity's NavMesh Agent and implements IAIMovement. It integrates root motion from the Animator with the NavMesh agent's velocity, so animation-driven locomotion and pathfinding work together without sliding. Stopping distance and speed are configurable per state.

Vision
Vision.cs is a lightweight distance-based detection system. It tracks the player's position continuously and exposes two ranges: a chase range (when the agent starts pursuing) and an engagement range (when combat logic activates). While simple, it's sufficient for the isometric combat context and integrates cleanly with the behavior tree conditions.

**- Behavior Tree System**

The AI's decision-making is built on a custom behavior tree with full node composition support.

Tree Structure
BehaviorTree.cs implements the fundamental node types: Sequence (all children must succeed), SequenceMem (remembers progress across frames), Selector (first successful child wins), Inverter, and Leaf (executes an ILeafStrategy). These compose into arbitrarily deep decision trees evaluated each frame.

Tree Construction
BehaviorTreeExpert.cs is responsible for building and running the tree. The tree is constructed programmatically and evaluates priority in this order:

Hit Interrupt — If the agent is being hit (light or heavy), drop everything and enter the appropriate hit-response state. If the damage taken results a death situation, the states is forcefully changed to Death State.
Combat Flow — If a combat token is held:
Manage weapon draw/sheathe.
If in range, select and execute an attack.
If out of range, approach or reposition.
Chase — If no token but player is in range, pursue.
Patrol / Idle — Default behavior when no engagement is active.
This priority structure guarantees that higher-urgency behaviors always interrupt lower ones.

Strategies
ActionStrategies.cs contains all executable leaf behaviors:

StateTransition<T> and ForceStateTransition<T> — trigger state machine transitions (conditional or forced).
RunQuery — executes a spatial query to find a valid positioning point.
AttackSelector — randomly selects an available attack from the AI's data.
AskTicket — requests a combat token from the CombatManager.
ConditionStrategies.cs contains twelve pure condition evaluators (weapon state, slot claim status, distance thresholds, hit type detection, token availability) that return Success/Failure without side effects. Clean separation of conditions from actions is a hallmark of well-structured behavior trees.

**- Blackboard System**

BlackBoard.cs is a type-safe shared memory store for AI agents. Keys are hashed using FNV-1a for fast lookup, and values are stored as typed BlackboardEntry<T> objects. This prevents the stringly-typed dictionaries common in simpler implementations.

BlackBoardExpert.cs wraps the blackboard with domain-specific accessors for the AI — claimed spatial slots, attack selections, current targets, etc. — acting as the bridge between raw blackboard storage and meaningful game state.

**- Combat Manager & Token System**

CombatManager.cs is the global combat coordinator — a singleton that manages two critical resources shared across all AI agents:

Token System
Combat tokens control how many enemies can actively attack the player simultaneously. There are two close-range tokens (allowing two enemies to engage up close) and one far-range token (with a 3-second cooldown to prevent ranged harassment spam). Enemies must hold a token to enter attack states; if unavailable, they reposition and wait.

This system is what prevents the unrealistic "enemy pile-on" behavior common in simpler AI implementations. Each enemy independently competes for tokens, creating naturally staggered attack patterns.

Spatial Slot System
The combat manager generates candidate positioning points in dynamic rings around the player — an inner ring at roughly 4 units and an outer ring at roughly 6 units, with approximately 17 evenly distributed points per ring. These slots are regenerated each frame as the player moves.

Enemies claim slots using the TSP (Tactical Spatial Positioning) system and hold them until they disengage. This prevents multiple enemies from stacking on the same position, producing the flanking and surrounding behavior seen in polished action games.

- Tactical Spatial Positioning (TSP)
TSP.cs implements a spatial query and scoring system for selecting the best positioning target from a set of candidate points. It operates in two phases:

Filtering — eliminates candidates that fail hard conditions (e.g., already claimed by another agent).
Scoring — ranks remaining candidates by weighted criteria (e.g., proximity to current position) and selects the highest-scoring one.
Criterion.cs defines individual scoring and filtering criteria (DistanceCriterion, NotClaimed), and QueryLibrary.cs acts as a registry for named query configurations. This makes the positioning behavior composable and editable without touching query logic.

Ref :
- GameAIPro - Chapter 26,Tactical Position Selection

**- AI State Machine**

The AI uses the same generic state machine as the player. Its states are organized into three behavioral tiers:

Locomotion — IdleStateAI, PatrolStateAI (waypoint traversal), ChaseStateAI (NavMesh pursuit).

Weapon Transitions — IdleFromNwToWW (draw weapon) and IdleFromWWToNW (sheathe weapon) are dedicated transition states that play weapon management animations before entering combat or returning to patrol, preserving visual fidelity.

Combat — ApproachStateAI (close the gap to the player), CombatStateAI (execute attacks with frame-based hitbox windows matching the animation), PositioningStateAI (move to a claimed spatial slot while waiting for a token).

Hit Response — LightHitStateAI, HeavyHitStateAI are interrupt states triggered by the behavior tree's top-priority hit check. DeathStateAI handles death animation and cleanup.

- Data Layer — ScriptableObjects
ActorData.cs defines a ScriptableObject hierarchy for configuring actors without code changes:

ActorData — base health, damage thresholds, animation data.
PlayerData / AIData — extend base with role-specific fields (vision ranges, patrol waypoints, etc.).
AnimationData — stores animation hash IDs, playback speeds, and frame rates.
CombatData — defines the seven frame-window markers per attack (startup, active, recovery, combo window, etc.).
LocomotionData — movement speeds and 8-directional blend tree mappings.
SpatialQuery.cs is a ScriptableObject that configures TSP query behavior — generation type, condition configs, and weight configs — making positioning tuning fully designer-accessible.

System Interaction Map

Player Input ──► PlayerController (Context)
                    ├── InputHandler     ──► State Machine ──► Player States
                    ├── MovementController                        (Locomotion / Combat)
                    ├── AnimationController
                    ├── WeaponController ──► HurtBox ──► IDamagable (enemies)
                    └── Health

AI Agent ──► AIController (Context)
               ├── Vision               ──► detects player proximity
               ├── Movement (NavMesh)   ◄── AI States drive destination
               ├── AnimationController
               ├── WeaponController     ──► HurtBox ──► IDamagable (player)
               ├── Health
               ├── BlackBoard           ◄──► BehaviorTreeExpert (reads/writes)
               └── BehaviorTree         ──► drives AI State Machine

CombatManager (Singleton)
   ├── Token System   ◄──► AskTicket (BT Strategy) — limits simultaneous attackers
   └── Slot System    ◄──► TSP + RunQuery           — distributes positioning targets
   
**Architectural Highlights**

- Separation of concerns is consistently maintained: input, movement, animation, combat, and AI decision-making are each isolated behind interfaces and never directly reference each other's implementations.
- Data-driven design via ScriptableObjects means frame windows, speeds, ranges, and spatial query configs are tunable without recompilation.
- Shared infrastructure — the state machine, animation controller, health, and weapon systems are used identically by both player and AI, minimizing code duplication.
- Emergent AI behavior arises not from scripted patterns but from the interaction of independent systems: tokens limit attackers, spatial slots distribute positions, and the behavior tree prioritizes actions — together producing coordinated, organic-feeling group combat.

**Further Developments**
Custom A* - Although navmesh agents are great for simple stuff, they are not good during tactical positioning.


