# Study 01 — Onsen management (a step from the campus sim)

[← Studies](./README.md) · [Next: RPG →](./02-rpg-combat-and-stories.md)

## 1. Pitch

You inherit a hot-spring inn. You run it **day by day**: greet guests, manage the baths and
rooms, keep the kitchen and the linen stocked, hire and train staff, set prices, and live the
small stories of everyone who passes through. Snow season, festival crowds, a regular who
always takes the corner room, a rival inn down the road.

This is deliberately **one step** from the source game. Venus University is already a
**time-slot loop** with a **cast that has schedules and relationships**, a **resolver that turns
prose into state** (memories, money, stats), and **side systems** (jobs, projects, texting).
An onsen manager is those same pieces with **management numbers** in place of romance stats and
a **business loop** in place of the semester.

## 2. What changes vs the source

| Source (campus) | Onsen management |
|---|---|
| 2 slots/day (Day/Night) | **day phases** — Dawn prep · Morning service · Afternoon · Evening service · Night close |
| Semester calendar, holidays | **seasons & festivals** (snow, blossom, Obon) |
| Characters (students) | **guests · staff · regulars · townsfolk** |
| Player stats brain/body/heart | **inn stats** (cleanliness, comfort, reputation, staff morale) + player **skills** (hospitality, cooking, maintenance, business) |
| Affection (weighted memories) | **guest satisfaction** and **staff loyalty** — *the same math* |
| Classes / jobs / projects | **rooms · services · upgrades · supply contracts** |
| Money (scene ledger `spent`) | the **day's books** (takings, wages, upkeep, stock) |
| Texting (Bunnyboard) | **guest book / reviews** + staff group chat |
| ComfyUI sprites & CGs | guest portraits, **room art**, seasonal backgrounds, bath scenes |

The romance layer can stay (staff/guest relationships), but it becomes one system among
several rather than the point.

## 3. Bricks mobilized (and how)

| Brick (from main docs) | Reuse | How |
|---|---|---|
| **Kernel — Clock** | ✅ mostly | Keep `date`; replace `time ∈ {0,1}` with a **phase enum**. `nextSlot` → `nextPhase`. |
| **Kernel — World** | ✅ | Same record+overlay. Entities become guests/staff/rooms/services. |
| **Kernel — EventBus** | ✅ | New events: `guest:arrived`, `room:dirty`, `shift:started`, `review:posted`. |
| **Kernel — Rng** | ✅ | Guest generation, weather, reviews — named streams. |
| **Kernel — Persistence** | ✅ | Same slot-save chain; a "slot" becomes a phase. |
| **Kernel — LlmGateway + middleware** | ✅ unchanged | Auth/retry/schema-downgrade as-is. |
| **Kernel — MediaGateway** | ✅ | ComfyUI for portraits/rooms; same workflow pattern (base → expressions). |
| **Scene engine** | ✅ | The guest/staff **scenes** (greeting, serving, bathing, dining, night). Prompt blocks (Persona/JsonRules/Setting/Cast/Now/Lore/Tail) reused verbatim. |
| **Intent engine** | ✅ | Classify free text → management intent ("clean the baths", "seat the party", "check the kitchen"). `actionKind` becomes an open registry. |
| **Resolver engine** | ✅ **core** | The **day ledger** — the same second-pass pattern: prose → money, satisfaction, cleanliness, reputation, memories. |
| **Loop engine** | ✅ reframed | `beginSlot` → `openDay`; `crossSlotBoundary` → `closeDay`. Side loops become `System`s. |
| **Modifier engine** | ✅ | Season, rating/SFW, guest/staff traits, business rules. |
| **Presenter engine** | ✅ + new view | Reuse stage/directive reducer for scenes; add a **management view** (rooms, ledger, roster). |
| **Content packs** | ✅ | Define guests/staff/rooms/services/items/festivals as data. |
| **The relationship math** | ✅ | Guest satisfaction & staff loyalty = weighted sum of typed memories — unchanged. |

**Net:** the onsen game is ~70% reuse. The real work is the **loop reframe** (day phases), the
**resolver swap** (business numbers), and a handful of **management systems**.

## 4. New plugins / engines to create

