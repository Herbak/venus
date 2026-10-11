# 05 — Resolver engine (the "ledger")

[← Index](./README.md) · [Intent engine](./04-intent-engine.md) · [Next: Loop engine →](./06-loop-engine.md)

The resolver is the **second pass**: after a scene is written, a *separate* structured call
reads it and emits **durable state** — memories, events, money, stats, plans. This decoupling
(the writer writes prose; the resolver writes state) is the single biggest idea in the source
game.

---

## 1. Contract

```ts
interface ResolverEngine {
  resolve(scene: SceneReply, ctx: Context): Promise<Deltas>
}

interface Deltas {
  stats?: Record<string, number>
  resources?: Record<string, number>          // money, hp, ...
  events?: DomainEvent[]                       // relationship transitions, milestones
  memories?: MemoryEntry[]                     // weighted history
  plans?: Plan[]                               // future appointments
  flags?: Record<string, unknown>              // per-actor flag transitions
  summary?: string                             // for the record
}
```

## 2. The source ledger

`buildLedgerPrompt` → `ledgerSchema`. Its `user` blocks, in order:

```
MONEY      → total dollars the reader spent this scene (integer ≥ 0)
EXPULSION  → was he CAUGHT doing something expellable? (boolean, false almost always)
PLANS      → future appointments (only when there's a cast)
MEMORIES   → one durable memory per character (type ∈ liked/disliked/loved/hated)
EVENTS     → milestones that happened THIS scene (from a closed gloss list)
ACTS       → kisses and sex (kind ∈ kiss/sex, inPublic, chars[])
STATS      → which of brain/body/heart the scene exercised
classSummary / classFactoid  → for a class scene
```

Schema (abridged):

```
ledger: {
  memories: [{ charKey, type ∈ MEMORY_TYPES, desc }],
  events:   [{ charKey, event ∈ EVENT_GLOSS }],
  acts:     [{ kind ∈ ACT_KINDS, inPublic, chars[] }],
  spent:    integer ≥ 0,
  expelled: boolean,
  plans?, stats?, classSummary?, classFactoid?
}
```

Then `applyLedger(ledger)` folds it into the world: flags via `applyEvent`, memories appended,
money debited, expulsion set, plans added, NPC bookkeeping. `resolveScene()` derives **stat
deltas** from the ledger + context (solo bonus, class outcome, job shift, grade outcome) and
`applyStatDeltas` clamps them.

## 3. Why two passes

- The writer stays focused on **prose quality**; the resolver on **schema fidelity**.
- The resolver can run at a **different model/thinking level** (often cheaper).
- The resolver sees the **whole finished scene**, so it can judge consequences holistically.
- State stays **deterministic and auditable** — prose never mutates the world directly.

This is also why the game can run a **memory-edit gate** after the ledger: the resolver's
memories are surfaced to the player for correction before they're committed.

## 4. The relationship engine (a resolver consumer)

Affection is **computed, not stored** — the resolver just appends typed memories; the score is
derived:

```
affection = Σ memories  POINTS[type] × weightFor(daysAgo)   +  likedPostBonus
POINTS    = { loved:+2, liked:+1, disliked:-1, hated:-2 }
weightFor = ≤7 days: ×4,  ≤30 days: ×2,  else ×1
```

Memories merge from four sources (`mergedMemories`): base + `textMemory` + `jealousyMemories`
+ `giftMemories`. The score maps to a **disposition** (`friendly 15`, `trusted 30`,
`devoted 50`, symmetric negatives → `annoyed`/`hostile`) which drives which authored behavior
stage the character is written in (`behaviorLevelOf`).

**Generalization:** "relationship = weighted sum of typed memories" works for trust, fear,
reputation, faction standing, karma. Keep it — it's genre-neutral.

## 5. Source map

| Concept | Source symbol |
|---|---|
| Prompt | `buildLedgerPrompt`, `ledgerSchema`, `LEDGER_PERSONA` |
| Apply | `applyLedger`, `applyEvent`, `recordAcademics` |
| Stat deltas | `resolveScene`, `resolveStatDeltas`, `applyStatDeltas`, `tierUps` |
| Memory/affection | `affectionFor`, `affectionOf`, `mergedMemories`, `POINTS`, `weightFor`, `dispositionOf` |
| Gate | `askMemoryEdits`, `fileMemoryEdits`, `sceneMemoryRows` |

## 6. Extension points

- **New deltas:** a combat resolver returns `resources.hp`, `events` (injury), `memories`
  (trauma/bond). The kernel applies them generically.
- **New memory types / events:** register them; the affection math and flags react via
  subscriptions.
- **Per-system resolvers:** a loop can own its own resolver (combat resolves HP; the VN loop
  resolves relationships).

---

**Next:** [06 — Loop engine](./06-loop-engine.md) — hosting loops and parallel systems.
