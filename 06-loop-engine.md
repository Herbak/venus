# 06 — Loop engine

[← Index](./README.md) · [Resolver engine](./05-resolver-engine.md) · [Next: Modifier engine →](./07-modifier-engine.md)

The loop engine **hosts time and control flow**. It runs a stack of **loops** and a set of
**systems** that tick alongside them (texting, exams, jobs, projects, grades).

Loops come in a few reusable **skeletons** (see [§2.1](#21-loop-skeletons)). A genre is a
**config** over a skeleton plus its own **systems** and **resolver** — *not* a new loop. This is
why "the VN loop" and "an onsen loop" are the same code with different data.

The scope this engine targets — a **VN base with optional, non-real-time sub-systems** — is
spelled out in [14 — VN-first + sub-systems](./14-vn-plus-systems.md).

---

## 1. Contracts

### 1.1 Loop

```ts
interface LoopPlugin {
  id: string
  priority: number
  canEnter(world: World, e: DomainEvent): boolean
  enter(ctx: LoopContext): void
  step(): LoopStatus                 // "running" | "awaitingInput" | "done"
  exit(): Deltas                     // returned to the loop below
  suspend?(): Blob                   // serialize for the stack
  resume?(blob: Blob, deltas: Deltas): void
}
```

### 1.2 System (parallel, non-owning)

```ts
interface System {
  id: string
  onTick(world: World, clock: Clock): void
  onEvent(e: DomainEvent, world: World): void
}
```

A `System` never owns the main loop; it runs **between** scenes.

## 2. The loop stack

Loops form a **modal stack**. The top loop owns input; pushing/popping transfers control:

```
stack: [ ScheduleLoop (the day) ]
  ScheduleLoop.step() detects a trigger (combat intent, exam, cutscene)
    -> stack.push(RoundLoop config, { reason, actors, stakes })
    -> ScheduleLoop.suspend()                 // serialize
    -> the pushed loop runs (its own SceneEngine + schema + presenter)
    -> pushed.step() == "done"
    -> deltas = pushed.exit()
    -> stack.pop(); ScheduleLoop.resume(deltas)
```

This is exactly how you "insert a combat system in parallel of this VN loop": same kernel,
same scene pipeline, a different **skeleton config** + schema + presenter + modifiers. See
[Worked example: combat](./11-worked-example-combat.md).

**Design questions to answer per loop:**
- Does the handoff cost time? (combat consumes a slot? an hour? nothing?)
- What state is shared vs copied? (deltas are the boundary)
- Can the player flee back to the loop below mid-combat?

### 2.1 Loop skeletons

The loops you actually write fall into a handful of reusable **shapes** ("skeletons"). A genre
is a **config** over a skeleton, not a new loop:

| Skeleton | Shape | Examples |
|---|---|---|
| **ScheduleLoop** | phases over a calendar: open cycle → phases → resolve → advance | VN dating sim, onsen management, overworld RPG, most slice-of-life/work sims |
| **RoundLoop** | turn order, repeat until a win/lose condition | combat, exams, minigames |
| **GraphLoop** | rooms/nodes + exits until an exit condition | dungeons, investigation maps |
| **MenuLoop** | nested menus until a choice | shops, crafting, character screens |
| **CutsceneLoop** | non-interactive beats | intros, endings, story beats |

**`ScheduleLoop`** is the common one, and Venus and an onsen are the *same loop* configured
differently:

```ts
interface ScheduleLoop extends LoopPlugin {
  schedule: Phase[]                          // DATA, not code
  onCycleStart?(w: World): void
  onPhaseEnter?(p: Phase, w: World): void
  onPhaseExit?(p: Phase, w: World): void
  allowedActions?(p: Phase, w: World): ActionKind[]
  resolvePhase?(p: Phase, w: World): Deltas   // optional, per phase
  resolveCycle?(w: World): Deltas             // the day/cycle ledger
}

interface Phase {
  id: string                 // "morning" | "night" | "afternoon" ...
  name: string
  sceneKind: string          // which SceneEngine runs here
  activities: ActionKind[]   // what the player may do
  actions?: number           // budget per phase (default 1)
  pre?: string[]             // systems that tick on enter
  resolve?: boolean          // run a phase resolver here?
}
```

Same skeleton, different data:

| | phases | actions/phase | resolver cadence | scene kind |
|---|---|---|---|---|
| **Venus** | Day, Night | 1 | **per phase** (slot) | `scene` |
| **Onsen** | Dawn … Night (5) | 1–2 | **per day** | `serviceScene` |
| **Overworld RPG** | Morning … Night (4) | 1 (+ travel) | per day | `explore` / `dialogue` |

The loop is **time-agnostic**: the `schedule` can be discrete phases *or* **windows derived from a
continuous [24-hour clock](./02-kernel.md#12-continuous-a-24-hour-timer)**. Either way the loop
asks the clock to advance and reads the current window; with a continuous clock the advance is the
action's **cost**, from a structured-only
[annotation call](./04-intent-engine.md#6-the-annotation-call-structured-only).

So "the VN loop" and "the onsen loop" are **one skeleton + two configs**. A genre's real new
code is its **systems** and its **resolver** — not a new loop. Skeletons don't unify *everything*
(a fight is a `RoundLoop`, a dungeon is a `GraphLoop`), but they unify the **day**, which is the
base of the stack.

### 2.2 Player-driven handoff: interjecting

Not every handoff is triggered by a system. The player can **interject** — cut into a scene
while it is still playing — which is a handoff *from* the scene loop back to the player:

```
SceneLoop.step() is streaming beats
  -> player sends an interject (the divider offers it)
  -> abort the in-flight scene call (jobs.cancelGroup(SCENE_LLM_GROUP))
  -> truncate the unread tail (drop everything past the player's read point)
  -> reach the decision point (awaitingInput = true)
  -> the action becomes a continuation turn
```

It's the *same* suspend / truncate / resume idea as a system handoff, but initiated by input.
See [Presenter §3 — Interjection](./08-presenter-engine.md#interjection-interrupting-a-scene).

## 3. The VN loop (a `ScheduleLoop` config)

The source's VN loop is the reference **config** of `ScheduleLoop` — the same skeleton every
slice-of-life/work game uses:

```
beginSlot()                     // opening beat + world upkeep
  ├─ game-over / graduation / orientation short-circuits
  ├─ settleSpringBreak, resolveFriendRequests, settleShifts, settleGrades
  ├─ deliver Venus/Bunnybot messages, roll feed extras + suggestions
  ├─ slot-intro call: narration + hangouts + breakups + posts + plans + rumor  [SCHEMA$1]
  └─ foldOpeningIntoSlotSave(lines)

[await player input]
submitAction(action, gift, preset, planId)
  ├─ classifyAction()          // Intent engine
  ├─ castTurn()                // Cast engine (dispatch by actionKind)
  ├─ streamScene()             // Scene engine
  └─ runSceneTurn()            // continuation loop until end_scene
       └─ runEnding()          // closing/landing

crossSlotBoundary()
  ├─ applyLedger()             // Resolver engine
  ├─ resolveScene() → stat deltas
  ├─ job/project/text/NPC bookkeeping
  ├─ memory-edit gate
  ├─ writeSlotSave() → advanceSlot() → beginSlot()
```

That pipeline — open → phase → resolve → advance — **is** `ScheduleLoop`. The onsen study swaps
the phase list, the systems, and the resolver and changes nothing else (see
[Study 01](./studies/01-onsen-management.md)).

### 3.1 The playback state machine (keep separate!)

The source's `advance()` is **presenter** logic, not loop logic — it plays beats line by line
and respects `(awaitingInput, busy, streaming, waitingForLine, sceneEnding, endingInFlight)`.
Keep playback in the [Presenter](./08-presenter-engine.md) so **any** loop (combat included)
reuses transitions and line pacing.

## 4. Parallel systems (the source's side loops)

These run off the same slot and are the template for "systems in parallel":

| System | Source | What it does |
|---|---|---|
| **Texting / Bunnyboard** | `buildTextingPrompt`, `deliverSlotHangouts`, `useBunnyboardStore` | DMs, replies, hangout planning, blocking, feed posts |
| **Feed / social** | `deliverSlotPosts`, `rollFeedExtrasIfNewSlot`, likes | Status posts, likes, teasers |
| **Jobs** | `settleShifts`, `jobDefOf`, `recordShiftWorked` | Shifts, pay, raises, closures |
| **Projects** | `projectPeriodOf`, `workedOn`, `projectsNeedingWork` | Coursework progress, deadlines |
| **Grades** | `settleGrades`, `recordAcademics` | Standing, midterm/final outcomes |
| **Exams** | `startExam`, `buildQuizPrompt` | In-scene multiple-choice quiz from learned facts |
| **Occasions/weather** | `occasionsAt`, `slotWeather`, `rollNpcOverlayNow` | Holidays, weather, NPC overlays |
| **Rumors** | `slotRumor`, `rumorBlock` | Per-slot hooks that push the player somewhere |

**Generalization:** each becomes a `System` with `onTick` (per slot) and `onEvent`. A combat
"injury" system could subscribe to `sex`/`attack` events, a reputation system to `kissed`, etc.

## 5. Save points

- The VN loop saves at every **slot crossing** (`writeSlotSave`) and autosaves mid-scene
  (`writeAutosave`).
- A loop pushed on the stack should **suspend** to a blob and journal deltas so a save mid-combat
  reloads correctly.

## 6. Source map

| Concept | Source symbol |
|---|---|
| Loop host | `beginSlot`, `submitAction`, `crossSlotBoundary`, `enterGame` |
| Playback | `advance`, `deliverSceneLines`, `queueLines`, `coverSlotCrossing`, `revealSlot` |
| Side systems | `settleShifts`, `settleGrades`, `deliverSlotHangouts`, `deliverSlotPosts`, `startExam` |
| Handoff analog | `castClassTurn` → `startExam` (VN yields to an exam scene, then returns) |

> The exam is already a **mini loop handoff** in the source: `castClassTurn` detects an exam,
> calls `startExam`, which runs its own quiz flow and then returns to the slot. That is the
> pattern to generalize for combat.

## 7. Extension points

- **New loop:** prefer **configuring a skeleton** — a new `ScheduleLoop` schedule, or a
  `GraphLoop` map — before writing a `LoopPlugin` from scratch. Register it and push it when
  `canEnter` is true.
- **New system:** register a `System`; tick it per phase.
- **New trigger:** add to the intent's `actionKind` enum + a `CastStrategy`.

---

**Next:** [07 — Modifier engine](./07-modifier-engine.md) — adapting everything to specifics.