**The loop is not new.** The onsen is a **`ScheduleLoop` config** (see
[06 §2.1](../06-loop-engine.md#21-loop-skeletons)) — the same skeleton as the campus sim, with a
different phase list, a different resolver, and its own systems. Everything genuinely new is a
**system** or a **resolver** or a **presenter**:

| Slot | Plugin | Responsibility |
|---|---|---|
| `loop` | *(config, not a plugin)* | **`ScheduleLoop`**: phases `Dawn … Night`, per-day `DayLedger`, guest/staff scene kinds. |
| `scene` | **ServiceSceneEngine** | Guest/staff scene kinds (greet, serve, bathe, dine, night, dispute). Reuses `Beat`/`Directive`. |
| `intent` | **ManagementIntent** | Free text → `{ actionKind: serve/clean/assign/upgrade/price/tend, targets[] }`. |
| `resolver` | **DayLedger** | End-of-day second pass: takings, wages, upkeep, satisfaction, cleanliness, reputation, staff morale, memories. |
| `presenter` | **ManagementPresenter** | The inn's UI: room board, bookings, ledger, roster, stock — plus the scene stage. |
| `system` | **BookingSystem** | Reservations, occupancy, no-shows, overbooking. |
| `system` | **HousekeepingSystem** | Room/bath cleanliness decay; dirty rooms hurt satisfaction. |
| `system` | **StaffSystem** | Shifts, morale, skills, training, wages, quits/hires. |
| `system` | **EconomySystem** | Prices, takings, upkeep, stock, loans, upgrades. |
| `system` | **ReputationSystem** | Reviews → word-of-mouth → arrival rate. |
| `system` | **GuestFlowSystem** | Who arrives today (season × reputation × festivals). |
| `system` | **SeasonSystem** | Seasons, festivals, weather; drives guests, ambience, decor. |
| `modifier` | **SeasonModifier** | Seasonal prompt blocks + asset swaps. |
| `modifier` | **RatingModifier** | Reuse the SFW/content-rating chain. |
| `modifier` | **GuestTraitModifier** | Guest/staff traits → behavior lines (the source's `Trait` modifier, re-skinned). |
| `content` | **Onsen pack** | guests, staff, rooms, services, items, festivals, upgrades. |

## 5. New directives, schemas, entities

**Directives** (added to the extensible vocabulary):
`serve:<guest>`, `clean:<room>`, `assign:<staff>,<task>`, `upgrade:<room>`, `price:<service>,<n>`,
`seat:<party>,<room>`, `tend:<guest>`.

**Schemas** (structured-output contracts):
`dayReport`, `guestStay`, `booking`, `staffShift`, `review`, `serviceScene` (a `scene` variant).

**Entities**: `Guest`, `Staff`, `Room`, `Service` (bath/dining/shop), `Item` (stock/gift),
`Festival`, `Upgrade`, `Review`.

## 6. Loop diagram

```
ScheduleLoop  (onsen config: phases Dawn … Night, per-day resolve)
  openDay()
    ├─ SeasonSystem.roll()        weather, festivals
    ├─ GuestFlowSystem.roll()     today's arrivals (season × reputation)
    ├─ StaffSystem.roll()         shifts, morale, quits
    ├─ BookingSystem.sync()       rooms vs arrivals
    └─ opening scene: the inn waking up            [Scene: slot-intro variant]

  [phase loop: Dawn → Morning → Afternoon → Evening → Night]
    ├─ player action → ManagementIntent.classify()
    ├─ ServiceSceneEngine.run()   a guest/staff scene (VN)     [Scene]
    └─ systems tick per phase

  closeDay()
    ├─ DayLedger.resolve()        takings, wages, upkeep, satisfaction, reputation  [Resolver]
    ├─ HousekeepingSystem.decay() rooms/baths
    ├─ ReputationSystem.reviews() guest book → arrival rate
    ├─ applyLedger() → deltas
    └─ writePhaseSave() → nextPhase → openDay()
```

## 7. Worked flow — a guest's stay

1. **Arrival** (`GuestFlowSystem`): "A couple from the city, first-timers" enters the world.
2. **Greeting** (`ServiceSceneEngine`): a VN scene. The player types *"Welcome them and take their
   bags"*. The **Intent engine** classifies it (`serve`, target = the couple).
3. **Service**: the player runs the bath and dinner scenes. The **stage** shows the room; the
   **directive reducer** reuses `show:`/`sprite:` for the guests.
4. **Satisfaction**: the scene's beats feed typed **memories** ("the reader remembered her
   dietary restrictions") — the same weighted-sum math scores **guest satisfaction**.
5. **Review** (`ReputationSystem`): at checkout the guest leaves a review; reputation moves.
6. **Close of day** (`DayLedger`): takings − wages − upkeep = the day's books; cleanliness decayed
   in the busy rooms; staff morale ticked. `applyLedger` writes it all.
7. **Tomorrow**: higher reputation → more arrivals; a dirty room from yesterday → a bad review.

## 8. Build order (cheapest path)

1. **Reuse the source loop as-is**; rename slots to day phases.
2. **Swap the stat block** (brain/body/heart → inn stats) and the ledger fields (spent → books).
3. **Add EconomySystem + DayLedger**; make money the first management number.
4. **Add BookingSystem + HousekeepingSystem** (rooms exist and get dirty).
5. **Add StaffSystem** (shifts, morale) — reuses the source's job-shift code directly.
6. **Add SeasonSystem + ReputationSystem** (the long arc).
7. **ManagementPresenter** last — the UI is the most bespoke part.

Because the kernel, scene pipeline, resolver pattern, relationship math, and content packs all
carry over, steps 1–3 already produce a playable "inn that runs itself with LLM scenes".

---

[← Studies](./README.md) · [Next: RPG →](./02-rpg-combat-and-stories.md)
