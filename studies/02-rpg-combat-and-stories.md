# Study 02 — A true RPG (combat + VN stories)

[← Studies](./README.md) · [Onsen →](./01-onsen-management.md)

## 1. Pitch

A full RPG: explore a world, fight, level up, gear up, take quests — **and** live story scenes
with a cast, in the same second-person, prose-driven style as the source. The VN engine becomes
the **story/dialogue layer**; combat and the RPG systems are **additional loops and systems on
the same kernel**. Nothing about the VN half is thrown away.

This is the engine's **full showcase**: the loop stack, multiple scene kinds, multiple
resolvers, and a directive vocabulary that spans dialogue *and* violence.

## 2. What changes vs the source

| Source (campus VN) | RPG |
|---|---|
| 2 slots/day, no combat | **world clock** (time-of-day) + **combat rounds** |
| One scene kind (dialogue) | **many**: dialogue, exploration, combat, shop, cutscene, skill-check |
| Characters = romance cast | **party · NPCs · monsters · factions** |
| Stats brain/body/heart | **attributes** (STR/DEX/INT…), **HP/MP**, **level/XP**; relationships kept for the story layer |
| One resolver (ledger) | **several**: VN ledger, combat resolver, quest resolver, loot resolver |
| One loop | a **loop stack**: world/exploration ↔ dialogue ↔ combat ↔ menus |
| Affection | affection **and** reputation (factions), still weighted memories |

The critical continuity: **the same `Actor` entity carries both** the VN layer (memories, flags,
relationships) **and** the RPG layer (stats, HP, inventory). A single entity, two views.

## 3. Bricks mobilized (and how)

| Brick | Reuse | How |
|---|---|---|
| **Kernel — Clock** | ✅ extended | World time-of-day for the story; combat keeps its own **round counter** (the clock is pluggable per loop). |
| **Kernel — World** | ✅ | Same record+overlay; entities gain components (stats, inventory, quest). |
| **Kernel — EventBus** | ✅ **key** | `combat:started`, `actor:defeated`, `quest:updated`, `faction:hit` — systems subscribe. |
| **Kernel — Rng** | ✅ **key** | Deterministic combat rolls, loot tables, skill checks — named streams. |
| **Kernel — Persistence** | ✅ | Slot saves now carry character sheet + quest state; journal deltas per plugin id. |
| **Kernel — LlmGateway + middleware** | ✅ unchanged | Same provider/auth/retry/schema-downgrade. |
| **Kernel — MediaGateway** | ✅ | Portraits, monster art, item icons, maps, CG illustrations. |
| **Scene engine** | ✅ **generalized** | Used for **dialogue AND combat AND exploration** — each a `SceneEngine` with its own schema/directives/presenter. |
| **Intent engine** | ✅ **core** | Free text → `actionKind` from an **open registry**: `talk`, `travel`, `attack`, `use`, `skill`, `flee`, `examine`, `steal`, `buy`… |
| **Resolver engine** | ✅ **multiplied** | One resolver per domain: VN ledger (relationships), combat (HP/loot), quest (objectives), shop (economy). |
| **Loop engine** | ✅ **the star** | The **loop stack** is exactly how combat runs "in parallel": push the combat `RoundLoop`, run, return deltas. |
| **Modifier engine** | ✅ | Difficulty, class/race, rating, permadeath, genre — as modifier bundles. |
| **Presenter engine** | ✅ **multiplied** | VN presenter + **CombatPresenter** (HP bars, turn order) + **CharacterSheet/Inventory/Map/QuestLog** presenters. |
| **Content packs** | ✅ | classes, races, skills, items, monsters, dungeons, quests, maps, factions. |
| **The directive model** | ✅ **the bridge** | `show:`/`sprite:` for dialogue, `damage:`/`cast:` for combat — one pipeline. |
| **The ledger pattern** | ✅ **the pattern** | "Writer writes prose; resolver writes state" applied to combat, quests, loot. |

**Net:** the source contributes the **kernel, the scene pipeline, the resolver pattern, the
directive model, the modifier chain, and the loop host**. The RPG adds **loops, systems, and
content** on top — no kernel surgery.

## 4. New plugins / engines to create

