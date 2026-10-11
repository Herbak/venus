# 19 — Kits (the pluggable format)

[← Image pipeline](./18-image-pipeline.md) · [Index](./README.md) · [VN-first scope](./14-vn-plus-systems.md) · [Next: Build a Venus →](./20-build-a-venus.md)

A **kit** is a self-contained, optional bundle implementing one sub-system. This chapter is the
**format** (how kits plug in) plus the **catalogue** of kits a Venus-like game needs. See also
[14 §5 — the extension-kit catalogue](./14-vn-plus-systems.md#5-the-extension-kit-catalogue).

The engine loads **only the kits a game lists** — some systems, not all, or entirely new ones.

---

## 1. The kit manifest

```ts
interface KitManifest {
  id: string                        // "combat", "management", "base-romance"
  name: string
  version: string
  requires: Capability[]            // shared services it builds on
  conflicts: string[]               // other kits it can't coexist with
  provides: Capability[]            // what it offers others
  registers: {
    loops?: LoopConfig[]            // skeleton configs it adds
    scenes?: string[]               // scene kinds
    actionKinds?: string[]          // intents it handles
    resolvers?: string[]            // resolver passes
    systems?: string[]              // tick/event systems
    presenters?: string[]           // UI panels
    modifiers?: string[]            // modifier-chain plugins
    directives?: string[]           // new beat directives
    schemas?: string[]              // structured-output contracts
    content?: string[]              // content pack folders
  }
  state: { namespace: string; schema: JsonSchema }   // state it OWNS
  events: { emits: string[]; listens: string[] }
  params: JsonSchema                // the tunables (creator-app form)
  entry: { triggers: string[] }     // events/actionKinds that start it
}
```

## 2. Capabilities (the shared substrate)

Kits never depend on each other — only on **capabilities**:

| Always-on (core) | Opt-in (shared) |
|---|---|
| `clock` `events` `rng` `world` `save` `scene` `intent` `resolver` `presenter` `modifier` | `stats` `resources` `inventory` `relationships` `time`/`schedule` `economy` `reputation`/`factions` `status-effects` `places`/map |

So `combat requires ["stats","inventory"]`; `management requires ["economy","time"]`;
`base-romance requires ["relationships","time","schedule"]`. If a game omits a required
capability, the loader says so in plain terms — or a kit ships a **minimal fallback** provider.

## 3. Load rules

- **Optional** — registered only if listed; unlisted kits cost nothing.
- **Dependency-checked** — `requires` resolved; `conflicts` refused with a clear message.
- **Namespaced** — a kit owns only `state.namespace`; it never reads another kit's internals.
- **Decoupled** — kits talk via the **event bus**, **deltas**, and capabilities only.
- **Swappable** — two combat kits `conflicts`; enable one.
- **Open** — "even others" = anyone authors a new kit (or the [creator app](./13-creator-app.md)
  rule builder does).

A game's pack:

```jsonc
{
  "pack": "my-venus-like",
  "loop": { "skeleton": "ScheduleLoop", "config": "campus" },
  "kits": ["base-romance", "characters", "texting", "classes", "jobs", "occasions", "social-sim"],
  "params": { "classes": { "difficultySpread": "medium" } }
}
```

## 4. The base Venus kit set

A Venus-like game = these kits. Each is optional; omit any and the game still runs.

| Kit | requires | provides | Registers (short) |
|---|---|---|---|
| **`base-romance`** | `relationships`, `time`, `schedule` | `vn-base` | `ScheduleLoop` config, `SceneEngine` (vn), `SceneLedger`, `VnIntent`, `VnPresenter`, the personas |
| **`characters`** | `relationships` | `actors` | character-gen scene, `CharacterStore`, the ComfyUI pipeline, manage-characters presenter |
| **`texting`** | `relationships`, `schedule` | — | `TextingSystem`, `TextLedger`, hangout classifier, Bunnyboard presenter |
| **`classes`** | `time`, `schedule` | `quests` | class/project/exam systems, `QuizResolver`, enrollment |
| **`jobs`** | `time`, `economy?` | — | shift system, wages, boss chat |
| **`occasions`** | `time` | — | holidays, weather, weekends, spring break |
| **`social-sim`** | `relationships` | — | rumors, crush settle, friendships, NPC overlay |
| **`photos`** | `relationships` | — | photo capture/persist/export |
| **`endings`** | `relationships`, `time` | — | graduation, epilogue, farewells, ending art |

### 4.1 `base-romance` (the VN base) — manifest

```jsonc
{
  "id": "base-romance", "name": "Base Romance (VN)", "version": "1.0",
  "requires": ["relationships", "time", "schedule"],
  "conflicts": [],
  "provides": ["vn-base"],
  "registers": {
    "loops": [{ "skeleton": "ScheduleLoop", "config": "campus",
                "phases": ["day", "night"], "resolve": "per-phase" }],
    "scenes": ["vn"],
    "actionKinds": ["goto", "talk", "people", "idle"],
    "resolvers": ["SceneLedger"],
    "systems": ["TimeSystem", "WeatherSystem"],
    "presenters": ["VnPresenter"],
    "modifiers": ["ContentRating", "PersonaOverride", "AntiRepetition", "ContextBudget"],
    "directives": ["show", "hide", "sprite", "cg", "bg"],
    "schemas": ["scene", "ledger", "slotIntro"],
    "content": ["settings", "backgrounds", "sound"]
  },
  "state": { "namespace": "romance", "schema": { "type": "object" } },
  "events": { "emits": ["slot:opened", "scene:ended"], "listens": [] },
  "params": {
    "type": "object",
    "properties": {
      "phases": { "type": "array", "items": { "type": "string" }, "default": ["day", "night"] },
      "timeModel": { "type": "string", "enum": ["slots", "continuous"], "default": "slots" },
      "durationBuckets": { "type": "object",
        "default": { "instant": 0, "short": 15, "medium": 30, "long": 60, "very_long": 120 } },
      "persona": { "type": "string" },
      "lessNsfwText": { "type": "boolean", "default": false }
    }
  },
  "entry": { "triggers": ["loop:start"] }
}
```

### 4.2 `texting` — manifest (abridged)

```jsonc
{
  "id": "texting", "name": "Bunnyboard (texting)", "version": "1.0",
  "requires": ["relationships", "schedule"], "provides": [],
  "registers": {
    "scenes": ["texting", "hangout-classifier"],
    "actionKinds": [],
    "resolvers": ["TextLedger"],
    "systems": ["TextingSystem", "FeedSystem", "HangoutSystem"],
    "presenters": ["BunnyboardPresenter"],
    "schemas": ["texting", "hangoutClassifier"],
    "content": ["feed"]
  },
  "state": { "namespace": "bunnyboard", "schema": { "type": "object" } },
  "events": { "emits": ["text:received", "hangout:proposed"], "listens": ["slot:opened"] },
  "params": { "type": "object", "properties": {
    "replyFloorMs": { "type": "integer", "default": 3000 },
    "postChance":   { "type": "number", "default": 0.1 }
  } },
  "entry": { "triggers": ["slot:opened"] }
}
```

### 4.3 `classes` — manifest (abridged)

```jsonc
{
  "id": "classes", "name": "Classes & exams", "version": "1.0",
  "requires": ["time", "schedule"], "provides": ["quests"],
  "registers": {
    "scenes": ["class", "exam"],
    "actionKinds": ["goto_class", "project"],
    "resolvers": ["QuizResolver", "GradeResolver"],
    "systems": ["ClassSystem", "ProjectSystem", "GradeSystem", "EnrollmentSystem"],
    "presenters": ["EnrollmentPresenter", "GradesPresenter"],
    "schemas": ["exam_quiz", "classFact"],
    "content": ["classes", "professors"]
  },
  "state": { "namespace": "academics", "schema": { "type": "object" } },
  "events": { "emits": ["class:attended", "exam:taken"], "listens": ["slot:opened"] },
  "params": { "type": "object", "properties": {
    "difficultySpread": { "type": "string", "enum": ["easy","medium","hard"], "default": "medium" },
    "weeksPerSemester": { "type": "integer", "default": 14 }
  } },
  "entry": { "triggers": ["action:goto_class"] }
}
```

## 5. Optional genre kits (add on top)

| Kit | requires | Registers |
|---|---|---|
| **`combat`** | `stats`, `inventory` | `RoundLoop` config, `CombatSceneEngine`, `CombatResolver`, `CombatPresenter`, `EncounterSystem`, `status-effects` modifier, directives `damage/heal/cast/flee`, schema `combatRound` |
| **`inventory`** | `stats` | `MenuLoop` config, `InventoryPresenter`, `InventorySystem`, directives `give/loot` |
| **`quests`** | `reputation?` | `QuestSystem`, `QuestResolver`, `QuestLogPresenter`, schema `questState` |
| **`factions`** | `reputation` | `FactionSystem`, `FactionPresenter` |
| **`management`** | `economy`, `time` | `ScheduleLoop` config (multi-phase), `DayLedger`, `EconomySystem`, `BookingSystem`, `StaffSystem`, `ReputationSystem`, `ManagementPresenter` |
| **`investigation`** | `relationships`, `places` | `GraphLoop` config, `ClueResolver`, `TheoryResolver`, `EvidenceSystem`, `AlibiSystem`, `CaseBoardPresenter`, directives `evidence/alibi/accuse` |
| **`survival`** | `resources`, `inventory` | resource systems, `CraftingSystem`, `SurvivalPresenter` |
| **`horror`** | `relationships` | `SanityResolver`, `StatusSystem`, `sanity` modifier |
| **`time-loop`** | `time` | `RunResolver`, meta-progress system |

Enable them by adding to `kits`. Because they share capabilities and communicate via events,
`combat` and `investigation` can coexist without knowing about each other.

## 6. Authoring a new kit

1. Pick the **slots** it fills (loop/scene/intent/resolver/systems/presenter/modifier).
2. Declare **`requires`** (capabilities) and **`state.namespace`**.
3. Add its **directives** + **schemas** + **content**.
4. Expose its **`params`** (the auto-form).
5. List its **events** (`emits`/`listens`).
6. Register it — no core code changes.

The [creator app](./13-creator-app.md) renders the `params` form, and the **partner agent** can
scaffold the whole manifest from a description ("make a fishing kit").

---

**Next:** [20 — Build a Venus](./20-build-a-venus.md).
