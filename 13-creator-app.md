# 13 — The creator app (no-code authoring)

[← Index](./README.md) · [Plugins catalogue](./studies/plugins-catalogue.md)

How a **game creator who doesn't code** builds, tunes, and remixes a game on this engine — and
how a **partner LLM agent** helps. The guiding rule:

> **Plugins are code; everything a creator changes is data.**
> If the engine exposes its knobs as *data + declared hooks*, then the creator app is a **data
> editor, a rule builder, and a prompt/schema editor** — with a partner agent that writes the
> same data for you.

This is why the [plugins catalogue](./studies/plugins-catalogue.md) matters: every plugin
publishes a **`params` JSON Schema** (its variables) and a **manifest** (its hooks, directives,
schemas, events). Those manifests are what the app renders and what the agent reads.

---

## 1. What a creator actually touches

| Layer | What it is | How the creator edits it | Backed by |
|---|---|---|---|
| **Content** | actors, places, items, quests, factions | forms / tables / CSV / import | `ContentProvider` |
| **Variables** | each plugin's `params` | **auto-generated settings forms** | plugin `params` JSON Schema |
| **Prompts & personas** | the LLM voice + blocks | a block editor (drag/reorder) | `PromptSpec`, block library |
| **Schemas** | structured-output contracts | a visual JSON-Schema builder | `JsonSchema` |
| **Rules** | "when X then Y" behavior | a **trigger → condition → action** builder | compiles to modifier/system hooks |
| **Flow** | **loop skeleton + schedule**, dialogue, scenes | pick a skeleton, edit its **phase schedule**; a node/edge editor for graphs | `ScheduleLoop`, scene kinds |
| **Assets** | art, audio, workflows | the media gateway (generate/upload) | `MediaGateway` |
| **Packaging** | the whole thing | export/import a **content pack** | `ContentProvider` |

Everything above is **data**. No step requires code — and every step can also be done by the
partner agent.

