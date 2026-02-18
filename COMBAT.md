# Combat System — Overview

High-level overview of the **7sins** combat system. This document describes design and behavior for showcase purposes.

---

## Core Loop

- **Tick-based** battle: time advances in fixed ticks (e.g. 1 second). Each tick:
  - The **action queue** is updated (FIFO).
  - The **front action** is executed (e.g. attack, heal, guard).
  - **Hazards** on the grid tick and apply their effects (damage, heal, debuffs).
- **Active (player) actions** can be inserted so they execute next (e.g. “use skill now” ahead of auto actions).

---

## Grid & Positioning

- Each side has a **grid** (e.g. 3×3). Every cell can have:
  - **Monster** (one per cell)
  - **Hazards** (multiple per cell; same effect type overwrites, different types stack)
  - **Obstacles** (block movement until destroyed; have HP and/or duration)
- Movement uses **links** between cells (up/down/left/right). Pathing and placement respect traversability and occupancy.

---

## Monsters & Stats

- **HP** — current and max; death at 0.
- **Action Gauge (AG)** — fills over time; when full, the monster can perform an action and the gauge resets.
- **Energy Gauge (EG)** — separate resource with threshold and max; used for stronger or special abilities.
- **Combat multipliers** (modifiable in combat):
  - Physical / Elemental **attack**
  - Physical / Elemental **defense**

Monsters are defined by **templates**: element, class, primary action type, base stats, and which actions they can use.

---

## Actions

**Auto actions** (AI / automatic behavior):

- Chosen each turn from a **weighted random** set (e.g. PhysAttack, ElmtAttack, Heal, Guard, Focus, Slack).
- Weights can be changed by skills (e.g. “Maximize primary” to force one action).
- Each action has targeting (e.g. single target), power (damage/heal values), and animation duration.

**Active actions** (player-chosen):

- Selected from the same or an extended set; when used, they can be **queued to run next** so they override the normal auto order.

**Examples:**

- **PhysAttack / ElmtAttack** — single-target damage (physical vs elemental).
- **Heal** — single-target heal.
- **Guard** — increases defense for a period.
- **Focus** — increases offense for a period.
- **Slack** — no combat effect (pass turn).

Damage and healing are subject to the relevant attack/defense multipliers.

---

## Hazards

- **Hazards** are placed on grid cells and last for a **duration** (number of ticks).
- Each tick they apply an **effect** (e.g. damage, defense down, heal) with a given **strength**.
- Types are defined in a central list (e.g. Damage, DefDown_10, DefDown_40, Heal) with duration and strength; specific instances can override strength.

---

## Teams & Reserve

- A **team** is a grid plus:
  - **Active** slots: monsters on the grid.
  - **Reserve**: an ordered list of monsters not on the grid.
- **Swap from reserve**: a reserve monster can replace an active one at the same cell; the previously active monster goes into reserve at the swapped position.
- **Auto-swap**: when an active monster dies, it can be replaced by the next alive monster in reserve (e.g. front of list), so combat can continue.

---

## Obstacles

- **Obstacles** occupy cells and can have:
  - **HP** — reduced by damage; when they reach 0, the obstacle is destroyed.
  - **Duration** — counts down each tick; when it reaches 0, the obstacle is destroyed.
- A cell is blocked for movement while its obstacle is not destroyed.

---

## Summary

| Concept        | Role |
|----------------|------|
| **Grid**       | 3×3 (or similar) per side; cells hold monsters, hazards, obstacles. |
| **Tick**       | Fixed timestep; drive queue execution and hazard ticks. |
| **Action queue** | FIFO; front action runs each tick; active actions can be inserted at front. |
| **Monsters**   | HP, AG, EG, phys/element attack and defense multipliers; auto + active actions. |
| **Hazards**    | Duration + effect (damage/heal/debuff) per tick on a cell. |
| **Teams**      | Active grid slots + ordered reserve; swap and auto-swap on death. |
| **Obstacles**  | Block cells until destroyed (by HP or duration). |

This document is a **design overview** for the combat system. Implementation details and source are not included here.
