# 08 — Presenter engine

[← Index](./README.md) · [Modifier engine](./07-modifier-engine.md) · [Next: Prompt system →](./09-prompt-system.md)

The presenter turns validated **beats + directives** into **view state**, and owns **playback**
(line pacing, transitions) and **audio**. Keeping it separate from the loop is what lets a
combat loop reuse the same reading experience.

---

## 1. Contracts

```ts
interface PresenterEngine {
  apply(beat: Beat, view: View): View       // fold directives into view state
  tick(view: View): void                    // advance playback
  // optional
  transitions?: TransitionDriver            // covers / crossings
  audio?: AudioDirector
}

interface View {
  background: string | null
  portraits: (ActorId | null)[]             // fixed number of slots
  emotions: Record<ActorId, string>         // sprite ref (emotion + outfit suffix)
  flipped: Record<ActorId, boolean>
  departed: ActorId[]
  offStage: Record<ActorId, number>
  overrides: Record<string, unknown>
  // plugin view state: hpBars, turnOrder, ...
}
```

## 2. The directive reducer (source: `stepStage`)

`stepStage(stage, line, ctx)` folds a beat's directives into the stage, in order:

| Directive | Effect |
|---|---|
| `bg:<id>` (or `line.bg`) | set background |
| `show:<key>` | place a portrait in a free slot; auto-flip if another actor shares the pose |
| `hide:<key>` | remove the portrait; mark off-stage |
| `sprite:<key>,<emotion><_outfit>` | set the actor's sprite ref (outfit = suffix) |
| `cg:<position>` | only when exactly one actor is on screen and she has that CG |

Extras the source tracks:
- **`offStage` / `departed`:** a hidden actor is charged each reader line and leaves after
  `DEPARTURE_TURNS` (`chargeAbsences`).
- **Portrait slots:** `PORTRAIT_SLOTS`; `stagePortraitPlacement` / `stageCgPlacement` position
  them (aspect-aware).
- **`stageOverride` / `outfitLock` / `cgLock`:** scene-scoped manual display layer.
- **SFW substitution:** `sfwCgRefOf` / `sfwSpriteRefOf` swap explicit refs when rating forbids.

**Generalization:** the reducer is the presenter's `apply`. A combat presenter adds
`attack:`/`defend:` to its own `View` (HP bars, turn order) without touching the VN one.

## 3. Playback (source: `advance`)

`advance()` is a **playback state machine**, separate from loop logic. It respects:

```
awaitingInput · busy · streaming · waitingForLine · sceneEnding · endingInFlight
```

It plays `pendingLines` one at a time, skipping silent lines, and yields when the player must
act or a stream is in flight. **Keep this out of the loop engine.**

### Interjection: interrupting a scene

The player can cut into a scene **while it is still playing** — an *interject*. It is offered
whenever the scene is still "interruptible", and it is a genuine interruption: the in-flight
generation is aborted and the **unread tail is discarded**.

**The offer** (`interjectOfferOf`):

```js
interruptible =
  pendingLines.length > 0 ||                       // beats still queued to play
  sceneEnding ||                                    // the scene is closing
  (streaming && last line isn't a reader line);     // the reply is still streaming
```

…gated by `lockOf`: a **class** scene locks after the player's first action
(`sceneClass !== null && acted <= 1 ? "locked" : "open"`).

The offer surfaces as the **divider under the scene text**:

| Word | State |
|---|---|
| `HOVER TO SHOW ACTION BOX` | offer is `open` — hover to reveal the action box and cut in |
| `CLASS IN SESSION` | offer is `locked` — class scene, already acted once |
| `SCENE IS ENDING` | the scene is wrapping up |

**The interruption** (`interject()` → `takeTurnFromReply()`):

```
abortSceneCall()      // cancel the in-flight scene call (jobs.cancelGroup(SCENE_LLM_GROUP))
truncateUnread()      // DROP the text past the player's read point (pendingLines = [])
reachDecisionPoint()  // awaitingInput = true; settle departures; mark a decision point
submitAction(text)    // the player's action becomes a normal CONTINUATION turn
```

So the unread tail of the current beat is **lost**, and the writer continues from the
interruption point (a `buildContinuationPrompt` turn). There is also a **reply row** path
(`replyRowOfferOf`) that lets the player reach the decision point when the scene has stopped
but input was not handed over.

**The ending guardrail.** Interrupting *while the scene is ending* is special, because the
ending has already dispatched its state-writing work. When a scene sets `end`, `runEnding()`
fires **three calls in parallel**:

```
closing prose   runClosing()      -> the landing/goodbye        (ENDING_LLM_GROUP)
ledger          startLedger()     -> memories/events/acts/money  (ENDING_LLM_GROUP)   ┐ "bookkeeping"
texting ledger  claimTextLedger() -> the slot's texting notes    (LOOP_LLM_GROUP)     ┘
```

