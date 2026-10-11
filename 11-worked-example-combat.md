# 11 — Worked example: combat in parallel

[← Index](./README.md) · [Content & mods](./10-content-and-mods.md) · [Next: Roadmap →](./12-roadmap-and-rules.md)

Goal: add a **combat system that runs alongside the VN loop**, triggered from free text, with
**zero kernel changes**. Everything here uses the contracts from
[Loop](./06-loop-engine.md), [Scene](./03-scene-engine.md), [Intent](./04-intent-engine.md),
[Resolver](./05-resolver-engine.md), [Modifier](./07-modifier-engine.md), and
[Presenter](./08-presenter-engine.md).

---

## 0. Why this is additive

The source already proves the pattern: `castClassTurn` detects an **exam**, calls `startExam`
(which runs its own quiz flow), then returns to the slot. Combat is the same handoff, with its
own schema, presenter, and resolver.

## 1. The content pack

```
combat/
  pack.json
  entities/enemies.json
  schemas/combat.json
  prompts/combat.ts
  directives/combat.ts
  modifiers/violence.ts
  loops/combat.ts
  systems/injuries.ts
  assets/
```

## 2. The combat scene (Scene engine plugin)

```ts
// schemas/combat.json  (abridged)
{
  "rounds": [{
    "actor": "string",
    "action": "string",           // "attack" | "defend" | "item" | "flee"
    "target": "string",
    "hpDelta": "integer",
    "narration": "string"
  }],
  "outcome": "string"             // "win" | "lose" | "fled"
}
```

```ts
// prompts/combat.ts
const CombatScene: SceneEngine = {
  build(req, ctx) {
    return {
      persona: "You narrate fast, punchy combat in second person, present tense.",
      blocks: [CombatRules, ActorsBlock, TerrainBlock, StatusBlock, CombatTail],
      schema: loadSchema("combat"),
      route: { writer: "big", thinking: "low" },
      group: "COMBAT_LLM_GROUP",
    }
  },
  async run(req, ctx) { /* stream + validate + apply, as the Scene engine does */ },
}
```

```ts
// directives/combat.ts — the extensible vocabulary
registerDirective("attack", { target: "actor", damage: "int" })
registerDirective("defend", {})
registerDirective("item",   { item: "id" })
registerDirective("flee",   {})
```

## 3. The combat loop (a `RoundLoop` config)

Combat is the **`RoundLoop` skeleton** (see [06 §2.1](./06-loop-engine.md#21-loop-skeletons)),
specialized: turn order, repeat until a win/lose condition. The genre-specific parts are the
**schema**, the **resolver**, and the **presenter** — the loop itself is a config.

```ts
// loops/combat.ts  — the RoundLoop config for combat
const CombatLoop: LoopPlugin = {
  id: "combat",
  priority: 10,
  canEnter(world, e) { return e.type === "combat:start" },
  enter(ctx) { /* seed HP from actors, set turn order */ },
  step() { return "awaitingInput" },   // or "running"/"done"
  exit(): Deltas {
    return {
      resources: { hp: /* final hp */ },
      events: [{ type: "combat:resolved", outcome }],
      memories: [],   // e.g. a bond from fighting side by side
    }
  },
  suspend() { /* serialize combat state */ },
  resume(blob, deltas) { /* restore */ },
}
```

Register it: `registry.register("loop", "combat", CombatLoop)`.

## 4. The intent (Intent engine plugin)

Add `combat` to the open `actionKind` enum and a **cast strategy**:

```ts
registry.register("caster", "combat", {
  matches: (k) => k === "combat",
  cast: (intent, ctx) => ({
    loop: "combat",
    actors: [ctx.player, ...intent.actors],
    target: intent.extra?.target,
  }),
})
```

When `classifyAction` returns `actionKind: "combat"`, the base `ScheduleLoop` pushes the combat loop.

## 5. The handoff (Loop stack)

```ts
// inside the VN loop, after casting
if (plan.loop === "combat") {
  stack.push(registry.resolve("loop", "combat"), {
    reason: "player attacked", actors: plan.actors, stakes: plan.target,
  })
  return   // base loop suspends; the combat loop takes over
}
// ... later, the combat loop exits and the base loop resumes:
// stack.pop(); deltas = exit(); world.apply(deltas); resumeBase()
```

## 6. The resolver (Resolver engine plugin)

```ts
const CombatResolver: ResolverEngine = {
  async resolve(scene, ctx) {
    // parse `rounds` → hp deltas, injuries, loot
    return { resources: { hp: netHp }, events: [...], memories: [...] }
  },
}
```

Registered per-scene-kind, so `runSceneTurn` picks the combat resolver for combat scenes and
the VN ledger for dialogue scenes.

## 7. The modifier (Modifier engine plugin)

```ts
const Violence: ModifierPlugin = {
  id: "rule:violence",
  applies: () => true,
  beforePrompt(spec) {
    return { ...spec, blocks: [ViolenceRules, ...spec.blocks] }
  },
  schema(schema) {
    // add attack/defend/item/flee to the directive enum
    return withCombatDirectives(schema)
  },
  afterResolve(deltas) {
    // clamp hp to [0, maxHp]; mark death
    return clampHp(deltas)
  },
}
```

The existing **ContentRating** modifier still runs first and can veto graphic beats.

## 8. The presenter (Presenter engine plugin)

```ts
const CombatPresenter: PresenterEngine = {
  apply(beat, view) { /* fold attack/defend directives → hpBars, turnOrder */ return view },
  tick(view) { /* animate HP changes */ },
}
```

Transitions reuse the source's `cover` system; SFX use the audio table (`hit`, `clash`...).

## 9. What did *not* change

- Kernel: clock, world, events, RNG, persistence, gateways.
- The LLM gateway, middleware, caching, cancellation.
- The validator pipeline, the prompt block library, the schema builder.
- The VN loop's own logic (it only gained one `if`).

That is the payoff of the engine split: **a genre system is a bundle of plugins**, not a fork.

---

**Next:** [12 — Roadmap & rules](./12-roadmap-and-rules.md).
