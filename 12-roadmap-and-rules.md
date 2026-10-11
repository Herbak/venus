# 12 — Roadmap & rules

[← Index](./README.md) · [Worked example](./11-worked-example-combat.md) · [Appendix →](./appendix-a-source-reference.md)

How to get from the source game to the plugin/engine architecture, and the design rules to hold
onto.

---

## 1. Keep vs parameterize

### Already generic in concept (keep)
- The structured-output contract (`jsonRules` + schema builder).
- Block-composed prompt assembly + `cacheKey` + seed word.
- The classifier → cast → scene → ledger → apply pipeline.
- Record+overlay persistence and the memory-weighted relationship math.
- The directive reducer (`stepStage`) and validator (`sanitizer`).
- Model routing, streaming, cancellation, covers.

### Hardcoded (parameterize)
- `EMOTIONS`, `STOCK_POSITIONS`, `ACT_KINDS`, `EVENT_GLOSS`, `STAT_KEYS` → content-pack enums.
- `beginSlot`'s fixed upkeep calls → registered `System`s.
- `castTurn`'s `if actionType === ...` chain → a **cast strategy dispatch table**.
- `sceneSchema` / `ledgerSchema` fields → composed from enabled modifier/schema plugins.
- The campus `SETTING` / persona → pack data.

## 2. Phased migration

| Phase | Deliverable | From |
|---|---|---|
| 1 | Extract the two primitives (`CALL`, `REDUCE`) | `completeStructured`, `applyLedger` |
| 2 | Put scene / ledger / classifier behind interfaces | `streamScene`, `buildLedgerPrompt`, `classifyAction` |
| 3 | Introduce the modifier chain; route SFW/traits/mood through it | `personaFor`, `hasTrait`, `moodLine`, `noNsfwImages` |
| 4 | Introduce the loop stack; move texting/exams/jobs to `System`s | `deliverSlotHangouts`, `startExam`, `settleShifts` |
| 5 | Make directives/entities/content data-driven | `sceneSchema`, `quickstart.json`, `character.json` |
| 6 | Register combat as a second `LoopPlugin` | new |

Each phase is shippable; the game keeps working throughout.

## 3. Design rules

1. **The LLM never mutates state.** All changes go through `REDUCE`.
2. **One call, one schema.** No free-form parsing of prose into state.
3. **Stable prefix, variable tail.** `cacheKey` for prompt caching; seed word near the end.
4. **Validate every directive** against the live registry before it reaches the presenter.
5. **Separate playback from loop logic.** `advance()` is presenter; loops just hand it beats.
6. **Deltas are the only cross-loop boundary.** Loops don't reach into each other's state.
7. **Version everything.** Schemas, saves, packs carry `schemaVersion` + migrations.
8. **Content packs are data + declared engines.** No arbitrary code execution.
9. **Deterministic-first, LLM-fallback** for classification; gate on failure.
10. **Order matters** in the modifier chain; document the default order.

## 4. Pitfalls

| Pitfall | Mitigation |
|---|---|
| Determinism vs LLM | keep transitions pure; never trust prose |
| Schema drift | version + migrate; the source uses `schemaVersion` |
| Token budgets | per-block budgets (`memoryBudgets`), trim before send |
| Directive hallucination | validate against the registry; drop unknowns |
| Save compat across plugins | journal deltas, namespace by plugin id |
| Mod sandboxing | packs declare engines; no code execution |
| Interrupt semantics | define suspend/resume + time cost per handoff |
| Streaming desync | discard preview if streamed ≠ parsed count |
| Asset invention | build enums per scene from the registry |
| Provider quirks | `LlmMiddleware` for headers/format (e.g. `json_schema` → `json_object`) |

## 5. Open design questions

- **Time cost of a loop handoff:** does combat consume a slot? Is it free? Configurable per loop?
- **Multi-actor scenes at scale:** the source caps portraits (`PORTRAIT_SLOTS`); how does combat
  with many combatants map to the same view?
- **Resolver cost:** one resolver per scene is fine; a turn-based combat may need per-round
  resolution — is that a `System` or a `Resolver`?
- **Cross-loop memory:** how do combat outcomes feed the relationship engine (trauma, bonding)?
- **Deterministic replay:** journaling all deltas enables replays — do combat rolls need their
  own seeded stream?

## 6. Suggested next artifacts

- A **minimal kernel** (the `CALL`/`REDUCE`/registry in code).
- A **combat pack skeleton** wired to these interfaces.
- A **prompt/schema DSL** so packs can declare blocks + schemas as data.
- A **validator/plugin test harness** (feed synthetic beats, assert view state).

---

[← Back to index](./README.md) · [Appendix →](./appendix-a-source-reference.md)