| Slot | Plugin | Responsibility |
|---|---|---|
| `loop` | **`ScheduleLoop` config** (overworld) | The world day: Morning … Night — travel, talk, quests. *(Not a new loop.)* |
| `loop` | **`RoundLoop` config** (combat) | Turn-based combat. Pushed onto the stack by an encounter. |
| `loop` | **`GraphLoop` config** (exploration) | Travel the map / crawl a dungeon: rooms, exits, loot, traps, encounters. |
| `loop` | **`GraphLoop` config** (dialogue tree) | Branching story conversations (authored nodes + LLM prose). |
| `loop` | **`MenuLoop` config** (shop/craft) | Buy/sell/craft menus. |
| `loop` | **`CutsceneLoop`** (skeleton) | Non-interactive story beats. |
| `scene` | **CombatSceneEngine** | Combat beats → `attack/defend/skill/item/flee` + HP deltas. Schema `combatRound`. |
| `scene` | **DialogueSceneEngine** | Dialogue beats; schema `dialogueNode` (speaker, text, choices, flags). |
| `scene` | **ExplorationSceneEngine** | Describe a room/area; schema `explore` (description, exits, points of interest). |
| `intent` | **RpgIntent** | The open `actionKind` registry + a **CastStrategy** per kind. |
| `resolver` | **CombatResolver** | Rounds → HP/injury/loot/XP. |
| `resolver` | **QuestResolver** | Objectives → quest state transitions. |
| `resolver` | **LootResolver** | Drop tables, gold, item acquisition. |
| `system` | **EncounterSystem** | Roll encounters (map/region/quest-driven) → push the combat `RoundLoop`. |
| `system` | **SpawnSystem** | Populate dungeons/regions; respawn timers. |
| `system` | **TimeSystem** | World clock, day/night, NPC schedules, quest timers. |
| `system` | **QuestSystem** | Quest lifecycle: offered → active → complete; rewards. |
| `system` | **FactionSystem** | Reputation, alliances, prices, hostility. |
| `system` | **CraftingSystem** | Recipes, materials, upgrades. |
| `system` | **StatusEffectSystem** | Buffs/debuffs/poison — on `Actor`. |
| `presenter` | **CombatPresenter** | HP/MP bars, turn order, targeting. |
| `presenter` | **CharacterSheetPresenter** | Attributes, level, skills, equipment. |
| `presenter` | **InventoryPresenter** | Items, gold, equip/compare. |
| `presenter` | **MapPresenter** | World/dungeon map, travel. |
| `presenter` | **QuestLogPresenter** | Active/completed quests, objectives. |
| `modifier` | **DifficultyModifier** | Enemy scaling, damage rules, save-scumming rules. |
| `modifier` | **ClassModifier** | Per-class skills/behaviors. |
| `modifier` | **PermadeathModifier** | Death rules (`afterResolve` on the combat resolver). |
| `modifier` | **RatingModifier** | Reuse the content-rating chain. |
| `content` | **RPG pack** | classes, races, skills, items, monsters, dungeons, quests, maps, factions. |

## 5. New directives, schemas, entities

**Directives** (one vocabulary, spanning both halves):

```
dialogue:  show:  hide:  sprite:  bg:  cg:            (reused from the VN engine)
combat:    damage:  heal:  cast:  apply:  summon:  flee:
world:     move:  open:  loot:  flag:  quest:  give:  roll:
```

**Schemas**: `combatRound`, `skillCheck`, `characterSheet`, `inventoryChange`, `questState`,
`dialogueNode`, `loot`, `explore`.

**Entities**: `Actor` (party/NPC/monster), `Class`, `Race`, `Skill`, `Item`, `Equipment`,
`Monster`, `Dungeon`/`Area`, `Quest`, `Faction`, `Map`, `EncounterTable`.

## 6. Loop diagram (the stack at work)