The last two are the **"bookkeeping"** — the resolver passes. By the time the confirm appears,
they have already been sent. Confirming the interrupt calls `dropEnding()`, which cancels
`ENDING_LLM_GROUP` and clears `pendingLedgerResult`, so **those results are thrown away**: the
memories / money / stats / events the ending would have written are not applied (they're simply
recomputed at the next ending). Hence the confirm's wording — *"Bookkeeping prompts have already
been sent. They'll be thrown away if you interrupt now."* (`InterruptEndingModal`,
`warnEndingInterrupt` default on; the "Don't warn me again" box turns it off). A sibling confirm
guards **editing** the ending scene with the same reasoning — *"Bookkeeping prompts have already
been sent. Your changes won't be reflected unless you insert another action."* (`EditEndingModal`,
`warnEndingEdit`).

**Generalization:** this is a **player-driven loop handoff** (see
[Loop engine §2](./06-loop-engine.md#2-the-loop-stack)) — the scene loop yields early, the
presenter truncates to the read point, and control returns to the player. Expose it as a
presenter action (`canInterject()` + `interject(text)`), make the "discard unread" policy
configurable (drop / keep / rewind), and treat a handoff that lands **mid-resolution** (the
bookkeeping already out) as a case that needs its own confirm + cancel semantics.

## 4. Streaming integration

- Beats are delivered as they stream (`deliverSceneLines`), with a **floor** (`owedFloor`,
  `REPLY_FLOOR_MS = 3000`) so the first beat isn't instant, and held behind a **cover** while a
  transition plays.
- If streamed count ≠ final parsed count, the preview is discarded (`dropReadPreview`) and the
  parsed beats applied wholesale.
- `queueLines` injects authored lines (e.g. an exam's opening) with no LLM.

## 5. Transitions (covers / crossings)

The source's `useCrossingStore` drives **screen transitions** between slots and scenes:
`beginCrossing`, `endCrossing`, `coverSlotCrossing`, `coverSceneOpening`, `revealSlot`, with
phases `idle → holding → closing`. A **splash** stamp carries `{ date, time, occasion, weather }`.

**Generalization:** a `TransitionDriver` any loop can call to cover a load/generation gap.

## 6. Audio

A declarative track table drives everything; selection is context-driven.

```ts
interface AudioDirector {
  play(cue: string, opts?): void
  setGroupVolume(group: string, v: number): void
  musicFor(ctx): string | null
  ambienceFor(ctx): string | null
}
```

Source:
- **Table** `MUSIC` keyed by cue, each `{ path, group, loop, trimDb, jitter }`, groups
  `music` / `ambience` / `nsfw` / `sfx`.
- **Ambience selection** `ambienceFor(base, kind, half, weather)` → outdoor/indoor × day/night ×
  clear/rain/storm; `SILENT_BACKGROUNDS` opt out.
- **Venue music** `VENUE_MUSIC` maps a place to a track (`club → venue_edm`, `mall → venue_pop`...).
- **SFX cues:** `advance`, `voice`, `narrator`, `gift`, `happy`, `money`, `rank_up`, `milestone`,
  `text_in`, plus NSFW cues (`cg_sex`, `climax`...).
- **Pitch/time:** `signalsmith-stretch` powers voice pitch (`previewVoice`, semitones).

## 7. Source map

| Concept | Source symbol |
|---|---|
| Reducer | `stepStage`, `stageAt`, `stageOnLoad`, `blankStage`, `freshStage` |
| Placement | `stagePortraitPlacement`, `stageCgPlacement`, `PORTRAIT_SLOTS` |
| Playback | `advance`, `advanceLine`, `isSilentLine`, `deliverSceneLines`, `writeSceneLines` |
| Interjection | `interjectOfferOf`, `lockOf`, `interject`, `takeTurnFromReply`, `truncateUnread`, `reachDecisionPoint`, `abortSceneCall`, `dropEnding` |
| Transitions | `useCrossingStore`, `beginCrossing`, `coverSlotCrossing`, `coverSceneOpening` |
| Audio | `MUSIC`, `ambienceFor`, `VENUE_MUSIC`, `useAudioStore` |
| SFW swaps | `sfwCgRefOf`, `sfwSpriteRefOf` |

## 8. Extension points

- **New view:** a combat presenter renders HP/turn order from its own directives.
- **New transitions:** register a `TransitionDriver` variant (fade, cut, combat wipe).
- **New cues:** add rows to the audio table; selection stays context-driven.
- **Interjection policy:** register a strategy for the unread tail on interrupt (drop / keep /
  rewind), and a `canInterject()` predicate per scene kind (the class lock is one such policy).

---

**Next:** [09 — Prompt system](./09-prompt-system.md) — blocks, roster, schemas, routing.
