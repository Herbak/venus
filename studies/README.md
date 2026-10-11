# Studies

Applied design studies: taking the engine from [the main docs](../README.md) and working out how
to build a **different game** from the same bricks — which existing engines/systems to reuse,
and which new plugins to write.

| # | Study | Premise | Scope |
|---|-------|---------|-------|
| 01 | [Onsen management](./01-onsen-management.md) | Run a hot-spring inn day by day | **A small step** from the campus sim — mostly re-skinning the existing loop + resolver |
| 02 | [True RPG: combat + VN stories](./02-rpg-combat-and-stories.md) | Explore, fight, level up, and live story scenes | **The full showcase** — the loop stack, multiple scene kinds, combat + RPG systems |
| 03 | [Detective / mystery](./03-detective-mystery.md) | Solve a case: investigate, interrogate, deduce | **Information as the verb** — evidence, alibis, a deterministic verdict over LLM prose |
| — | [Plugins catalogue](./plugins-catalogue.md) | Every plugin named across the docs | The aggregate manifest, also read by the partner agent |

Related: **[13 — The creator app](../13-creator-app.md)** — how a non-coder builds/remixes these
plugins (variables, rules, prompts, and a partner LLM agent).

Each study follows the same shape:

1. **Pitch** — what the game is.
2. **What changes vs the source** — the deltas from Venus University.
3. **Bricks mobilized** — every existing engine/system and *how* it's reused.
4. **New plugins/engines** — the plugins to write, per slot, with responsibilities.
5. **New directives, schemas, entities** — the vocabulary the engine gains.
6. **Loop diagram** — control flow.
7. **Worked flow** — one turn/day end-to-end.
8. **Build order** — the cheapest path from the source to a playable slice.

Read [01 — Overview](../01-overview.md) first if the terms (kernel, loop stack, directive,
modifier) aren't fresh.
