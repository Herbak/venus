# Venus Engine Design

**A genre-neutral, plugin/engine-centric framework for LLM-driven visual-novel + simulation
games**, reverse-engineered and generalized from **Venus University v0.4.0**
(Electron + React + LLM + ComfyUI).

The central idea:

> The source game is really **two primitives** wrapped in **five engines**.
> Build the primitives well, put the engines behind interfaces, and any genre system
> (combat, management, investigation, survival...) becomes another plugin on the same kernel.

```
CALL(persona, blocks[], schema) -> structured JSON    // every LLM task
REDUCE(events[]) -> deltas -> newState                // every state change
```

---

## Read this first

| # | Document | What it covers |
|---|----------|----------------|
| 01 | [Overview & concepts](./01-overview.md) | The mental model, architecture diagram, the five engines, glossary |
| 02 | [Kernel](./02-kernel.md) | Clock, world state, event bus, RNG, persistence, LLM/media gateways, plugin registry |
| 03 | [Scene engine](./03-scene-engine.md) | **The main engine** — play a scene (prompt → call → validate → apply) |
| 04 | [Intent engine](./04-intent-engine.md) | Free text → structured intent (the classifier) |
| 05 | [Resolver engine](./05-resolver-engine.md) | Narrative result → state deltas (the "ledger") |
| 06 | [Loop engine](./06-loop-engine.md) | Loops + parallel systems (VN loop, combat loop, texting, exams, jobs) |
| 07 | [Modifier engine](./07-modifier-engine.md) | Adapt to specificities (genre, SFW, traits, mood, combat) |
| 08 | [Presenter engine](./08-presenter-engine.md) | Beats/directives → screen; playback, transitions, audio |
| 09 | [Prompt system](./09-prompt-system.md) | Blocks, roster, schemas, model routing, middleware |
| 10 | [Content & mods](./10-content-and-mods.md) | Content packs, entities, custom content, ComfyUI setup |
| 11 | [Worked example: combat](./11-worked-example-combat.md) | Plugging a combat loop in parallel, end to end |
| 12 | [Roadmap & rules](./12-roadmap-and-rules.md) | Keep vs parameterize, phased migration, pitfalls |
| 13 | [The creator app](./13-creator-app.md) | No-code authoring: variables, rules, prompts, and a partner LLM agent |
| 14 | [VN-first + sub-systems](./14-vn-plus-systems.md) | The engine's scope: VN base + optional non-real-time kits |
| 15 | [Reference kernel (code)](./15-reference-kernel.md) | Runnable TypeScript skeleton of the kernel + five engines |
| 16 | [Schemas & prompts (full)](./16-schemas-and-prompts.md) | Every JSON Schema + the exact personas and blocks |
| 17 | [State & save format](./17-state-and-save.md) | The world state, flags, memories, and the persistence chain |
| 18 | [Image pipeline (ComfyUI)](./18-image-pipeline.md) | The base→derived workflows, poses, models |
| 19 | [Kits (pluggable format)](./19-kits.md) | The kit manifest + capability model + the kit catalogue |
| 20 | [Build a Venus (from scratch)](./20-build-a-venus.md) | Milestones + minimum kit set + content templates |
| 21 | [UI screen inventory](./21-ui-screens.md) | Every screen and presenter the base needs |
| A | [Appendix: source reference](./appendix-a-source-reference.md) | Enums, schemas, IPC surface, workflows, stores, character fields |

---

## Applied studies

Worked examples of building **different games** from these bricks — which existing engines to
reuse and which plugins to write.

| Study | Premise | Scope |
|-------|---------|-------|
| [Studies index](./studies/README.md) | — | How the studies are structured |
| [01 — Onsen management](./studies/01-onsen-management.md) | Run a hot-spring inn day by day | **A small step** from the campus sim (~70% reuse) |
| [02 — True RPG: combat + VN stories](./studies/02-rpg-combat-and-stories.md) | Explore, fight, level up, live story scenes | **The full showcase** — loop stack, multiple scene kinds, combat + RPG systems |
| [03 — Detective / mystery](./studies/03-detective-mystery.md) | Solve a case: investigate, interrogate, deduce | **Information as the verb** — evidence, alibis, a deterministic verdict |
| [Plugins catalogue](./studies/plugins-catalogue.md) | Every plugin named across the docs | The aggregate manifest, also read by the partner agent |

---

## The five engines at a glance

| Engine | Job | Source analog |
|--------|-----|---------------|
| **Scene** | Play a scene | `buildScenePrompt` → `streamScene` → `stepStage` |
| **Intent** | Free text → structured intent | `classifyAction` / `buildClassifierPrompt` |
| **Resolver** | Result → state deltas | `buildLedgerPrompt` → `applyLedger` / `resolveScene` |
| **Loop** | Host loops + scheduled systems | `beginSlot` … `crossSlotBoundary` |
| **Modifier** | Adapt everything to specifics | SFW toggles, traits, mood, weather, `scenePersona` |

Plus a **kernel** (clock, world, events, RNG, persistence, gateways) and a **content-pack**
system for mods.

---

## How to use this doc set

- Building a VN/sim from scratch? Read **01 → 02 → 03 → 05 → 06**, then **07**.
- Adding a second genre system (combat etc.)? Read **06**, **07**, then **11**.
- Extracting from the source game? Read **12** and the **appendix**.
- Want a concrete "build a different game" walkthrough? Read the [studies](./studies/README.md).
- Want a non-coder to build or remix a game? Read **13** and the [plugins catalogue](./studies/plugins-catalogue.md).
- Wondering whether the engine can make *your* game? Read **14** (the VN-first scope + kit catalogue).
- Actually building a Venus-like game? Read **20**, then **15 → 16 → 17 → 18 → 19 → 21**.
