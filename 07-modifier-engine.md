# 07 — Modifier engine

[← Index](./README.md) · [Loop engine](./06-loop-engine.md) · [Next: Presenter engine →](./08-presenter-engine.md)

The modifier engine **adapts everything to specifics** — genre, content rating, character
traits, mood, weather, and even whole rule systems like combat. It is one **ordered chain** of
plugins with hooks across the whole pipeline.

In the source, these concerns are scattered: SFW toggles, `scenePersona`, traits, mood cycle,
weather, occasions, seed word, memory budgets, hidden cast, `inPublic`. Here they become a
single, testable pipeline.

---

## 1. Contract

```ts
interface ModifierPlugin {
  id: string
  applies(ctx: Context): boolean
  beforePrompt?(spec: PromptSpec, ctx): PromptSpec          // shape the prompt
  schema?(schema: JsonSchema, ctx): JsonSchema              // shape/limit directives
  afterReply?(reply: SceneReply, ctx): SceneReply           // fixups, substitutions
  afterResolve?(deltas: Deltas, ctx): Deltas                // transform state deltas
  onEvent?(e: DomainEvent, world: World): void              // react to events
}
```

Modifiers run **in registration order** at each hook. Each hook is a pure transform: it can
rewrite, inject, restrict, or (by returning a sentinel) veto.

## 2. Two sub-kinds

| Kind | Hooks | Space | Example |
|---|---|---|---|
| **Prompt/Schema modifier** | `beforePrompt`, `schema` | text | SFW removes sex rules + CG directives; genre injects a setting bible |
| **Rule/Outcome modifier** | `afterReply`, `afterResolve`, `onEvent` | state | combat applies HP damage; a "no violence" rule vetoes a beat |

## 3. The source's scattered modifiers → hooks

| Source feature | Hook(s) | Generalized modifier |
|---|---|---|
| `lessNsfwText` | beforePrompt | **ContentRating (text)** — drops sex direction lines |
| `noNsfwImages` | afterReply + schema | **ContentRating (visual)** — swaps CG/sprite refs, removes CG directives |
| `scenePersona` (custom system prompt) | beforePrompt | **PersonaOverride** |
| `seedWord` | beforePrompt | **AntiRepetition** — inject a random word to break patterns |
| traits (`Mood-swings`, `Disagreeable`, `Promiscuous`...) | beforePrompt | **Trait** — inject behavior lines; flags like `harem` |
| mood cycle | beforePrompt | **WorldState** — how she feels today |
| weather / occasions | beforePrompt | **WorldState** — conditions + events |
| `hiddenCast` / `inPublic` | beforePrompt + schema | **Presence/Visibility** |
| `memoryBudgets` | beforePrompt | **ContextBudget** — trim the CAST/memory blocks |
| `outfitLock` / `cgLock` / `stageOverride` | afterReply | **StageOverride** — force display state |
| — (new) | afterResolve | **Combat** — HP, damage, death |

## 4. Genre packs = modifier bundles

A genre is a bundle of modifiers plus a content pack:

```ts
const noir: ModifierBundle = {
  id: "genre:noir",
  modifiers: [HardboiledProse, CynicalTone, LowMagic, RainyPalette],
}
```

Register the bundle and it composes with the rating/trait/world modifiers automatically.

## 5. Composition order

Order matters. A sensible default:

```
1. ContentRating      (restrict first — later modifiers must respect the rating)
2. Genre              (setting bible, tone)
3. WorldState         (mood, weather, occasions)
4. Trait              (per-actor behavior)
5. Presence/Visibility
6. ContextBudget      (trim to fit the model window)
7. AntiRepetition     (seed word)
8. Rule/Outcome       (combat, economy, custom rules)
```

The chain is data — expose it in settings so a player can toggle "less explicit text" or a mod
can add a rule.

## 6. Why a pipeline, not if-statements

- **Composable:** rating × genre × traits × rules without combinatorial code.
- **Testable:** each modifier is a pure function of `(spec/deltas, ctx)`.
- **Extensible:** a mod adds a modifier; it doesn't fork the engine.
- **Reversible:** toggle a modifier off and the behavior reverts exactly.

## 7. Source map

| Concept | Source symbol |
|---|---|
| SFW text | `personaFor`, `SFW_PERSONA`, `lessNsfwText` |
| SFW visual | `noNsfwImages`, `sfwCgRefOf`, `sfwSpriteRefOf`, `stockCgsAllowed` |
| Persona override | `scenePersonaFor`, `scenePersona`, `SystemPromptModal` |
| Seed word | `seedWordBlock`, `SEED_WORDS`, `draw(SEED_WORD_BAG, ...)` |
| Traits | `CHARACTER_TRAITS`, `TRAIT_DESCRIPTIONS`, `hasTrait` |
| Mood | `moodLine`, `moodCycleOffset`, `isMoodHomebound` |
| Weather/occasions | `weatherLines`, `occasionLoreLines`, `occasionsAt` |
| Budgets | `memoryBudgetFor`, `memoryBudgets` |

---

**Next:** [08 — Presenter engine](./08-presenter-engine.md) — beats to screen.
