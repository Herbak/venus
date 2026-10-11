# 04 — Intent engine

[← Index](./README.md) · [Scene engine](./03-scene-engine.md) · [Next: Resolver engine →](./05-resolver-engine.md)

The intent engine turns **free text** ("I take Sarah back to my room and shut the door") into a
**structured verdict** the loop can act on. It is deliberately a *separate, cheap* call from
the expensive scene writer.

---

## 1. Contract

```ts
interface IntentEngine {
  classify(input: string, ctx: Context): Promise<Intent>
}

interface Intent {
  actors: ActorId[]          // who is present / being interacted with
  mentions: ActorId[]        // named but not present
  actionKind: string         // open enum: "" | "goto_class:BIO 210" | "job" | "project:ART 110" | "combat" ...
  place?: string             // scene location (id or free words)
  visibility: "public" | "private"
  duration?: string          // a time bucket: "instant" | "short" | "medium" | "long" | "very_long"
  annotations?: Record<string, unknown>   // other closed-set facts (cost, involves, mood, danger…)
  extra?: Record<string, unknown>   // engine-specific (target, attackKind, ...)
}
```

## 2. The source verdict (reference shape)

`buildClassifierPrompt` returns a 9-field verdict:

```
{ characters[], mentionedOnly[], goingToClass, classCode,
  goingToWork, workingOnProject, projectClassCode, inPublic, sceneLocation }
```

`CLASSIFIER_SYSTEM` is a careful, example-driven prompt ("You read one sentence describing what
someone does and report nine things about it") with ~10 worked examples. Key rules baked in:

- **Naming ≠ being present:** `characters` (in the scene) vs `mentionedOnly` (named but absent).
- **`inPublic`** gates the witness roll; default `true` when unspecified.
- **`goingToClass` / `goingToWork` / `workingOnProject`** are mutually exclusive priority rules.
- **`sceneLocation`** prefers a known location id; otherwise plain words ("the river", "beach").

The verdict drives **casting** (`castTurn` dispatches on `actionType`:
`project:` / `goto_class:` / `job` / people) and the scene's `sceneLocation` + `inPublic`.

## 3. Deterministic-first, LLM-fallback

`classifyAction`:
1. If the roster is empty, short-circuit to an empty verdict — **no call**.
2. Otherwise call the classifier.
3. On failure, open a **gate** that waits on the player (`classifierGate.ask`) and retry —
   the loop never proceeds on a missing verdict.

**Generalization:** the same pattern serves any intent engine. Keep a deterministic
short-circuit for the trivial cases and a gate for failures.

## 4. Generalizing `actionKind` (the key change)

In the source, `actionType` is a **closed set** consumed by an `if actionType === ...` chain in
`castTurn`. For a plugin engine, make it an **open registry**:

```ts
type ActionKind = string   // "goto_class:BIO 210", "job", "project:ART 110", "combat", "travel:..."
```

Each loop/scene registers the kinds it handles and a **cast strategy**:

```ts
interface CastStrategy {
  matches(actionKind: string): boolean
  cast(intent: Intent, ctx): CastResult
}
```

So "add combat" = register `combat` as an `actionKind` + a `CastStrategy` + a `LoopPlugin`.
No edits to the VN loop.

## 5. The second classifier (hangout)

The source also has a **hangout classifier** — a *post-hoc* judge that reads one text exchange
and decides whether the player asked to meet up **now** and the character agreed:

```
hangoutClassifier: { playerAsked: boolean, characterOffered: boolean, description: string }
```

It runs on `minThinking: "low"` and turns "meet now" into a calendar **plan** with a
`description` like "Coffee at the student union with Mina and Mia."

**Pattern to keep:** a classifier that converts *conversational content* into a *world state
change* (a plan) is a reusable idea for any genre (an invitation, a challenge, a truce).

## 6. The annotation call (structured-only)

The intent call is the first of a family: **structured-only LLM calls** that return **no prose** —
just a schema-shaped object of **closed sets** (enums, multi-selects, booleans). They are the
cheapest, most reliable, most cacheable use of an LLM, and the engine already has the slot.

The one a [continuous clock](./02-kernel.md#12-continuous-a-24-hour-timer) needs is a **time /
cost annotator**: given the player's action, it reports the numbers and enums the systems need —
and **never a free-text number**.

```jsonc
// the annotation schema (no free text anywhere)
{
  "actionKind": "enum: talk | social | class | job | project | travel | rest | idle",
  "present":    "array (enum): <charKey>",
  "location":   "enum: <location id> | ''",
  "duration":   "enum: instant | short | medium | long | very_long",
  "involves":   "array (enum): eat | drink | study | flirt | exercise | shop | travel",
  "cost":       "enum: none | small | medium | large"
}
```

The **duration bucket maps to minutes in data** (`instant 0 · short 15 · medium 30 · long 60 ·
very_long 120`) — a kit param, **not** an LLM computation. The multi-select (`involves`) is the
most reliable form: the model only **classifies** ("does this involve eating? travel?"), and the
data turns the picks into a cost. **The model never does arithmetic.**

**Robustness:**
- **Validate** against the enum; drop unknown values.
- **Fallback** — on failure, use the `actionKind`'s default cost from a table.
- **Cache** — a `cacheKey` (the same action text recurs).
- **Cheap model** — a `minimal`/`low` thinking call, like the classifier.

**Generalization — an "Annotation" engine.** Time is one annotation. The same structured-only call
returns **money, effort, mood, danger, tags, who's present, where** — all closed sets. So instead
of a bespoke time estimator, one **Annotation engine** (an extension of Intent) annotates a player
action with every numeric/enumerable fact the systems need. The split becomes:

> **the writer writes prose · the annotator writes numbers · the resolver writes state.**

## 7. Source map

| Concept | Source symbol |
|---|---|
| Classifier entry | `classifyAction`, `classifyOnce`, `buildClassifyAttempt` |
| Prompt | `buildClassifierPrompt`, `CLASSIFIER_SYSTEM` |
| Gate | `classifierGate`, `EMPTY_VERDICT` |
| Consumer | `castTurn` dispatch table |
| Hangout judge | `buildHangoutClassifierPrompt`, `hangoutClassifierSchema`, `normalizeHangout` |
| Annotation (pattern) | the classifier + hangout judge are **structured-only** calls; a time/cost annotator is the same shape (see §6) |

---

**Next:** [05 — Resolver engine](./05-resolver-engine.md) — turning a finished scene into state.