> The unit a creator actually picks and tunes is a **kit** — a bundle implementing one
> sub-system (combat, management, investigation…). See
> [14 §5 — the extension-kit catalogue](./14-vn-plus-systems.md#5-the-extension-kit-catalogue).

## 2. Capability tiers

Let the creator stop wherever they're comfortable:

| Tier | Name | The creator does | Example |
|---|---|---|---|
| **0** | No-code | set variables, write prompts, build rules, add content | "bath costs 900; a rival undercuts me in winter" |
| **1** | Low-code | write a small **expression** inside a rule | `damage = attack * (1 + crit ? 1.5 : 0) - defense` |
| **2** | Full code | write a real plugin | a bespoke `CombatResolver` |

Tier 0 is the target ("an app where the creator isn't coding except some variables"). Tiers 1–2
are escape hatches the app scaffolds and the agent can draft.

## 3. Using a plugin vs. modifying a plugin

The catalogue's plugins are **used** and **modified** in four ways:

| Way | No-code? | How |
|---|---|---|
| **Configure** | ✅ | Set the plugin's `params` (auto-form from its schema). |
| **Compose** | ✅ | Enable/disable plugins; order the modifier chain; stack loops. |
| **Extend** | ✅ | Write **rules** that compile into the plugin's hooks (`beforePrompt`, `afterResolve`, `onEvent`). |
| **Customize** | ✅ | Rewrite its prompts/personas and its output schema. |
| **Derive** | ✅ | Duplicate a plugin's data and tweak it (a "rival inn" = a copy of the guest flow with different params). |
| **Fork** | ⚠️ Tier 1–2 | Write a new implementation for the slot (rare; the app scaffolds it). |

### 3.1 The loop is a choice + a schedule, not code

A creator never writes a "genre loop". They **pick a skeleton** and edit its **schedule**:

- **`ScheduleLoop`** → edit the **phase list** (add "afternoon", rename "night", set actions per
  phase). This is the one most games need — Venus, onsen, and overworld RPG are all the *same*
  loop with different phases.
- **`RoundLoop`** → combat/exams (edit initiative + win conditions).
- **`GraphLoop`** → dungeons / dialogue trees (edit the node graph).
- **`MenuLoop`** → shops / crafting.

So "make an onsen game" = pick `ScheduleLoop`, set five phases, and add the management systems —
a data edit, exactly what the [onsen study](./studies/01-onsen-management.md) does. The phase
schedule is just another **params form**.

## 4. The rule DSL (the no-code way to extend behavior)

A **rule** is trigger → condition → action. It compiles to the engine's hooks, so it's a
first-class modifier/system without the creator writing one.

```jsonc
{
  "on": "guest:arrived",                       // an EventBus event, or a loop hook
  "when": "season == 'winter' && reputation < 20",
  "then": [
    { "price":   { "service": "bath", "delta": -0.1 } },
    { "narrate": "A rival inn is undercutting you." },
    { "memory":  { "actor": "$guest", "type": "disliked", "desc": "found the inn pricey" } }
  ]
}
```

Compiles to:

```ts
const Rule_underCut: ModifierPlugin = {
  id: "rule:under-cut",
  applies: () => true,
  onEvent(e, world) {
    if (e.type !== "guest:arrived") return;
    if (!(world.season === "winter" && world.reputation < 20)) return;
    world.economy.price("bath", -0.1);
    world.narrate("A rival inn is undercutting you.");
    world.memory(e.guest, "disliked", "found the inn pricey");
  }
}
```

**The builder's palette** = the catalogue: every `emits`/`listens` event, every directive, every
param, every entity field becomes a dropdown. That's the whole trick — the app never invents
vocabulary; it exposes the plugins'.

## 5. The partner LLM agent

An optional agent that **writes the same data the UI edits**, so the creator can always see and
change it.

**What the agent is given as context:**
- the **plugins catalogue manifest** (slots, plugins, `params` schemas, directives, schemas, events);
- the creator's current **pack** (content + params + rules + prompts);
- the engine's **style guide** (the `CALL`/`REDUCE` primitives, the block library).

**What it produces (always as validated data):**
- **Content** — actors, places, items, quests, factions.
- **Variables** — proposed `params` values (balance).
- **Prompts & schemas** — personas, blocks, output contracts.
- **Rules** — the trigger/condition/action JSON above.
- **Scaffolds** — a starting pack for a genre ("make me an onsen", "add a combat system").

**Modes:**

| Mode | Creator says | Agent returns |
|---|---|---|
| **Scaffold** | "I want an onsen management game" | a pack: `uses` + params + seed content + rules |
| **Add system** | "add turn-based combat" | combat plugins enabled + params + a combat persona + rules |
| **Write content** | "six quirky guests" | six `Actor` records conforming to the schema |
| **Balance** | "combat is too easy" | tweaked `CombatResolver` params |
| **Voice** | "make the innkeeper a grumpy old man" | a rewritten persona block |
| **Explain / fix** | (pastes a validation error) | the corrected data + why |

**Guardrails:**
- The agent only emits **data that validates** against the plugins' schemas; the app rejects the
  rest and shows the creator a plain-language error.
- **No arbitrary code** — Tier 0–1 output only (data + expressions).
- **Diff & preview** before applying; nothing changes until the creator accepts.
- The agent is a **helper, not a dependency** — everything it does is editable by hand.

## 6. Validation & preview

- **Schema validation** on every artifact (content, params, schemas, rules).
- **A sandbox playtest** — the app runs the engine against the pack in a preview pane (the
  [loop stack](./06-loop-engine.md) in a test harness), so the creator sees the game.
- **Creator-facing errors** — map engine errors to creator terms ("`CombatResolver.critChance`
  must be between 0 and 1", not a stack trace).

## 7. The source already has the seeds

The creator app isn't new in spirit — the source game already exposes a small authoring surface:

| Seed in the source | Becomes |
|---|---|
| Settings that are variables (sampling, memory budgets, `scenePersona`, model routing) | the **params** panel |
| `SystemPromptModal` (edit a prompt) | the **prompt/persona editor** |
| Custom backgrounds / outfits / CGs / rooms | the **asset manager** |
| Created scenes (`useSceneCreatorStore`) + `scenes:*` | the **scene/flow editor** |
| Grab bags (random pools) | the **pool editor** |
| Import/export (`backup:*`, `chars:*`) | **pack export/import** |
| `setup:*` (ComfyUI install) | the **asset pipeline** setup |

The creator app is these seeds, generalized and made data-driven.

## 8. How the app itself maps to the engine

Eat the dogfood — the app is built from the same primitives:

- **A `CreatorMode` presenter** — the editors (params forms, rule builder, flow graph).
- **The engine runs in a sandbox** — a headless `World` + the loop stack, no window.
- **The pack is the artifact** — the app reads/writes a `ContentPack` folder (see
  [Content & mods](./10-content-and-mods.md#2-content-pack-layout)).
- **The partner agent is an `LlmGateway` consumer** — the same gateway + middleware, with the
  catalogue manifest in the prompt.

## 9. Build order

1. **Params panel** — render every plugin's `params` schema as a form. (Biggest single win.)
2. **Content editor** — CRUD forms for entities + CSV/JSON import.
3. **Prompt/persona editor** — the block library, reorderable.
4. **Rule builder** — trigger/condition/action, compiling to hooks; palette from the catalogue.
5. **Schema builder** — visual JSON-Schema editing (or agent-generated).
6. **Sandbox preview** — run a pack headless.
7. **Packaging** — export/import packs.
8. **Partner agent** — wire the catalogue manifest + a validation loop.

Steps 1–2 alone turn the engine into something a non-coder can reskin. Steps 4 and 8 are what
let them **add systems** (combat, economy) without writing a plugin.

---

[← Index](./README.md) · [Plugins catalogue](./studies/plugins-catalogue.md)
