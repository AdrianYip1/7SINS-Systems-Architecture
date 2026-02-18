# 7SINS Combat System — Overview

This document outlines the high-level design of the 7SINS grid-based combat engine.  
It focuses on system behaviour and architecture rather than implementation details.  
Source code is kept private; access can be provided upon request.

---

## Core Loop

Combat runs on a fixed tick (approximately half a second) to maintain deterministic execution.

During each tick:
- The action queue updates.
- The next action executes (attack, heal, guard, etc.).
- Hazards on the grid apply their effects.

Player-triggered actions can be inserted at the front of the queue so they resolve immediately without disrupting the overall system order.

---

## Grid and Positioning

Each team operates on its own grid, typically 3×3.

A cell may contain:
- one monster
- multiple hazards
- one obstacle

Hazards of the same type overwrite existing ones, while different hazard types can stack.

Movement is controlled through directional links between cells (up, down, left, right).  
Traversal checks both connectivity and cell occupancy.

---

## Monsters and Stats

Monsters are defined through reusable templates describing their element, class, base stats, and available actions.

Core runtime values:
- HP — current and maximum health
- Action Gauge (AG) — fills over time; when full, an action can be performed
- Energy Gauge (EG) — secondary resource for stronger abilities

Combat multipliers include:
- physical attack and defense
- elemental attack and defense

These values can change dynamically through skills and hazard effects.

---

## Action System

### Auto Actions

Monsters select actions from a weighted pool such as:
PhysAttack, ElmtAttack, Heal, Guard, Focus, and Slack.

Weights may be modified by skills or combat state.  
Each action defines targeting rules, power values, and animation timing.

### Active Actions

Player-selected actions use the same execution pipeline but can be queued to run next.

Examples include:
- PhysAttack or ElmtAttack — single-target damage
- Heal — restores HP
- Guard — temporary defense increase
- Focus — temporary offense increase
- Slack — pass turn

Damage and healing scale according to attack and defense multipliers.

---

## Hazards

Hazards occupy grid cells for a fixed duration and trigger once per tick.

Typical effects include:
- damage over time
- healing
- defense reduction

Hazard types are defined centrally with default duration and strength, though individual instances can override these values.

---

## Teams and Reserve System

A team consists of active grid slots and an ordered reserve list.

Key behaviours:
- Reserve monsters can swap into active positions.
- When an active monster dies, the next alive reserve unit can automatically replace it.
- Swaps preserve grid position to maintain predictable system behaviour.

---

## Obstacles

Obstacles block movement while active.

An obstacle may be removed by:
- reducing its HP to zero, or
- waiting for its duration to expire.

---

## Summary

System        | Purpose
------------- | -------
Grid          | Holds monsters, hazards, and obstacles
Tick          | Drives combat timing
Action Queue  | Controls execution order
Monsters      | Stats, gauges, and actions
Hazards       | Timed cell effects
Teams         | Active grid and reserve list
Obstacles     | Temporary blockers
