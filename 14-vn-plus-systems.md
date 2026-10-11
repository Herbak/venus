# 14 — VN-first + optional sub-systems

[← Index](./README.md) · [Loop engine](./06-loop-engine.md) · [Creator app](./13-creator-app.md)

This chapter fixes the **scope the engine targets**: a **visual-novel core** with **optional,
non-real-time sub-systems** layered on. It answers one question —

> Is the loop model enough to make a variety of games this way?

— with **yes**, and it states exactly why, and where the boundary is.

---

## 1. The invariant

The **VN loop is always the base** — the bottom of the [loop stack](./06-loop-engine.md#2-the-loop-stack).
It is a [`ScheduleLoop`](./06-loop-engine.md#21-loop-skeletons) config: time advances in phases,
each phase plays a **VN scene**, and a **resolver** writes state at the boundary.

Everything else is **optional** and attaches **without changing that base**. A game is:

```
VN base  +  0..N sub-systems (each a "kit")
```

A dating sim has zero kits. An RPG has a combat kit, a quest kit, an inventory kit. An onsen sim
has a management kit. None of them replaces the VN loop.

## 2. The five attachment shapes

Every qualifying sub-system is one — or a combination — of these. If a proposed system fits one,
**it fits the engine as-is**:

| # | Shape | What it adds | Example |
|---|---|---|---|
| 1 | **Push a sub-loop** | a modal loop on top of the VN scene | combat (`RoundLoop`), dungeon (`GraphLoop`), inventory/shop (`MenuLoop`) |
| 2 | **Tick as a system** | state that changes alongside the VN | economy, time/weather, quests, factions, status effects |
| 3 | **Extend the scene** | new directives + a modifier | `damage:`, `evidence:`, `price:` — the LLM narrates the new verbs |
| 4 | **Add a resolver** | a second pass that writes that system's state | combat→HP/loot, case→evidence, inn→the books |
| 5 | **Add a presenter** | a UI panel for the system's state | HP bars, inventory, case board, ledger |

No new **engine category** is required — only new **implementations** of the existing slots.

## 3. The one design rule: decision → resolution → prose

The discipline that keeps sub-systems inside the model:

> A sub-system is **decision → resolution → prose**, never a continuous simulation.

- **Decision** — the player chooses (a menu, free text, a scene action).
- **Resolution** — a **deterministic** function (a resolver) computes the outcome: damage, a
  skill check, a resource total, a threshold.
- **Prose** — the scene engine narrates the outcome as beats + directives.

As long as every sub-system stays in that shape, the VN base, the loop stack, and the five
engines hold — and "a different game" is just **picking kits**.

## 4. The fidelity dial

A sub-system can be implemented at **any depth**, from "just a modifier on the VN scene" to "a
full pushed loop" — same game, different effort. Start light; promote only if it earns the
complexity.

| Sub-system | Light (no new loop) | Heavy (new loop) |
|---|---|---|
| **Combat** | a VN scene with a `combatRound` schema — the fight is narrative beats + `damage:` directives | a pushed `RoundLoop` with initiative, HP bars, a `CombatResolver` |
| **Inventory** | an `items` field the scene reads/writes | a `MenuLoop` + `InventoryPresenter` + shops |
| **Management** | scene-level choices + an economy modifier | a second `ScheduleLoop` config + economy systems |
| **Investigation** | clues as scene directives + a resolver | a `GraphLoop` case map + evidence systems |

The VN base never changes between the two columns.

## 5. The extension-kit catalogue

A **kit** is the bundle that implements one sub-system: a loop config (optional) + scene kind
(optional) + `actionKind`s + a resolver + systems + a presenter + modifiers + content + the
directives/schemas it adds. Kits are the unit a creator picks ("add combat").

| Kit | Loop (optional) | Resolver | Systems | Unlocks |
|---|---|---|---|---|
| **Relationship / romance** *(base)* | — | `SceneLedger` | schedules, occasions | dating sim, drama, harem |
| **Combat** | `RoundLoop` | `CombatResolver` | status effects | RPG, tournament, war story |
| **Management / economy** | `ScheduleLoop` config | `DayLedger` | economy, bookings, staff | onsen, shop, estate, agency |
| **Investigation / evidence** | `GraphLoop` | `ClueResolver`, `TheoryResolver` | evidence, alibis, timeline | detective, mystery, spy |
| **Survival / crafting** | `MenuLoop` | `CraftResolver` | resources, recipes | castaway, base-building VN |
| **Quests / factions** | — | `QuestResolver` | quests, factions | open-world-ish RPG, politics |
| **Jobs / calendar** | — | — | shifts, projects, grades | life sim, school, career |
| **Horror / sanity** | — | `SanityResolver` | status, flags | horror, psychological thriller |
| **Time-loop / run-based** | `CutsceneLoop`/`RoundLoop` | `RunResolver` | flags, meta-progress | time-loop mystery, roguelite VN |

Every row is **"VN base + N kits"**, and every kit is the five shapes. Variety is bounded by
**how many kits you write**, not by the loop model.

## 6. What you still need (not a new engine)

Sub-systems have **rules math** — damage, hit chance, skill checks, resource totals, thresholds.
That lives in a **Resolver** (or a plain pure helper it calls). It's already a slot; you write a
small deterministic function per sub-system and keep it **outside the LLM**. No new engine
category.

## 7. Out of scope (and why)

| Excluded | Why |
|---|---|
| **Real-time / twitch** (action, platformer, shmup, racing) | the base advances in discrete phases/rounds; input is a decision, not a stream |
| **Physics-driven** | no continuous simulation substrate |
| **Rules-algebra minigames** (card battlers, engine-building games) | need a dedicated rules/effects engine (priority, triggers, zones) — a new engine, not a loop |
| **Multiplayer / online** | single-player, local saves, no authority/netcode |

A VN can still **abstract** these (a narrated battle, a menu-driven "card" choice), but the
engine doesn't simulate them.

## 8. Verdict

For **VN-first + non-real-time sub-systems**, the loop model is **sufficient**. The remaining
work is a **kit library** — systems, resolvers, presenters, and content per theme — not new loop
or kernel machinery.

See the [creator app](./13-creator-app.md) for how a non-coder picks and tunes kits, and the
[plugins catalogue](./studies/plugins-catalogue.md) for the concrete plugins each kit contains.

---

[← Index](./README.md) · [Loop engine](./06-loop-engine.md) · [Creator app](./13-creator-app.md)
