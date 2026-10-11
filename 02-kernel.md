# 02 — Kernel

[← Index](./README.md) · [Overview](./01-overview.md) · [Next: Scene engine →](./03-scene-engine.md)

The kernel holds everything **engine-agnostic**: services that every game needs regardless of
genre. Each is an interface with swappable implementations, resolved from the
[plugin registry](#7-plugin-registry).

---

## 1. Clock

Defines what "a turn" is. The clock is a **plugin** — a game can use discrete slots, rounds, or
a **continuous 24-hour timer**. The loop never cares which: it only asks the clock to *advance*
and to report the current **window**.

### 1.1 Discrete (the source)

The source uses **2 slots per day**.

```ts
interface Clock {
  now(): Slot                       // opaque, comparable, serializable
  tick(): Slot                      // advance
  schedule(at: Slot, fn): void      // run fn when the clock reaches `at`
  slotsUntil(a: Slot, b: Slot): number
}
```

Source analog: `date` (integer) + `time ∈ {0,1}`; `nextSlot`, `prevSlot`, `weekOf`, semester
weeks, `HOLIDAYS`. Everything is keyed by `(date, time)` — schedules, plans, job shifts, class
meetings, replays, saves.

### 1.2 Continuous (a 24-hour timer)

The same slot serves a **continuous** clock — a timestamp (minutes since day 0, or a full
date-time) with **windows derived from it**. The old "phases" become **windows over time**:

```ts
interface TimeModel {
  now(): Timestamp
  advance(cost: Duration): Timestamp   // spend time: an action's cost, or a real-time tick
  window(): WindowDef                  // derived: "morning" | "afternoon" | ...
  windows: WindowDef[]
}
interface WindowDef { id: string; from: string; to: string; sceneKind: string; activities: string[] }
```

- **morning = 06:00–12:00**, **night = 22:00–02:00** — windows are **data**, derived from the
  clock, not fixed indices.
- Time advances either by **action cost** (each action spends minutes — *recommended for a VN*) or
  by a **real-time tick** (wall-clock; needs offline progression + clock-spoof handling).
- **Timers/alarms** fire at a timestamp ("the 10:00 bell", "the deadline in 3 days").
- **Systems** tick on `dt` (elapsed time) rather than on a phase change.

The **action cost** comes from a structured-only **annotation call** — see
[04 §6](./04-intent-engine.md#6-the-annotation-call-structured-only). A `ScheduleLoop` works with
slots *or* a continuous clock unchanged ([06 §2.1](./06-loop-engine.md#21-loop-skeletons)).

**Generalization:** granularity is a plugin choice — a slot, a round, a minute, a day. Keep
`Slot`/`Timestamp` opaque so a combat loop can use "rounds" while the VN loop uses "windows".

## 2. World state

The single source of truth. The source uses zustand stores plus a module-level scratch object
(`loopState`).

```ts
interface World {
  get<T>(id: EntityId): T | undefined
  query(q: Query): Entity[]
  apply(deltas: Deltas): void
  view(scope): WorldView            // read-only projection for prompt building
}
```

**Record + overlay pattern** (from the source's persistence):
- **Record** — the immutable world seed (characters, classes, occasions, weather table...).
- **Overlay** — the mutable delta (charInfo, flags, memories, money, stats...).

Source stores: `useGameStore` (world), `useUiStore`, `useSettingsStore`, `useSaveStore`,
`useAudioStore`, `useAssetStore`, `useCharacterStore`, `useBunnyboardStore`, `useJobStore`,
`useComfyStore`, `usePhotoStore`, `useSceneCreatorStore`, `useGrabBagStore`, `useSetupStore`,
`useCrossingStore`, `useUpdateStore`.

**Generalization:** treat the world as entities + components. The record/overlay split is what
makes saves small and migrations tractable.

## 3. Event bus

Domain events decouple systems from the loop.

```ts
interface EventBus {
  emit(e: DomainEvent): void
  on(type: string, h: (e, world) => void): Unsub
}
type DomainEvent = { type: string; [k: string]: unknown }
```

Source analog: implicit — `applyEvent(flags, event)` transitions flags from ledger events
(`kissed`, `sex`, `became_lovers`, `broke_up`, `gave_contact_info`, `unblocked`,
`agreed_to_harem`). Make these first-class events so any system can subscribe (a jealousy
system, a reputation system, a combat system).

## 4. RNG

```ts
interface Rng {
  stream(name: string): () => number   // 0..1, reproducible per named stream
}
```

Source analog: `Math.random` in places, plus explicit `seed` per character
(`generationSeed`) and `SeedNode` in ComfyUI workflows. **Generalization:** named streams let
you reproduce world generation, image generation, and combat rolls independently.

## 5. Persistence

```ts
interface SaveProvider {
  snapshot(): Blob                    // full world
  journal(d: Deltas): void            // append deltas (crash-safe)
  load(id): World
  list(playthroughId): SaveMeta[]
}
```

Source chain:

```
playthrough (folder) ─► record: chars, classes, profiles, jobs, occasions, weather,
                                 jobClosures, haunts, feeds, springBreak
                     ─► enrollment (course registration draft)
                     ─► ≤9 slot saves  (state + scene + ledger + stage)
                     ─► replays[date][time] (word-for-word scene history)
                     ─► photos
```

Source details: `writeSlotSave`, `writeAutosave`, `MAX_SLOT_SAVES = 9`, `useSaveStore`,
`readPlaythroughRecord`, `stampRecord`. Settings + encrypted keys live in `data/settings.json`.

**Generalization:** namespace deltas by plugin id so a combat pack's HP journal coexists with
the VN journal; version schemas (`schemaVersion`) and migrate.

## 6. Gateways

### 6.1 LLM gateway + middleware

```ts
interface LlmGateway {
  complete(spec: PromptSpec): Promise<RawReply>       // non-streaming
  stream(spec: PromptSpec): AsyncIterable<Delta>       // streaming
  listModels?(endpoint): Promise<ModelInfo[]>
}
interface LlmMiddleware {
  onRequest(req: Request): Request      // auth, headers, format translation
  onResponse(res: Response): Response   // retries, downgrade, cost accounting
}
```

Source analog: `openaiAdapter` / `geminiAdapter` with `buildCall`, `contentOf`,
`envelopeErrorFor`, `sendCall`, `readStream`; provider table `PROVIDERS`.

> The **header-injecting proxy** built for OpenCode Go is already an `LlmMiddleware` plugin:
> it adds `User-Agent` + `x-opencode-session` and translates `json_schema` → `json_object`.
> Formalize middleware as the place for auth, retries, schema downgrade, logging, and cost.

### 6.2 Media gateway

```ts
interface MediaGateway {
  image(spec: ImageSpec): Promise<ImageRef>
  audio(spec: AudioSpec): Promise<AudioRef>
}
```

Source analog: ComfyUI queue (`/prompt`, websocket), the `character*` workflows, and
`buildImageCall` for the images API. See [Content & mods](./10-content-and-mods.md).

## 7. Plugin registry

Everything is registered under a **slot** and resolved by name:

```ts
interface EngineRegistry {
  register<T>(slot: string, id: string, impl: T): void
  resolve<T>(slot: string, id?: string): T   // id omitted = default
}
```

Slots:

```
loop       : LoopPlugin        // VN loop, combat loop, management loop...
scene      : SceneEngine       // "play a scene"
intent     : IntentEngine      // classifier
resolver   : ResolverEngine    // ledger / outcome
caster     : CastEngine        // who is present
presenter  : PresenterEngine   // beats -> view
modifier   : ModifierPlugin[]  // ordered chain
content    : ContentProvider   // entities/assets
llm        : LlmGateway
media      : MediaGateway
save       : SaveProvider
```

## 8. Content registry

```ts
interface ContentProvider {
  entities(kind: string): Entity[]
  prompts(id: string): PromptSpec
  schemas(id: string): JsonSchema
  asset(id: string): AssetRef
}
```

Source analog: `assets/` (characters, pose, sound, workflows), `quickstart.json`, the `data/`
folder, and the ComfyUI workflow JSONs. See [Content & mods](./10-content-and-mods.md).

---

**Next:** [03 — Scene engine](./03-scene-engine.md) — the main engine that plays a scene.
