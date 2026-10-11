# 03 — Scene engine (the main engine)

[← Index](./README.md) · [Kernel](./02-kernel.md) · [Next: Intent engine →](./04-intent-engine.md)

The scene engine is **"play a scene"**: assemble a prompt, make one structured LLM call,
validate the reply, and apply its directives. Everything the player reads comes through here.

---

## 1. Contract

```ts
interface SceneEngine {
  build(req: SceneRequest, ctx: Context): PromptSpec
  run(req: SceneRequest, ctx: Context): Promise<SceneReply>
}

interface SceneRequest {
  sceneKind: string          // "dialogue" | "combat" | "exam" | "cutscene" | plugin-defined
  actors: ActorId[]          // who is present
  action: PlayerAction       // what the player just did
  state: WorldView           // read-only projection for prompt building
  options?: SceneOptions     // genre/sfw/... (modifiers read this)
  continuation?: boolean     // is this a follow-up turn?
}

interface PromptSpec {
  persona: string
  blocks: PromptBlock[]      // ordered
  schema: JsonSchema
  cacheKey?: string          // stable prefix for prompt caching
  route?: ModelRoute         // writer vs cheap model, thinking level
  group?: string             // for cancellation
  images?: ImageRef[]        // multimodal inputs
}

interface SceneReply {
  beats: Beat[]              // was `lines`
  summary?: string
  end?: boolean
}
```

## 2. The `Beat` / `Directive` model

The single most important generalization. The source's `lines[]` becomes a typed directive
stream:

```ts
interface Beat {
  speaker: ActorId | ""        // "" = narration
  text: string
  directives?: Directive[]     // applied in order
  bg?: string                  // convenience shorthand for a bg directive
}
type Directive = { kind: string; [k: string]: unknown }   // EXTENSIBLE
```

Source directives: `show:<key>`, `hide:<key>`, `sprite:<key>,<emotion><_outfit>`,
`cg:<position>`, plus the `bg` field.

**Extension:** a combat engine registers `attack:`, `defend:`, `item:`, `flee:` — the same
pipeline carries them. The **directive vocabulary is built per scene from the registry** (which
backgrounds exist, which outfits/CGs an actor has), exactly as the source builds `bg`/`sprite`
enums dynamically.

## 3. The run pipeline

```
build(req)                 → PromptSpec  (persona + blocks + schema)
   │
   ├─ LLM stream (llm.stream)
   │     └─ line extractor  (tokens → whole beats, e.g. createLineExtractor)
   ├─ validator[]           (createSceneSanitizer + splitOverflow)
   │     └─ drop unknown speaker/bg/directive, enforce limits, split long lines
   ├─ apply directives      (stepStage → presenter)
   └─ resolve summary/end
```

### 3.1 Prompt assembly
See [Prompt system](./09-prompt-system.md). The scene's block order (from the source):

```
SYSTEM:  persona → "" → JSON RULES (output contract) → "" → SETTING
USER:    READER → CAST → NOW → CLASS → LORE → SCENE SO FAR → YOUR TURN (+ seed word + action)
```

### 3.2 Streaming + line extraction
The source streams tokens (`llm.onSceneDelta`) and reassembles whole JSON beats with an
extractor, holding a **floor** so the first beat isn't instant (`owedFloor`), and hiding the
gap behind a **transition cover** (`coverSceneOpening`). If the streamed count disagrees with
the final parsed count, the preview is discarded and the parsed beats applied wholesale.

### 3.3 Validation
`createSceneSanitizer` is the reference validator: unknown speaker → narration; unknown
background → keep current; unknown character in a directive → drop it; `show:` past
`PORTRAIT_SLOTS` → drop; `sprite:` for an unrendered outfit → fall back to the default;
`cg:` without exactly one eligible on-screen actor → drop. `splitOverflow` splits a too-long
line at sentence boundaries using `boxFits`.

**Generalize as `Validator[]`** — ordered, each returns a cleaned beat or drops it.

### 3.4 Application
Validated beats go to the [Presenter](./08-presenter-engine.md), which folds directives into
view state (`stepStage`) and drives playback (`advance`).

## 4. Scene variants

The source has four scene prompt variants that reuse `castScenePrompt`:

| Variant | Builder | Tail |
|---|---|---|
| Opening (cast) | `buildScenePrompt` | "write a kickass scene…", end not set |
| Solo | `buildSoloPrompt` | narration-only JSON rules, one-shot |
| Continuation | `buildContinuationPrompt` | "continue from SCENE SO FAR", ordinal action count |
| Closing | `buildClosingPrompt` | "and cut!", goodbye + `hide:` everyone |

A scene is a **sequence of continuation turns** until the model sets `end` (or a solo scene
completes in one). A continuation turn is also what an **interjection** produces when the
player cuts in mid-scene (see [Presenter §3](./08-presenter-engine.md#interjection-interrupting-a-scene)).

## 5. Model routing & caching

- **Route:** `writer` model for prose; a cheaper `secondary` for some kinds
  (`PROMPT_KINDS = [solo, examQuiz, slotIntro, texting]`, `DEFAULT_SECONDARY_KINDS = [solo,
  examQuiz]`). Thinking level per call.
- **Cache:** `cacheKey` marks the stable prefix (the source uses `playthroughId` for scenes,
  `"texting"`, `"quiz"`...). Put the variable tail last.
- **Cancel:** `group` ids (`SCENE_LLM_GROUP`, `LOOP_LLM_GROUP`) let the loop abort a whole
  group (e.g. when the player leaves).

## 6. Source map

| Concept | Source symbol |
|---|---|
| Scene engine entry | `streamScene`, `completeStructured` |
| Prompt builders | `buildScenePrompt`, `buildSoloPrompt`, `buildContinuationPrompt`, `buildClosingPrompt`, `castScenePrompt` |
| Output contract | `jsonRules`, `sceneSchema` |
| Streaming/extraction | `createLineExtractor`, `deliverSceneLines`, `queueLines`, `owedFloor` |
| Validation | `createSceneSanitizer`, `splitOverflow`, `boxFits`, `sanitizeScene` |
| Application | `stepStage`, `stageAt`, `advance` |
| Turn wrapper | `runSceneTurn` |

## 7. Extension points

- **New scene kind:** register a `SceneEngine` (or a variant) with its own schema + directives.
- **New directives:** add to the per-scene directive vocabulary; the presenter + validator must
  know them.
- **New validators:** drop-in to the `Validator[]` chain (e.g. a profanity filter, a
  content-rating gate).
- **New blocks:** see [Prompt system](./09-prompt-system.md).

---

**Next:** [04 — Intent engine](./04-intent-engine.md) — how free text becomes structured intent.
