# 7SINS Combat Engine — System Design Overview

This document describes the high-level architecture of the 7SINS grid-based combat engine.
The focus is on subsystem structure, data flow, and deterministic execution rather than gameplay presentation.

Source code is private. Documentation is provided for design review and portfolio purposes.

---

## 1. System Functions

- **Deterministic combat resolution** using a fixed timestep tick system
- **Modular subsystem separation** to reduce cross-system coupling
- **Data-driven behavior** for monsters, hazards, and actions via template system
- **Predictable state transitions** to simplify debugging and testing
- **Spatial consistency** through grid-based positioning and link-based traversal

---

## 2. High-Level Architecture

The combat engine is organized into independent subsystems that communicate through shared state and scheduled updates.

| Subsystem        | Function | Inputs | Outputs |
|------------------|---------------|--------|---------|
| Tick Controller  | Advances time and drives updates | elapsed time | tick events |
| Action Queue     | Stores and resolves actions (FIFO) | monster actions, player input | executed actions |
| Grid System      | Spatial layout and movement | entity updates | cell states |
| Entity State     | Monster stats, gauges, modifiers | combat events | updated stats |
| Hazard System    | Timed cell effects | tick events | damage/heal/debuff events |
| Team Manager     | Active + reserve logic | death/swap events | updated lineup |
| Obstacle System  | Cell blocking and traversal rules | damage/duration updates | cell availability |

Subsystems do not directly modify each other's internals; changes propagate through events generated during each tick.

---

## 3. Execution Model

Combat runs on a fixed tick (~0.5s timestep).

**Per tick sequence:**

1. **Tick Controller** advances time accumulator
2. **Action Queue** selects and dequeues the next action
3. **Entity State** applies action results (damage, healing, stat modifications)
4. **Hazard System** processes active hazards (tick duration, apply effects)
5. **Team Manager** resolves deaths and triggers auto-swap if needed
6. **Grid System** updates occupancy and link states if required

This update order ensures deterministic behavior regardless of frame rate or client performance.

---

## 4. Grid System

Each team operates on a determined grid size (typically 3×3, configurable).

**Cell structure:**
- One monster (or nil)
- Multiple hazards (array, same type overwrites, different types stack)
- One obstacle (or nil)
- Directional links (up, down, left, right) defining traversability

**Traversal rules:**
- Valid link must exist between source and destination cells
- Destination cell must not contain a blocking obstacle
- Destination cell must be unoccupied (no monster)

**Spatial operations:**
- Monster placement/removal maintains position mapping
- Hazard addition merges or overwrites based on effect type
- Link modification enables dynamic grid restructuring

---

## 5. Entity State Model

Monsters are instantiated from reusable templates defining element, class, base stats, and available actions.

### Main Values Required for Battle Loop

| Value | Description |
|------|-------------|
| HP | Current and maximum health; death state at 0 |
| AG | Action Gauge; fills up with ticks, activates auto actions when full |
| EG | Energy Gauge; separate resource with threshold and max for special abilities |

### Combat Multipliers

- Physical attack multiplier
- Elemental attack multiplier
- Physical defense multiplier
- Elemental defense multiplier

Multipliers can change dynamically through actions and hazards, with reset methods to restore template defaults.

### Action Selection

Monsters maintain:
- **Auto actions**: weighted probability distribution (modifiable at runtime)
- **Active actions**: player-selectable actions with metadata (damage, targeting, duration)

---

## 6. Action System

### Auto Actions

Monsters select actions from a weighted pool: PhysAttack, ElmtAttack, Heal, Guard, Focus, Slack

**Selection algorithm:**
- Cumulative probability distribution over available actions
- Random selection weighted by current probabilities
- Probabilities can be modified mid-combat (e.g., "MaximizePrimaryProb" effect)

Each action defines:
- Targeting rules (single target, area, self, etc.)
- Power values (damage, heal amounts)
- Execution timing (animation duration)

### Active Actions

These use the same execution pipeline but can be inserted at the front of the queue

This allows manual overrides without breaking execution or requiring separate code paths.

---

## 7. Hazard System

Hazards exist on grid cells with defined properties:

| Property | Description |
|----------|-------------|
| Duration | Number of ticks remaining (decrements each tick) |
| Strength | Effect magnitude (damage amount, heal amount, debuff percentage) |
| Type     | Effect category (damage, heal, def_down, etc.) |

**Per-tick behavior:**
- Duration decreases by 1
- Effect is applied to the occupying monster (if present)
- Expired hazards are removed from the cell

**Hazard management:**
- Centralized definitions maintain consistency (duration, type, default strength)
- Instance-level strength overrides allow customization
- Same-type hazards overwrite existing ones; different types stack

---

## 8. Team and Reserve Management

A team consists of:
- **Active slots**: monsters placed on the grid with position tracking
- **Reserve list**: ordered collection of monsters not currently on the grid

**Swap behavior:**
- A reserve monster may replace an active one at the same cell position
- The previously active monster moves to reserve at the swapped position
- Position consistency is maintained (no grid restructuring required)

**Auto-swap on death:**
- When an active monster dies, the next alive reserve unit automatically swaps in
- Reserve list is compacted (alive monsters first) before selection
- Ensures combat continuity without manual intervention

---

## 9. Obstacles

Obstacles occupy cells and block traversal.

**Removal conditions (OR logic):**
- HP reaches zero (damage-based removal)
- Duration expires (time-based removal)

**State evaluation:**
- Obstacle state is checked during each tick after action resolution
- Destroyed obstacles immediately free the cell for traversal and placement

---

## 10. Data Flow Between Subsystems

```
Tick Controller
    ↓
Action Queue → Entity State
    ↓              ↓
Hazard System → Team Manager
    ↓              ↓
Grid System ←─────┘
```

**Flow characteristics:**
- Tick Controller acts as the root scheduler
- Subsystems update sequentially to avoid ambiguous state changes
- Entity State modifications trigger downstream updates (death → Team Manager)
- Grid System reflects final state after all modifications complete

---

## 11. Design Patterns

**Encapsulation:**
- Monster state (HP, gauges, multipliers) is private with controlled accessors
- Grid cells expose only necessary state through getter methods
- Template system separates data definition from runtime behavior

**Separation of Concerns:**
- Action definitions separate from execution logic
- Hazard effects separate from grid management
- Team management separate from entity state

**Determinism:**
- Fixed tick rate ensures consistent timing
- FIFO queue ensures predictable execution order
- Sequential subsystem updates prevent race conditions

---

## 12. Summary

| System        | Purpose |
|---------------|---------|
| Tick Controller | Drives deterministic updates at fixed intervals |
| Action Queue  | Controls execution order with FIFO + priority insertion |
| Grid System   | Manages spatial state and traversal rules |
| Entity State  | Stores monster stats, gauges, and modifiers |
| Hazard System | Applies timed effects with duration tracking |
| Team Manager  | Handles swaps and reserve logic with auto-recovery |
| Obstacles     | Temporary movement blockers with dual removal conditions |

The combat engine emphasizes deterministic execution, modular subsystem boundaries, and predictable data flow.
This structure supports scalable feature additions without requiring changes to core architecture.
