# 09 — Prompt system

[← Index](./README.md) · [Presenter engine](./08-presenter-engine.md) · [Next: Content & mods →](./10-content-and-mods.md)

Every LLM task in the source is the same shape: **persona + ordered blocks + JSON schema**, with
a `cacheKey` and a model route. This chapter generalizes that into a reusable prompt system.

---

## 1. The shape

```ts
interface PromptSpec {
  persona: string                 // the "voice" (source: RITA)
  blocks: PromptBlock[]           // ordered user-message fragments
  schema: JsonSchema              // the structured-output contract
  cacheKey?: string               // stable prefix for prompt caching
  route?: ModelRoute              // writer vs cheap model, thinking level
  group?: string                  // cancellation group
  images?: ImageRef[]             // multimodal inputs
  minThinking?: ThinkingLevel     // floor for this task
}

interface PromptBlock {
  id: string
  render(world: WorldView, ctx: Context): string | null   // null = omit
}
```

A **block** is a reusable fragment that renders itself (or omits itself) from world state.

## 2. The block library

All of these exist in the source:

| Block | Renders |
|---|---|
| `Persona` | the writer's voice (RITA + anti-slop rules) |
| `JsonRules` | the output contract (beats, directives, summary, end) |
| `Setting` | the world's premise |
| `Protagonist` / `Reader` | the player + his schedule |
| `Cast` | per-actor: appearance, outfits, personality, flags, affection, mood, relationships, schedule, plans, memories, notes |
| `Now` | date/time, semester, weather, current bg/CG, on-screen emotions, hidden cast |
| `Context` | class/job/quest context |
| `Lore` | lorebook entries gated by a scan of the action/text |
| `SceneSoFar` | recap + word-for-word transcript stubs |
| `Tail` | the "YOUR TURN" mission + the reader's action |
| `SeedWord` | a random word to fight repetition |

**Block order for a scene** (the reference):

```
SYSTEM:  Persona → "" → JsonRules → "" → Setting
USER:    Reader → Cast → Now → Context → Lore → SceneSoFar → Tail(+SeedWord+action)
```

## 3. The output contract (`JsonRules`)

```
beats[]: { speaker: "" | <actorKey>, text, bg?: <bg enum>, directives?: <enum>[] }
directives: show:<key> | hide:<key> | sprite:<key>,<emotion><_outfit> | cg:<position>
+ summary?: string   + end?: boolean
```

The `bg` and `sprite` enums are **built per scene** from the registry (available backgrounds +
actor room bgs; emotions + outfit suffixes the actor actually has). This is what keeps the
model from inventing assets that don't exist.

## 4. The prompt roster

| Builder | Persona | Blocks (order) | Schema | Notes |
|---|---|---|---|---|
| `buildScenePrompt` | RITA | Reader→Cast→Now→Context→Lore→SceneSoFar→Tail | `scene` | `cacheKey: playthroughId` |
| `buildSoloPrompt` | RITA | same, solo rules | `scene` | solo |
| `buildContinuationPrompt` | RITA | StorySoFar→SceneSoFar→Tail+ordinal | `scene` | |
| `buildClosingPrompt` | RITA | SceneSoFar→Tail(goodbye) | `scene` | |
| `buildSlotIntroPrompt` | RITA | Opening→Recently→weather→campus→cast→askers→breakups→posts→rumor→Tail | `slotIntro` | `kind: slotIntro` |
| `buildLedgerPrompt` | ledger persona | Money→Expulsion→Plans→Memories→Events→Acts→Stats→class | `ledger` | resolver |
| `buildTextingPrompt` | texting persona | Reader→Character→Now→where→Lore→Thread→Messages→Tail→Blocking→Summary | `texting` | `cacheKey:"texting"` |
| `buildHangoutClassifierPrompt` | *(empty)* | Now→Background→Message→Reply→rules+examples | `hangoutClassifier` | `minThinking: low` |
| `buildClassifierPrompt` | classifier system | Roster→classes→projects→locations→sentence | verdict | `minThinking: low` |
| `buildQuizPrompt` | exam writer | Facts | `exam_quiz` | `kind: examQuiz` |
| `buildCharacterPrompt` | character RITA | Personality→Behavior×5→Backstory→LoveLife→Likes→Gifts→Traits→Attraction→Voice→Appearance→Outfit→PE→Swim→Room→Pose→Expressions→brief | character | one-time |

## 5. Model routing

```ts
interface ModelRoute {
  writer: string                 // the main model
  secondaryFor?: string[]        // task kinds that use the cheap model
  thinking: ThinkingLevel        // minimal | low | medium | high
}
```

Source: `PROMPT_KINDS = [solo, examQuiz, slotIntro, texting]`;
`DEFAULT_SECONDARY_KINDS = [solo, examQuiz]`. `modelToRun` picks the model; `reasoningToSend`
picks the thinking level (with floors like the hangout classifier's `low`).

## 6. Caching, cancellation, middleware

- **Caching:** `cacheKey` marks the stable prefix; the variable tail goes last. The source uses
  `playthroughId` for scenes and constants for texting/quiz.
- **Cancellation:** `group` ids (`SCENE_LLM_GROUP`, `LOOP_LLM_GROUP`) abort a whole group.
- **Middleware:** `LlmMiddleware.onRequest/onResponse` for auth, headers, retries, **format
  translation** (e.g. `json_schema` → `json_object` for DeepSeek), logging, cost. The
  header-injecting proxy from this project is exactly such a plugin.

## 7. The persona

The source's "RITA" persona is a compact bundle of **voice + anti-slop rules**:

> "You are RITA, author of steamy reader-POV romance fiction... present-tense, second person...
> you really, really HATE repetitive, robotic slop-writing... you NEVER use semicolons or
> emdashes... you always return a single, fully-formed JSON object matching the provided schema
> exactly."

Generalize as a `Persona` content asset (swappable per genre) with variants (SFW, ledger,
texting).

## 8. Source map

| Concept | Source symbol |
|---|---|
| Assembly | `castScenePrompt`, `systemPrompt`, `jsonRules`, `whoBlock`, `nowBlock`, `classBlock` |
| Personas | `PERSONA`, `LEDGER_PERSONA`, `SFW_PERSONA`, `TEXTING_PERSONA`, `scenePersonaFor` |
| Schema helpers | `objectSchema`, `enumField`, `enumArray`, `stringArray` |
| Routing | `modelToRun`, `providerToRun`, `reasoningToSend`, `PROMPT_KINDS` |
| Seed word | `seedWordBlock`, `SEED_WORDS` |
| Lorebook | `lorebookBlock`, `characterLoreFor`, `alwaysLore` |

---

**Next:** [10 — Content & mods](./10-content-and-mods.md) — packs, custom content, ComfyUI setup.