```
GraphLoop (exploration)               GraphLoop (dialogue tree)
  step():                             step():
    ├─ examine / move                   ├─ render node (authored)
    ├─ EncounterSystem.roll()           ├─ or DialogueSceneEngine.run() (LLM)
    │     └─ combat? ──┐                └─ apply flags/choices
    └─ TimeSystem.tick()
                        ▼
                  RoundLoop (combat)  — pushed on the stack
                    enter(): seed HP, initiative
                    step():  CombatSceneEngine.run() → rounds
                             CombatResolver.resolve() → deltas
                    exit():  Deltas { hp, xp, loot, faction, memories }
                        │
      stack.pop() ──────┘   exploration resumes(deltas)
                             QuestSystem.onEvent("actor:defeated")
```

Everything above is the [loop stack](../06-loop-engine.md#2-the-loop-stack): a system or the
player pushes a loop, it runs its own `SceneEngine` + schema + presenter, and returns **deltas**
— the only boundary between loops.

## 7. How the VN half and the combat half share state

This is the whole point, and it's why the split pays off:

- **One `Actor`, two views.** The same entity has `memories`/`flags`/`relationship` (VN) *and*
  `stats`/`hp`/`inventory` (RPG). A companion is both a love interest and a party member.
- **Deltas are the bridge.** The combat `RoundLoop`'s exit returns deltas the VN layer already
  understands:
  - an `actor:defeated` event can append a **memory** ("the reader killed her brother") →
    the affection math moves;
  - killing a faction's guard hits **faction reputation** (`FactionSystem`), which the VN layer
    reads when writing that faction's NPCs.
- **The resolver pattern keeps it deterministic.** Combat numbers come from a **resolver**, not
  from prose — the LLM narrates the fight, the resolver decides the outcome.
- **Skill checks are a hybrid.** A locked door: **roll** (`Rng`, DEX vs DC) → **narrate** (LLM).
  The roll is engine-side; the prose is presentation.

## 8. Worked flow — quest → travel → combat → return

1. **Quest (VN):** in town, a dialogue scene (a `GraphLoop` config). The player says *"I'll find
   your brother."* `QuestSystem` moves the quest to **active**; a memory is written for the giver.
2. **Travel (exploration):** the player types *"head into the woods."* `RpgIntent` →
   `actionKind: travel`; the exploration `GraphLoop` moves them; `SpawnSystem`/`EncounterSystem` roll.
3. **Encounter:** a wolf pack → `EncounterSystem` **pushes the combat `RoundLoop`**.
4. **Combat:** `CombatSceneEngine` streams rounds; `CombatResolver` applies HP/loot. The
   `CombatPresenter` shows HP bars and turn order. The player can `flee`.
5. **Exit:** the combat `RoundLoop` exits with `{ xp, loot, hp, events: [actor:defeated] }`; the
   exploration loop resumes with the deltas; `QuestSystem.onEvent` ticks the objective.
6. **Return (VN):** back in town, the quest completes; the giver's memory of the rescue feeds
   the relationship; `FactionSystem` may note the wolves' kin.

## 9. Build order (cheapest path)

1. **Start from the VN layer** — reuse the source almost verbatim for dialogue/story.
2. **Add `CharacterSheetPresenter` + `InventorySystem`** (stats, HP, items) — no combat yet.
3. **Add the combat `RoundLoop`** — the [worked example](../11-worked-example-combat.md) fleshed
   out: `CombatSceneEngine` + `CombatResolver` + `CombatPresenter` + `EncounterSystem`.
4. **Add the exploration `GraphLoop` + `MapPresenter`** (travel, rooms, encounters).
5. **Add `QuestSystem` + `QuestLogPresenter`** (the spine that ties loops together).
6. **Add the dialogue `GraphLoop`** with branching trees (authored nodes + LLM prose).
7. **Layer systems**: `TimeSystem`, `FactionSystem`, `CraftingSystem`, `StatusEffectSystem`.

Steps 1–3 already give a playable "VN that can break into turn-based combat and return".

## 10. Why this proves the architecture

The source game never had combat, a party, an inventory, or a map — yet **none of the kernel,
the LLM pipeline, the scene contract, the resolver pattern, the directive model, or the modifier
chain had to change**. Combat is *another loop*, quests are *another system*, and the story stays
exactly the VN the engine was built for. That's the test of a plugin/engine-centric design: a
whole new genre composes on top instead of forking the engine.

---

[← Studies](./README.md) · [Onsen →](./01-onsen-management.md)
