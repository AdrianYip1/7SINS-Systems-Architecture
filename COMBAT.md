# 7SINS Combat Engine — System Overview

This document describes the high-level architecture of the 7SINS grid-based combat engine.
The focus is on subsystem structure, data flow, and deterministic execution rather than gameplay presentation.

Source code is private. Documentation is provided for design review and portfolio purposes.

---

## 1. Core Functions

- Battles ooperate with a clock system, which is done too synchronize updates for the monsters, grid, and other battling logic
- The grid controls movement, hazards, obsticals, while keeping track of all their locations
- Monsters are created with templates, where all monsters have unique stats and actions

---

## 2. High-Level Architecture

The combat engine is organized into subsystems that communicate through shared states.

| Subsystem        | Function | Inputs | Outputs |
|------------------|---------------|--------|---------|
| Clock System     | Advances time and sends updates | elapsed time | tick events |
| Action Queue     | Stores and resolves actions (FIFO) | monster actions, player input | executed actions |
| Grid System      | Spatial layout and movement | entity updates | cell states |
| Entity State     | Monster stats, gauges, modifiers | combat events | updated stats |
| Hazard System    | Timed cell effects | tick events | damage/heal/debuff events |
| Team Manager     | Active + reserve logic | death/swap events | updated lineup |
| Obstacle System  | Cell blocking and traversal rules | damage/duration updates | cell availability |

---

## 3. Battle Flow

Combat runs on a 0.5s tick loop.

For each tick:
1. Monster action gauges increase
2. Monsters with full AG choose and execute their actions
   - If a monster chooses attack as their action, they get pushes into an attacking queue
3. Damage, healing, stat changes, and movement are applied
4. Dead monsters are checked and autoswaping logic is called when needed
5. The grid updates the cell states afterwards, if anything has changed

---

## 4. Grid System

Each team operates on a determined grid size (typically 3×3, configurable).

Cell Structure:
Each cell can contrain at maximum
- one monster
- multiple hazards
- one obstacle

Movement uses links which are passed as strings:
- up, down, left, right

Traversal Rules:
- a link between cells must exist
- the destination cell must not have a monster or obstacle occupying it

### Grid Helper Functions

Examples of helper functions for the grid:

- getCell(x, y)
- inBounds(x, y)
- canEnterCell(x, y)
- canTraverse(x, y, direction)
- movementCoordinates(x, y, direction)

### Moving and Editing the Grid

- moveMonster(...)
- addMonster(...)
- removeMonster(...)
- addHazard(...)
- removeHazard(...)
- addObstacle(...)
- removeObstacle(...)

Hazards of the same type overwrite each other.
Different hazard types stack.

---
## 5. Monster System
Monsters are split into 2 parts:
- `MonsterTemplate` — data of the monster loaded froom a database
- `Monster` — the monster object used during battle

Templates do not change during combat.  
Monster objects track HP, gauges, and combat effects.

---
### MonsterTemplate

Contains:
- name, element, and class tags
- primary action type
- level
- hp_max, ag_max, eg values
- battle multipliers
- auto action action lists and probabilities

---

## 6. Hazard System

Hazards are placed on grid cells and apply effects every tick.

Each hazard has:
- duration
- strength
- type

Every tick:
- duration decreases
- effect applies to the monster in that cell (if applicable)
- expired hazards are removed

---

## 7. Team and Reserve Management

Each team has:
- monsters on the grid (active monsters)
- an ordered reserve list

Swap behaviour:
- A reserve monster can replace an active one at the same position.
- When a monster dies, the next alive reserve monster automatically swaps in.

---

## 8. Obstacles

Obstacles block movement on a cell.

They are removed when:
- HP reaches zero or duration expires

---

## 9. Data Flow Between Subsystems

```
Clock System
    ↓
Action Queue → Entity State
    ↓              ↓
Hazard System → Team Manager
    ↓             
Grid System ←─────┘
```

---
## 10. Summary

The combat system:

- Grid handles space.
- Monster handles battle stats for entities.
- Templates hold the database for monsters, and future battling features.
- Hazards and obstacles modify cells.
- The tick loop keeps everything in sync.

This setup allowing for future features to be added without rewriting existing systems.
