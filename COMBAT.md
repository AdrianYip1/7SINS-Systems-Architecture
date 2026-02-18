# 7SINS Combat Engine — System Design Overview

This document describes the high-level architecture of the 7SINS grid-based combat engine.
The focus is on subsystem structure, data flow, and deterministic execution rather than gameplay presentation.

Source code is private. Documentation is provided for design review and portfolio purposes.

---

## 1. System Goals

- Deterministic combat resolution using a fixed timestep.
- Modular subsystem separation to reduce cross-system coupling.
- Data-driven behaviour for monsters, hazards, and actions.
- Predictable state transitions to simplify debugging and testing.

---

## 2. High-Level Architecture

The combat engine is organized into independent subsystems that communicate through shared state and scheduled updates.

| Subsystem        | Responsibility | Inputs | Outputs |
|------------------|---------------|--------|---------|
| Tick Controller  | Advances time and drives updates | elapsed time | tick events |
| Action Queue     | Stores and resolves actions (FIFO) | monster actions, player input | executed actions |
| Grid System      | Spatial layout and movement | entity updates | cell state |
| Entity State     | Monster stats, gauges, modifiers | combat events | updated stats |
| Hazard System    | Timed cell effects | tick events | damage/heal/debuff events |
| Team Manager     | Active + reserve logic | death/swap events | updated lineup |

Subsystems do not directly modify each other’s internals; changes propagate through events generated during each tick.

---

## 3. Execution Model

Combat runs on a fixed tick (~1s timestep).

Per tick sequence:

1. Tick Controller advances time.
2. Action Queue selects the next action.
3. Entity State applies action results.
4. Hazard System processes active hazards.
5. Team Manager resolves deaths and swaps.
6. Grid System updates occupancy and links if required.

This update order ensures deterministic behaviour regardless of frame rate.

---

## 4. Grid System

Each team operates on a grid (typically 3×3).

A cell may contain:
- one monster
- multiple hazards
- one obstacle

Directional links define movement: (up, down, left, right)


Traversal requires:
- a valid link between cells
- no blocking obstacle
- an unoccupied destination cell

Hazards of the same type overwrite existing ones, while different hazard types can stack.

---

## 5. Entity State Model

Monsters are instantiated from reusable templates defining element, class, base stats, and available actions.

### Core Runtime Values

| Value | Description |
|------|-------------|
| HP | Current and maximum health |
| AG | Action Gauge; enables actions when full |
| EG | Energy Gauge for stronger abilities |

### Combat Multipliers

- Physical attack / defense
- Elemental attack / defense

Multipliers can change dynamically through actions and hazards.

---

## 6. Action System

### Auto Actions

Monsters select actions from a weighted pool: PhysAttack, ElmtAttack, Heal, Guard, Focus, Slack


Weights may be modified by skills or combat state.

Each action defines:
- targeting rules
- power values
- execution timing

### Active Actions

Player-selected actions use the same execution pipeline but can be inserted at the front of the queue.

This allows manual overrides without breaking deterministic execution.

---

## 7. Hazard System

Hazards exist on grid cells with defined properties:

| Property | Description |
|----------|-------------|
| Duration | Number of ticks remaining |
| Strength | Effect magnitude |
| Type     | Damage, Heal, Debuff, etc. |

Each tick:
- Hazard duration decreases.
- Effects are applied to the occupying monster.

Centralized definitions maintain consistency while allowing instance-level overrides.

---

## 8. Team and Reserve Management

A team consists of:
- active monsters placed on the grid
- an ordered reserve list

Swap behaviour:
- A reserve monster may replace an active one at the same cell.
- When an active monster dies, the next alive reserve unit may automatically swap in.

This preserves spatial consistency and avoids grid restructuring during combat.

---

## 9. Obstacles

Obstacles occupy cells and block traversal.

Removal conditions:
- HP reaches zero, or
- duration expires.

Obstacle state is evaluated during each tick after action resolution.

---

## 10. Data Flow Between Subsystems
Tick Controller
↓
Action Queue → Entity State
↓ ↓
Hazard System → Team Manager
↓
Grid System


The Tick Controller acts as the root scheduler.
Subsystems update sequentially to avoid ambiguous state changes.

---

## 11. Summary

| System        | Purpose |
|---------------|---------|
| Tick Controller | Drives deterministic updates |
| Action Queue  | Controls execution order |
| Grid System   | Manages spatial state |
| Entity State  | Stores monster stats and modifiers |
| Hazard System | Applies timed effects |
| Team Manager  | Handles swaps and reserve logic |
| Obstacles     | Temporary movement blockers |

The combat engine emphasizes deterministic execution, modular subsystem boundaries, and predictable data flow.
This structure supports scalable feature additions without requiring changes to core architecture.








