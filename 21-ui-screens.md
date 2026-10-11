# 21 — UI screen inventory

[← Build a Venus](./20-build-a-venus.md) · [Index](./README.md)

Every screen the base game needs, what it does, and how it enters. Extracted from the source
(Venus University v0.4.0).

---

## 1. Menu & setup

| Screen | Purpose | Enters from |
|---|---|---|
| **Boot / loading** | asset + setup checks | launch |
| **Title** | title art, music, "Press start" | boot |
| **Main menu** | New Game · Load · Manage Characters · Setup · Settings · Credits · Support · Feedback | title |
| **Setup** | ComfyUI install/verify (models + nodes), progress | main menu |
| **Settings** | provider/model, keys, sampling, memory budgets, `scenePersona`, volumes, SFW toggles, updates | menu / in-game |

## 2. Characters

| Screen | Purpose |
|---|---|
| **Manage characters** | list of characters with per-emotion/outfit/CG state (`unwritten`→`unfinished`→`ready`), create, duplicate, export (ZIP / SillyTavern card + sprites), delete |
| **Create / edit character** | the big form (personality, behavior×5, backstory, love life, likes, gifts, traits, attraction, voice, appearance, outfits, room, pose, expressions) + Generate |
| **Generate modal** | brief, name suggestions, seed word, reference image, pose pick |
| **Expression / outfit / CG status** | which sprites exist; "resume" partial generations; hands-fix |

## 3. New game

| Screen | Purpose |
|---|---|
| **New playthrough** | name the run, pick the player, enroll |
| **Enrollment** | choose classes (major/interest), see the schedule |
| **Orientation** | the tutorial slot (first-scene narration + a friendly intro) |

## 4. The game (VN)

| Screen | Purpose |
|---|---|
| **Landing** (between slots) | date/weather banner, money + stat radar, apps (Bunnyboard, Calendar, Shop, Map), "**What would you like to do?**" + slot actions (PLANS/CLASS/WORK/PROJECT) + a "Something else?" field |
| **Scene** | the stage (bg, portraits, CGs), the **dialogue box** (speaker, text, hints), the **action box** ("What do you do?") + the interject divider, the **rail** (Calendar, Bunnyboard, Settings, Background, Cast, ChatLog), the **gift** control |
| **Quiz** | an exam's multiple-choice questions (A–D buttons) |
| **Status / result** | post-scene stat movement, money, memories, milestones |
| **Memory edit** | review/edit the memories the ledger recorded |
| **Clock / day planner** | a running clock, today's plan by time, and (optionally) a per-action **time-cost preview** when a [continuous clock](./02-kernel.md#12-continuous-a-24-hour-timer) is used |

## 5. The apps (Bunnyboard etc.)

| Screen | Purpose |
|---|---|
| **Bunnyboard — Chats** | the DM threads (typing indicator, send, hangout "Yes/No", block notices) |
| **Bunnyboard — Feed** | status posts, likes, teasers |
| **Bunnyboard — Friends** | contacts, incoming/outgoing requests, discover suggestions |
| **BunnyShop** | spend currency on gifts/items |
| **BunnyMap** | where characters are; travel/visit |
| **Calendar** | the week's schedule, plans/events, reschedule |

## 6. Meta

| Screen | Purpose |
|---|---|
| **Save / Load** | slot saves (≤9), autosave, thumbnails, delete |
| **Photos** | captured photos: viewer, develop progress, export |
| **Settings (in-game)** | as above, plus model routing and persona |
| **ChatLog** | the scene transcript for the current slot |
| **Endings / epilogue** | graduation scroll, farewell menu, ending CG, final status |
| **Game over** | the three authored game-overs + a confirm |

## 7. Presenter mapping

| Screen | Presenter |
|---|---|
| Scene | `VnPresenter` (stage reducer + dialogue box + playback) |
| Landing | `LandingPresenter` (banner, radar, apps, ASK) |
| Clock / planner | `ClockPresenter` (time display + day plan + cost preview) |
| Bunnyboard | `BunnyboardPresenter` |
| Calendar / Shop / Map | `CalendarPresenter` / `ShopPresenter` / `MapPresenter` |
| Manage characters | `CharacterPresenter` |
| Quiz / Status | `QuizPresenter` / `StatusPresenter` |
| (kit screens) | `CombatPresenter`, `CaseBoardPresenter`, `ManagementPresenter`, … |

A kit that adds a sub-system adds its **own presenter** — the base screens are unchanged.

---

[← Back to index](./README.md)
