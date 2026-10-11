# Plugins catalogue

[← Studies](./README.md) · [Creator app →](../13-creator-app.md)

Every plugin named across the docs, in one place — the base engine, the combat example, and the
three genre studies. Each entry lists its **slot**, its **origin**, what it does, and the
**key params** (the tunable variables a creator would set). This catalogue is also the
**machine-readable manifest** a partner LLM agent reads when helping a creator — see
[§5](#5-machine-readable-manifest-sketch) and the [creator app](../13-creator-app.md).

> **Slot** = the registry key (`loop`, `scene`, `resolver`, …). **Origin**: `base` = the
> generalized source engine · `combat` = the [worked example](../11-worked-example-combat.md) ·
> `onsen` / `rpg` / `detective` = the studies.

---

## 1. Kernel & gateways

| Plugin (slot) | Origin | Responsibility | Key params |
|---|---|---|---|
| `Clock` | base | Time model (slots/phases/rounds) | slot count, phase names, start date |
| `World` | base | Entities + components; record/overlay | entity schemas |
| `EventBus` | base | Domain events | (declared event types) |
| `Rng` | base | Named reproducible streams | stream names, seeds |
| `SaveProvider` | base | Snapshot + journal | save slots, autosave cadence |
| `LlmGateway` (+ `LlmMiddleware`) | base | Structured calls | provider, endpoint, model, thinking |
| `MediaGateway` | base | Image/audio generation | provider, workflows, sizes |
| `ContentProvider` | base | Content packs | pack list, load order |

## 2. Engine slots (base)

| Plugin (slot) | Origin | Responsibility | Key params |
|---|---|---|---|
| **SceneEngine** | base | Play a scene | persona, blocks, schema, cacheKey, route |
| **IntentEngine** | base | Free text → intent | roster, `actionKind` registry, thinking floor |
| **ResolverEngine** | base | Result → deltas | schema, model, sections |
| **CastEngine** | base | Who is present | cast strategies, slot caps |
| **PresenterEngine** | base | Beats → view | layout, transitions, audio table |
| **ModifierPlugin** | base | Adapt everything | applies-when, hook order |
| **System** | base | Tick/event side-systems | tick cadence, subscriptions |
| **LoopPlugin** | base | Host a mode | priority, `canEnter`, time cost |

## 3. Genre plugins

### Loop skeletons (`loop`, code)

The reusable shapes. Genres **configure** these; they rarely add a new one.

| Skeleton | Shape | Used by |
|---|---|---|
| `ScheduleLoop` | phases over a calendar: open → phases → resolve → advance | VN, onsen, overworld RPG, investigation day |
| `RoundLoop` | turn order until a win/lose condition | combat, exams, minigames |
| `GraphLoop` | rooms/nodes + exits until an exit condition | dungeons, investigation maps |
| `MenuLoop` | nested menus until a choice | shops, crafting, character screens |
| `CutsceneLoop` | non-interactive beats | intros, endings, story beats |

### Loop configs (data — *not* plugins)

A genre's "loop" is a skeleton **config**: a phase/node list + which systems tick + which
resolver runs. `VNLoop` and `OnsenLoop` are therefore **not** separate plugins — they are two
`ScheduleLoop` configs.

| Config | Skeleton | Phases / shape | Resolver cadence |
|---|---|---|---|
| **VN (Venus)** | `ScheduleLoop` | Day, Night | per phase (slot) |
| **Onsen** | `ScheduleLoop` | Dawn … Night (5) | per day (`DayLedger`) |
| **Overworld RPG** | `ScheduleLoop` | Morning … Night (4) | per day |
| **Investigation day** | `ScheduleLoop` | hours until the case cools | per day (`ClueResolver`) |
| **Combat** | `RoundLoop` | initiative rounds | on exit (`CombatResolver`) |
| **Dungeon** | `GraphLoop` | rooms + exits | on room enter |
| **Interrogation** | `RoundLoop` | press rounds | on exit |
| **Deduction** | `MenuLoop` | assemble → accuse | on submit (`TheoryResolver`) |
| **Shop / crafting** | `MenuLoop` | menus | on exit |
| **Cutscene** | `CutsceneLoop` | beats | — |

**Key params of a `ScheduleLoop` config:** `phases[]` (`id`, `name`, `sceneKind`, `activities`,
`actions`, `pre`-systems, `resolve`), resolver cadence, save cadence.

### Scenes (`scene`)

| Plugin | Origin | Responsibility | Schema |
|---|---|---|---|
| `SceneEngine` (dialogue) | base | VN scenes | `scene` |
| `SoloSceneEngine` | base | One-shot narration | `scene` |
| `CombatSceneEngine` | combat | Combat beats | `combatRound` |
| `ServiceSceneEngine` | onsen | Guest/staff scenes | `serviceScene` |
| `DialogueSceneEngine` | rpg | Story dialogue | `dialogueNode` |
| `ExplorationSceneEngine` | rpg | Describe an area | `explore` |
| `InterrogationSceneEngine` | detective | Question/press | `interrogation` |
| `SearchSceneEngine` | detective | Examine a place | `search` |

### Intents (`intent`)

| Plugin | Origin | `actionKind` registry |
|---|---|---|
| `VnIntent` | base | goto/class/job/project/people |
| `CombatIntent` | combat | attack/defend/item/flee |
| `ManagementIntent` | onsen | serve/clean/assign/upgrade/price/tend |
| `RpgIntent` | rpg | talk/travel/attack/use/skill/flee/examine/buy |
| `InvestigateIntent` | detective | search/examine/question/press/tail/accuse |

### Resolvers (`resolver`)

| Plugin | Origin | Responsibility | Key params |
|---|---|---|---|
| `SceneLedger` | base | Relationships/memories/money/stats | memory points, recency weights |
| `TextLedger` | base | Texting summaries | window, budget |
| `CombatResolver` | combat | HP/injury/loot/XP | damage formula, crit, XP curve |
| `DayLedger` | onsen | The day's books | wage/upkeep/tax rates |
| `QuestResolver` | rpg | Objective state | objective types |
| `LootResolver` | rpg | Drop tables | drop rates, rarity |
| `ClueResolver` | detective | Evidence/leads | find chance, certainty |
| `TheoryResolver` | detective | Accusation verdict | proof rules |

### Systems (`system`)

| Plugin | Origin | Responsibility | Key params |
|---|---|---|---|
| `TextingSystem` | base | DMs, hangouts, blocking | reply floor, post chance |
| `FeedSystem` | base | Status posts, likes | post rate, like spread |
| `JobsSystem` | base | Shifts, wages, raises | shift length, pay |
| `ProjectsSystem` | base | Coursework progress | work needed, deadlines |
| `GradesSystem` | base | Standing, exams | weightings |
| `OccasionsSystem` | base | Holidays/festivals | calendar |
| `WeatherSystem` | base | Weather | table, transitions |
| `RumorSystem` | base | Per-slot hooks | rumor rate |
| `BookingSystem` | onsen | Reservations/occupancy | capacity, no-show rate |
| `HousekeepingSystem` | onsen | Cleanliness decay | decay rate, threshold |
| `StaffSystem` | onsen | Shifts/morale/skills | morale decay, quit threshold |
| `EconomySystem` | onsen | Prices/takings/upkeep | price list, upkeep |
| `ReputationSystem` | onsen | Reviews → arrivals | review weight, decay |
| `GuestFlowSystem` | onsen | Who arrives | arrival curve |
| `SeasonSystem` | onsen | Seasons/festivals | calendar, effects |
| `EncounterSystem` | rpg | Roll encounters | rate, tables |
| `SpawnSystem` | rpg | Populate/respawn | density, timers |
| `TimeSystem` | rpg | World clock/schedules | day length, schedules |
| `QuestSystem` | rpg | Quest lifecycle | states, rewards |
| `FactionSystem` | rpg | Reputation/alliances | rep scale, hostility |
| `CraftingSystem` | rpg | Recipes/upgrades | recipes, materials |
| `StatusEffectSystem` | rpg | Buffs/debuffs | effects, durations |
| `EvidenceSystem` | detective | The case file | certainty, links |
| `AlibiSystem` | detective | Alibis/contradictions | break rules |
| `SuspectSystem` | detective | Whereabouts/secrets | schedules |
| `TimelineSystem` | detective | Event order | conflict rules |
| `PressureSystem` | detective | Case clock | cooling rate |

### Presenters (`presenter`)

| Plugin | Origin | Responsibility |
|---|---|---|
| `VnPresenter` | base | Stage, dialogue box, portraits |
| `CombatPresenter` | combat | HP/MP bars, turn order, targeting |
| `ManagementPresenter` | onsen | Room board, ledger, roster, stock |
| `CharacterSheetPresenter` | rpg | Attributes, level, skills |
| `InventoryPresenter` | rpg | Items, gold, equip/compare |
| `MapPresenter` | rpg | World/dungeon map |
| `QuestLogPresenter` | rpg | Quests and objectives |
| `CaseBoardPresenter` | detective | Evidence, suspects, timeline |

### Modifiers (`modifier`)

| Plugin | Origin | Responsibility | Key params |
|---|---|---|---|
| `ContentRating` | base | SFW text/visual swaps | rating |
| `PersonaOverride` | base | Custom system prompt | persona text |
| `AntiRepetition` | base | Seed word | word list |
| `TraitModifier` | base | Character traits | trait list |
| `WorldStateModifier` | base | Mood/weather/occasions | — |
| `ContextBudget` | base | Trim blocks to fit | budgets |
| `ViolenceModifier` | combat | Combat rules/directives | gore level |
| `SeasonModifier` | onsen | Seasonal blocks/assets | season |
| `GuestTraitModifier` | onsen | Guest/staff traits | trait list |
| `DifficultyModifier` | rpg | Scaling/damage rules | curve |
| `ClassModifier` | rpg | Per-class behavior | class list |
| `PermadeathModifier` | rpg | Death rules | on/off |
| `GenreModifier` | detective | Noir tone bundle | genre |
| `WithholdModifier` | detective | How much the LLM hides | withholding |

---

## 4. What a plugin declares (the contract)

For the [creator app](../13-creator-app.md) to expose a plugin **without code**, every plugin
declares a small manifest:

```ts
interface PluginManifest {
  slot: string                 // "resolver", "system", ...
  id: string                   // "CombatResolver"
  origin: string               // "combat"
  summary: string
  params: JsonSchema           // the tunable variables -> auto-generated settings form
  hooks?: string[]             // which modifier hooks it uses (beforePrompt, afterResolve, ...)
  directives?: string[]        // directives it adds to the vocabulary
  schemas?: string[]           // structured-output contracts it defines
  requires?: string[]          // other plugins/slots it needs
  events?: { emits: string[]; listens: string[] }
}
```

The **`params` JSON Schema** is the crux: it's what a creator edits (the "except some
variables"), what the app renders as a form, and what the partner agent fills in.

## 5. Machine-readable manifest sketch

An aggregated manifest (one file) is what the partner LLM agent is given as context. Sketch:

```json
{
  "engine": "venus-engine",
  "version": "1.0",
  "slots": ["loop","scene","intent","resolver","caster","presenter","modifier","system","content","llm","media","save"],
  "plugins": [
    {
      "slot": "resolver",
      "id": "CombatResolver",
      "origin": "combat",
      "summary": "Rounds -> HP/injury/loot/XP.",
      "params": {
        "type": "object",
        "properties": {
          "critChance":   { "type": "number", "default": 0.1, "min": 0, "max": 1 },
          "critMultiplier": { "type": "number", "default": 2 },
          "xpCurve":      { "type": "string", "enum": ["linear","quadratic"], "default": "quadratic" }
        }
      },
      "directives": ["damage","heal","cast","apply","summon","flee"],
      "schemas": ["combatRound"],
      "events": { "emits": ["actor:defeated"], "listens": [] }
    },
    {
      "slot": "system",
      "id": "EconomySystem",
      "origin": "onsen",
      "summary": "Prices, takings, upkeep, stock.",
      "params": {
        "type": "object",
        "properties": {
          "bathPrice":  { "type": "integer", "default": 800 },
          "dinnerPrice":{ "type": "integer", "default": 1500 },
          "wageRate":   { "type": "number", "default": 1.0 },
          "upkeepRate": { "type": "number", "default": 0.15 }
        }
      }
    }
  ]
}
```

A creator's pack, in the same spirit, is **data**:

```json
{
  "pack": "my-inn",
  "loop": { "skeleton": "ScheduleLoop", "config": "onsen" },
  "uses": ["DayLedger", "EconomySystem", "ReputationSystem", "HousekeepingSystem"],
  "params": {
    "ScheduleLoop.phases": ["dawn", "morning", "afternoon", "evening", "night"],
    "EconomySystem": { "bathPrice": 900, "wageRate": 1.1 },
    "ReputationSystem": { "reviewWeight": 0.6 }
  },
  "rules": [
    { "on": "guest:arrived", "when": "season == 'winter' && reputation < 20",
      "then": [{ "price": { "service": "bath", "delta": -0.1 } },
               { "narrate": "A rival inn is undercutting you." }] }
  ]
}
```

See the [creator app](../13-creator-app.md) for how these become a no-code UI + a partner agent.

---

[← Studies](./README.md) · [Creator app →](../13-creator-app.md)
