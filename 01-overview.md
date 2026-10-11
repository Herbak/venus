# 01 — Overview & concepts

[← Index](./README.md) · [Next: Kernel →](./02-kernel.md)

---

## 1. The mental model

The source game (Venus University) looks monolithic but is **five narrow concerns** entangled
by naming. Pull them apart and each becomes an **engine**:

1. **Kernel** — clock, world state, events, RNG, persistence, gateways.
2. **Scene engine** ("play a scene") — prompt assembly → structured LLM call → validate → apply.
3. **Intent engine** — free text → structured intent (the classifier).
4. **Resolver engine** — narrative result → state deltas (the "ledger").
5. **Loop engine** — the scheduling/state-machine host that runs scenes and side-systems.

Plus a **modifier engine** — an ordered pipeline that transforms *all* of the above to adapt
to a specific game (genre, rating, traits, mood, combat rules...).

## 2. The two primitives

Everything reduces to:

```
CALL(persona, blocks[], schema) -> structured JSON    // every LLM task
REDUCE(events[]) -> deltas -> newState                // every state change
```

- **`CALL`** is a structured LLM request: a persona, an ordered list of prompt blocks, and a
  JSON Schema the reply must match. One call, one well-defined output shape.
- **`REDUCE`** is pure: narrative results and domain events in, deltas out, new state. The LLM
  never mutates state directly.

If you build only these two well, everything else is composition.

## 3. Architecture

```
                       ┌───────────────────────────────────────────────┐
                       │                   KERNEL                       │
                       │  Clock · World · EventBus · Rng · Persistence  │
                       │  LlmGateway(+middleware) · MediaGateway        │
                       │  ContentRegistry · EngineRegistry              │
                       └───────────────┬───────────────────────────────┘
                                       │  resolve(slot)
        ┌──────────────┬───────────────┼───────────────┬──────────────┐
        ▼              ▼               ▼               ▼              ▼
   ┌─────────┐   ┌──────────┐   ┌───────────┐   ┌───────────┐  ┌───────────┐
   │  LOOP   │──▶│  INTENT  │──▶│   SCENE   │──▶│ RESOLVER  │─▶│ PRESENTER │
   │ engine  │   │  engine  │   │  engine   │   │  engine   │  │  engine   │
   └────┬────┘   └──────────┘   └─────┬─────┘   └─────┬─────┘  └─────┬─────┘
        │                             │               │              │
        │        ┌────────────────────┴───────────────┘              │
        │        ▼                                                     │
        │   ┌─────────────────────────────┐                          │
        └──▶│      MODIFIER  pipeline     │  beforePrompt · schema ·  │
            │  (ordered plugins, 5 hooks) │  afterReply · afterResolve│
            └─────────────────────────────┘  · onEvent                │
                                                                       ▼
                                                              beats → view → screen
```

- The **Loop engine** drives time and hands control to the others.
- The **Modifier pipeline** sits across Scene/Resolver and adapts them.
- The **Presenter** turns `Beat`s (with directives) into the screen.
- The **Kernel** provides services and resolves every engine by name.

## 4. The five engines

| Engine | Contract (short) | Source analog |
|--------|------------------|---------------|
| **Scene** | `build(req) → PromptSpec`; `run(req) → SceneReply` | `buildScenePrompt`, `streamScene`, `stepStage` |
| **Intent** | `classify(text, ctx) → Intent` | `classifyAction`, `buildClassifierPrompt` |
| **Resolver** | `resolve(scene, ctx) → Deltas` | `buildLedgerPrompt`, `applyLedger`, `resolveScene` |
| **Loop** | `canEnter/step/exit`, plus `System.onTick` | `beginSlot` … `crossSlotBoundary` |
| **Modifier** | `beforePrompt/schema/afterReply/afterResolve/onEvent` | SFW, traits, mood, weather, persona override |

Full contracts live in their own chapters.

## 5. The universal scene contract (preview)

The bridge between the LLM and the screen is a **typed directive stream**:

```ts
interface Beat {
  speaker: ActorId | ""       // "" = narration
  text: string
  directives?: Directive[]    // show/hide/sprite/cg/bg + plugin directives
}
type Directive = { kind: string; [k: string]: unknown }   // EXTENSIBLE
```

Directives are a **plugin-extensible vocabulary** — backgrounds, outfits, CGs for a VN;
`attack:`/`defend:` for combat. See [Scene engine](./03-scene-engine.md) and
[Presenter engine](./08-presenter-engine.md).

## 6. Why plugin/engine-centric

Because every concern is behind an interface, adding a genre system is additive:

- A **combat system** = a new `LoopPlugin` + `SceneEngine` + `Modifier` bundle — no kernel changes.
- A **genre** = a bundle of `Modifier`s + a content pack (personas, blocks, schemas, assets).
- A **new provider** = an `LlmGateway`/`MediaGateway` adapter (and any header/format quirks are
  `LlmMiddleware` plugins).

See [Worked example: combat](./11-worked-example-combat.md).

## 7. Glossary

| Term | Meaning |
|------|---------|
| **Kernel** | Engine-agnostic services (clock, world, events, RNG, persistence, gateways). |
| **Engine** | A component with a defined contract, registered under a slot. |
| **Plugin** | A concrete implementation of an engine (or a modifier/loop/system). |
| **Slot** | A named registry key (`scene`, `loop`, `modifier`, ...). |
| **Beat** | One unit of scene output (speaker + text + optional directives). |
| **Directive** | A typed stage instruction attached to a beat (`show:`, `attack:`...). |
| **Block** | A reusable fragment of a prompt (`Persona`, `Cast`, `Now`...). |
| **CALL / REDUCE** | The two primitives. |
| **Ledger** | The resolver pass that turns finished prose into state. |
| **Loop stack** | The modal stack of active loops (VN ↔ combat ↔ ...). |
| **Content pack** | A folder of entities + prompts + schemas + assets defining a world/genre. |
