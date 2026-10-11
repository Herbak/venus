# Study 03 — Detective / mystery

[← Studies](./README.md) · [Plugins catalogue →](./plugins-catalogue.md)

## 1. Pitch

A mystery: a crime, a cast of suspects, and a case to close before the trail goes cold. You
**investigate** (search scenes, examine evidence), **interrogate** (press a suspect, catch a lie),
and **deduce** (assemble evidence into an accusation). The story is told in the same
second-person prose as the source; the detective work is new systems on the same kernel.

This is a good contrast study because it has **no combat and no economy** — the "verb" is
**information**. The engine's job is to track *who knows what, and whether the player can prove it*.

## 2. What changes vs the source

| Source (campus VN) | Detective / mystery |
|---|---|
| Day/Night slots, semester | a **case clock** (hours/days until the trail cools) |
| Cast = romance targets | **suspects · witnesses · victim · police** |
| Affection | **suspect trust** (same weighted-memory math) + **suspicion** |
| Class/job/project ledger | the **case file**: evidence, leads, alibis, timeline |
| Texting | **informants** / anonymous tips |
| CG/sprite directives | **evidence close-ups**, crime-scene backgrounds |
| "stats" brain/body/heart | **skills**: observation, empathy, intimidation, deduction |

## 3. Bricks mobilized (and how)

| Brick | Reuse | How |
|---|---|---|
| **Kernel** (clock/world/events/RNG/persistence/gateways) | ✅ | Clock becomes the **case clock**; events drive `evidence:found`, `alibi:broken`. |
| **Scene engine** | ✅ **core** | Interrogation and search scenes are ordinary VN scenes; the prompt blocks carry the case file as LORE. |
| **Intent engine** | ✅ | `actionKind` registry: `search`, `examine`, `question`, `press`, `tail`, `accuse`. |
| **Resolver engine** | ✅ **core** | **ClueResolver** (what the scene yielded) + **TheoryResolver** (does the accusation hold). |
| **Loop engine** | ✅ | A **`ScheduleLoop` config** (the case day) + a pushed **`RoundLoop` config** (interrogation). Same skeleton as the source, new phases. |
| **Modifier engine** | ✅ | `GenreModifier` (noir tone), `DifficultyModifier` (how much the LLM withholds), rating. |
| **Presenter engine** | ✅ + new view | Reuse the stage; add a **case board** (evidence, suspects, timeline). |
| **Content packs** | ✅ | the case: suspects, locations, evidence, alibis, clues as data. |
| **Relationship math** | ✅ | suspect trust = weighted memories; suspicion = its inverse. |
| **The ledger pattern** | ✅ **the key idea** | The **evidence ledger**: a scene yields *facts the player can now prove*, not prose. |

## 4. New plugins / engines to create

| Slot | Plugin | Responsibility |
|---|---|---|
| `loop` | **`ScheduleLoop` config** (case day) | Travel between locations, run scenes, tick the clock. *(Not a new loop.)* |
| `loop` | **`RoundLoop` config** (interrogation) | A pushed sub-loop for pressing a suspect. |
| `loop` | **`MenuLoop` config** (deduction) | The accusation: assemble evidence → verdict. |
| `scene` | **InterrogationSceneEngine** | Question/press beats; schema `interrogation`. |
| `scene` | **SearchSceneEngine** | Examine a location/object; schema `search` (findings, leads). |
| `intent` | **InvestigateIntent** | Free text → `{ actionKind: search/examine/question/press/tail/accuse }`. |
| `resolver` | **ClueResolver** | Scene → evidence items, leads, alibi states, suspicion deltas. |
| `resolver` | **TheoryResolver** | Accusation → verdict (does the evidence actually prove it?). |
| `presenter` | **CaseBoardPresenter** | Evidence board, suspects, timeline, contradictions. |
| `system` | **EvidenceSystem** | The case file: evidence items, who knows them, links. |
| `system` | **AlibiSystem** | Alibis as claims; contradictions break them. |
| `system` | **SuspectSystem** | Suspects' whereabouts, relationships, secrets. |
| `system` | **TimelineSystem** | What happened when; where claims conflict. |
| `system` | **PressureSystem** | The case clock; the trail cools, witnesses vanish. |
| `modifier` | **GenreModifier** | Noir/hardboiled tone blocks (a reusable bundle). |
| `modifier` | **WithholdModifier** | How much the LLM volunteers vs hides (difficulty). |
| `content` | **Case pack** | suspects, locations, evidence, alibis, clues, the solution. |

## 5. New directives, schemas, entities

**Directives**: `evidence:<id>`, `alibi:<who>,<claim>`, `reveal:<secret>`, `clue:<id>`,
`accuse:<who>`.

**Schemas**: `interrogation` (questions, tells, contradictions), `search` (findings, leads),
`evidence` (item, links, certainty), `deduction` (theory, supported-by), `accusation` (who, why,
proof[]).

**Entities**: `Suspect`, `Witness`, `Evidence`, `Lead`, `Alibi`, `Location`, `Secret`, `Case`.

## 6. Loop diagram

```
ScheduleLoop (case config; clock ticks)
  step():
    ├─ travel to a location            [Intent → SearchSceneEngine]
    ├─ search a room/object            [Scene] → ClueResolver → EvidenceSystem
    ├─ question a suspect
    │     └─ push RoundLoop (interrogation)  [Scene] → trust/suspicion, maybe an alibi breaks
    └─ PressureSystem.tick()           the trail cools; a witness leaves

Deduction (MenuLoop) when the player accuses
    ├─ TheoryResolver: is the accusation supported by evidence?
    └─ verdict scene                  [Scene] → ending (solved / wrong / cold)
```

## 7. Worked flow — catching a lie

1. **Search:** the player types *"search the study."* `InvestigateIntent` → `search`; the
   `SearchSceneEngine` narrates; `ClueResolver` yields **evidence: muddy footprints** + a lead.
2. **Interrogate:** the player questions the groundskeeper, who claims he was in the garden.
   Pushed the interrogation `RoundLoop`; `AlibiSystem` records the claim.
3. **Contradiction:** the footprints are *dry mud indoors* — the player examines them
   (`examine`). `EvidenceSystem` links evidence → `AlibiSystem` breaks the alibi; the suspect's
   **trust/suspicion** moves (a memory is written).
4. **Accuse:** the player assembles the footprints + the broken alibi and accuses.
   `TheoryResolver` checks the **proof chain** — not the prose. If supported → *solved*; if not →
   the suspect walks and the trail cools.

The crucial part: the verdict is **deterministic** (evidence graph), while the **prose is LLM**.
That's the resolver pattern applied to information instead of numbers.

## 8. Build order

1. **Reuse the VN scene engine** for search/interrogation scenes.
2. **Add `EvidenceSystem` + `ClueResolver`** (the case file is the first new state).
3. **Add `AlibiSystem` + `TimelineSystem`** (contradictions).
4. **Add the interrogation `RoundLoop`** (a pushed sub-loop, like combat).
5. **Add `CaseBoardPresenter`** (the UI).
6. **Add `TheoryResolver` + the deduction `MenuLoop`** (the accusation).

---

[← Studies](./README.md) · [Plugins catalogue →](./plugins-catalogue.md)
