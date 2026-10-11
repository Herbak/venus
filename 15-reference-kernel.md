# 15 — Reference kernel (code)

[← Index](./README.md) · [VN-first scope](./14-vn-plus-systems.md) · [Next: Schemas & prompts →](./16-schemas-and-prompts.md)

A **minimal but real** skeleton of the kernel and the five engines, in TypeScript. It is not
production code — it's the shape to build from. Every name maps to a chapter:

- the two primitives → [§1](#1-the-two-primitives)
- registry → [§2](#2-registry)
- world / events / clock / rng → [§3](#3-world-events-clock-rng)
- LLM gateway → [§4](#4-llm-gateway)
- scene engine → [§5](#5-scene-engine)
- loop engine (ScheduleLoop + stack) → [§6](#6-loop-engine)
- wiring → [§7](#7-wiring-a-tiny-game)

---

## 1. The two primitives

```ts
// types.ts
export type JsonSchema = Record<string, unknown>
export type ActorId = string
export type ThinkingLevel = "minimal" | "low" | "medium" | "high"

export interface DomainEvent { type: string; [k: string]: unknown }
export interface Deltas {
  stats?: Record<string, number>
  resources?: Record<string, number>
  events?: DomainEvent[]
  memories?: MemoryEntry[]
  [k: string]: unknown
}
export interface MemoryEntry { actor: ActorId; type: string; desc: string; date: number }

export interface Beat { speaker: ActorId | ""; text: string; bg?: string; directives?: Directive[] }
export type Directive = { kind: string } & Record<string, unknown>

export interface Context { world: World; events: EventBus; rng: Rng; llm: LlmGateway }
```

**`CALL`** — one structured request, one schema-shaped reply:

```ts
export interface PromptBlock { id: string; render(world: World, ctx: Context): string | null }
export interface PromptSpec {
  persona: string
  blocks: PromptBlock[]           // ordered; null blocks are omitted
  schema: JsonSchema
  cacheKey?: string               // stable prefix for prompt caching
  route?: ModelRoute
  group?: string                  // cancellation group
  images?: { mimeType: string; data: string }[]
}
export interface ModelRoute { writer: string; secondaryFor?: string[]; thinking?: ThinkingLevel }

export async function call(spec: PromptSpec, ctx: Context): Promise<unknown> {
  const user = spec.blocks.map((b) => b.render(ctx.world, ctx)).filter(Boolean).join("\n")
  return ctx.llm.complete({ system: spec.persona, user, schema: spec.schema, ...spec })
}
```

**`REDUCE`** — pure: deltas in, new state out:

```ts
export function reduce(world: World, deltas: Deltas): void {
  for (const [k, v] of Object.entries(deltas.stats ?? {})) world.addStat(k, v)
  for (const [k, v] of Object.entries(deltas.resources ?? {})) world.addResource(k, v)
  for (const m of deltas.memories ?? []) world.addMemory(m)
  for (const e of deltas.events ?? []) world.emit(e)   // systems react
}
```

## 2. Registry

```ts
// registry.ts
export class Registry {
  private slots = new Map<string, Map<string, unknown>>()
  register<T>(slot: string, id: string, impl: T): void {
    if (!this.slots.has(slot)) this.slots.set(slot, new Map())
    this.slots.get(slot)!.set(id, impl)
  }
  resolve<T>(slot: string, id = "default"): T {
    const impl = this.slots.get(slot)?.get(id)
    if (!impl) throw new Error(`no ${slot} registered as "${id}"`)
    return impl as T
  }
}
```

## 3. World, events, clock, rng

```ts
// world.ts
export interface Entity { id: string; kind: string; components: Record<string, unknown> }
export class World {
  private entities = new Map<string, Entity>()
  private stats: Record<string, number> = {}
  private resources: Record<string, number> = {}
  private memories: MemoryEntry[] = []
  constructor(private bus: EventBus, private clock: Clock) {}
  get(id: string) { return this.entities.get(id) }
  put(e: Entity) { this.entities.set(e.id, e) }
  query(kind: string) { return [...this.entities.values()].filter((e) => e.kind === kind) }
  addStat(k: string, d: number) { this.stats[k] = (this.stats[k] ?? 0) + d }
  addResource(k: string, d: number) { this.resources[k] = (this.resources[k] ?? 0) + d }
  addMemory(m: MemoryEntry) { this.memories.push(m) }
  emit(e: DomainEvent) { this.bus.emit(e) }
}

// events.ts
export class EventBus {
  private handlers = new Map<string, Set<(e: DomainEvent, w: World) => void>>()
  on(type: string, h: (e: DomainEvent, w: World) => void) {
    if (!this.handlers.has(type)) this.handlers.set(type, new Set())
    this.handlers.get(type)!.add(h)
    return () => this.handlers.get(type)!.delete(h)
  }
  emit(e: DomainEvent) { /* needs world; wire in the app */ }
}

// clock.ts
export interface Phase { id: string; name: string; sceneKind: string; activities: string[]; actions?: number; pre?: string[]; resolve?: boolean }
export class Clock {
  constructor(public day = 0, public phaseIndex = 0) {}
  get phase(): Phase { return this.schedule[this.phaseIndex] }
  constructor(private schedule: Phase[]) { /* ... */ }
  advance(): boolean {
    if (this.phaseIndex + 1 < this.schedule.length) { this.phaseIndex++; return true }
    this.phaseIndex = 0; this.day++; return false     // false = a new day began
  }
}

// rng.ts
export function mulberry32(seed: number): () => number {
  return () => {
    seed |= 0; seed = (seed + 0x6D2B79F5) | 0
    let t = Math.imul(seed ^ (seed >>> 15), 1 | seed)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
export interface Rng { stream(name: string): () => number }
```

## 4. LLM gateway

```ts
// llm.ts
export interface LlmRequest { system: string; user: string; schema: JsonSchema; images?: unknown[] }
export interface LlmGateway {
  complete(req: LlmRequest): Promise<unknown>          // returns parsed, schema-shaped JSON
  stream?(req: LlmRequest, onDelta: (d: string) => void): Promise<unknown>
}

// A minimal OpenAI-compatible adapter (add middleware for auth/retry/schema-downgrade)
export class OpenAiGateway implements LlmGateway {
  constructor(private baseUrl: string, private apiKey: string, private model: string) {}
  async complete(req: LlmRequest): Promise<unknown> {
    const res = await fetch(`${this.baseUrl}/chat/completions`, {
      method: "POST",
      headers: { "content-type": "application/json", authorization: `Bearer ${this.apiKey}` },
      body: JSON.stringify({
        model: this.model,
        messages: [{ role: "system", content: req.system }, { role: "user", content: req.user }],
        response_format: { type: "json_schema", json_schema: { name: "out", schema: req.schema, strict: false } },
      }),
    })
    const body = await res.json()
    if (!res.ok) throw new Error(body?.error?.message ?? `LLM ${res.status}`)
    return JSON.parse(body.choices[0].message.content)
  }
}
```

> Provider quirks (auth headers, `json_schema` → `json_object` downgrade, retries) belong in an
> `LlmMiddleware` chain in front of this — see [09 §6](./09-prompt-system.md#6-caching-cancellation-middleware).

## 5. Scene engine

```ts
// scene.ts
export interface SceneEngine {
  build(req: SceneRequest, ctx: Context): PromptSpec
  run(req: SceneRequest, ctx: Context): Promise<SceneReply>
}
export interface SceneRequest { sceneKind: string; actors: ActorId[]; action: string; continuation?: boolean }
export interface SceneReply { beats: Beat[]; summary?: string; end?: boolean }

export type Validator = (beat: Beat, ctx: Context) => Beat | null

export class DefaultSceneEngine implements SceneEngine {
  constructor(private spec: (req: SceneRequest, ctx: Context) => PromptSpec,
              private validators: Validator[] = [],
              private presenter: PresenterEngine) {}
  build(req: SceneRequest, ctx: Context) { return this.spec(req, ctx) }

  async run(req: SceneRequest, ctx: Context): Promise<SceneReply> {
    const raw = (await call(this.build(req, ctx), ctx)) as { beats: Beat[]; summary?: string; end?: boolean }
    const beats = (raw.beats ?? [])
      .map((b) => this.validators.reduce<Beat | null>((acc, v) => (acc ? v(acc, ctx) : null), b))
      .filter((b): b is Beat => b !== null)
    for (const b of beats) this.presenter.apply(b)   // fold directives into the view
    return { beats, summary: raw.summary, end: raw.end }
  }
}
```

A **validator** mirrors the source's sanitizer (drop unknown speaker/directive, split long lines):

```ts
export const knownActor: Validator = (b, ctx) =>
  b.speaker === "" || ctx.world.get(b.speaker) ? b : { ...b, speaker: "" }
```

## 6. Loop engine

```ts
// loop.ts
export type LoopStatus = "running" | "awaitingInput" | "done"
export interface LoopPlugin {
  id: string; priority: number
  canEnter(w: World, e: DomainEvent): boolean
  enter(ctx: Context): void
  step(): LoopStatus
  exit(): Deltas
  suspend?(): unknown
  resume?(blob: unknown, deltas: Deltas): void
}

export class LoopStack {
  private stack: LoopPlugin[] = []
  get top() { return this.stack[this.stack.length - 1] }
  push(loop: LoopPlugin, ctx: Context) { this.stack.push(loop); loop.enter(ctx) }
  pop(ctx: Context): Deltas {
    const loop = this.stack.pop()!
    const deltas = loop.exit()
    reduce(ctx.world, deltas)
    this.top?.resume?.(undefined, deltas)   // hand deltas to the loop below
    return deltas
  }
}
```

A **`ScheduleLoop`** is the VN base — phases + per-phase scene + optional resolver:

```ts
export class ScheduleLoop implements LoopPlugin {
  id = "schedule"; priority = 0
  constructor(private clock: Clock, private scene: SceneEngine,
              private sceneKind: string, private resolvePhase?: (ctx: Context) => Deltas) {}
  canEnter() { return true }
  enter() {}
  step(): LoopStatus { return "awaitingInput" }   // the VN waits on the player
  exit(): Deltas { return {} }
  // the app drives: await player action -> scene.run(...) -> clock.advance()
}
```

## 7. Wiring a tiny game

```ts
// main.ts
const bus = new EventBus()
const clock = new Clock([{ id: "day", name: "Day", sceneKind: "vn", activities: ["talk"] }])
const world = new World(bus, clock)
const llm = new OpenAiGateway("http://127.0.0.1:8787/v1", process.env.KEY!, "deepseek-v4.1-flash")
const ctx: Context = { world, events: bus, rng: { stream: () => Math.random }, llm }
const registry = new Registry()

const presenter = { apply: (_b: Beat) => {}, tick: () => {} }        // real one renders
const scene = new DefaultSceneEngine(
  (req) => ({ persona: "You are RITA...", blocks: [actorBlock(req), rulesBlock()], schema: SCENE_SCHEMA }),
  [knownActor],
  presenter,
)
const vn = new ScheduleLoop(clock, scene, "vn", () => ({ resources: {} }))
const stack = new LoopStack()
stack.push(vn, ctx)

// a turn
const reply = await scene.run({ sceneKind: "vn", actors: [], action: "I say hello." }, ctx)
reduce(world, { memories: reply.beats.filter((b) => b.speaker).map((b) => ({ actor: b.speaker as string, type: "liked", desc: "talked", date: clock.day })) })
```

---

**Next:** [16 — Schemas & prompts](./16-schemas-and-prompts.md) — the exact contracts and text.
