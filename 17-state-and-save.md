# 17 — State & save format

[← Schemas & prompts](./16-schemas-and-prompts.md) · [Index](./README.md) · [Next: Image pipeline →](./18-image-pipeline.md)

The **world state** and the **persistence chain** — what to store, how it's split, and how it
evolves. Extracted from the source (Venus University v0.4.0).

---

## 1. The two-layer state model

The source splits state into a **record** (the world seed) and an **overlay** (the delta):

| Layer | Written | Holds | Why |
|---|---|---|---|
| **Record** | once, at enrollment | characters, classes, profiles, jobs, occasions, weather table, job closures, haunts, feeds, spring-break plans | the immutable world |
| **Overlay (save)** | every slot | date/time, stats, money, charInfo (flags/memories), schedule, history, bunnyboard, events, job, inventory, classRecords, relationships, … | the mutable delta |

A save is the overlay **rejoined** with the record at load (`loadSave(save, record, characters)`).

## 2. The save shape (overlay)

From `useGameStore`'s `initialState`, the fields a save carries:

```
playthroughId, date, time,
chars[], playerFirstName, playerLastName, bio, tallies,
stats { brain, body, heart }, money,
charInfo { <charId>: { memories[], flags{}, nameKnown, year, dorm, major, schedule{}, gifts[] } },
classes{}, playerSchedule{}, history{}, replays{},
bunnyboard { conversations{}, requestsReceived[], requestsSent[], suggestions[], … },
events[], occasions[], occasionsDeclined[],
job, jobsClosed[], inventory[], classRecords{},
gradesStanding, expelled, midtermStandingDone, finalsScoresShown,
npcRelationships{}, npcFriendships[], npcOverlay,
slotRumor, lastSlotCast, weekendOutings{}, outingSlots{}, springBreakAway,
feedExtras, graduationSeen, farewellsDone[], endingArtWanted,
…feature flags (venusThrough, bunnybotThrough, bunnymapUnlocked, bunnyshopUnlocked, …)
```

Scene-scoped state (the **live scene**, not the world) is captured separately (`captureScene`)
so a mid-scene save can restore the transcript, stage, and pending lines.

With a [continuous clock](./02-kernel.md#12-continuous-a-24-hour-timer), `date` + `time` collapse
into a single **timestamp**; a **real-time** model additionally stores `lastPlayedAt` so elapsed
wall-clock time can be applied on load (offline progression).

## 3. Flags (the relationship state machine)

`charInfo[charId].flags` — `emptyFlags()`:

```
hasMet, hasCrush, friendZoned, friendZonedBy, isLover, brokenUp (count),
hasKissed, hadSex, benefits, harem, gaveContactInfo, blocked,
knowsTraits, knowsBackstory, knowsLoveLife
```

`applyEvent(flags, event)` transitions them from ledger events:

| Event | Effect |
|---|---|
| `kissed` | `hasKissed = true` |
| `sex` | `hadSex = true`, `hasKissed = true`, `benefits = !isLover` |
| `became_lovers` | `isLover = true` (voids lesser arrangements) |
| `broke_up` | `brokenUp += 1`, `isLover = false` |
| `friendzoned_by_reader` / `friendzoned_reader` | the corresponding `friendZoned`/`friendZonedBy` |
| `gave_contact_info` | `gaveContactInfo = true` |
| `unblocked` | `blocked = false` |
| `agreed_to_harem` | `harem = true` |

Traits seed some flags (`Promiscuous → harem`).

## 4. Memories & affection

```
affection = Σ memories  POINTS[type] × weightFor(daysAgo)  + likedPostBonus
POINTS    = { loved:+2, liked:+1, disliked:-1, hated:-2 }
weightFor = ≤7 days: ×4,  ≤30: ×2,  else ×1
tiers     = friendly 15, trusted 30, devoted 50 (symmetric negatives)
```

Memories merge from four sources (`mergedMemories`): base `memories` + `textMemory` +
`jealousyMemories` + `giftMemories`. Disposition → `behaviorLevelOf` → which of the 5 authored
behavior stages the character is written in.

## 5. The persistence chain

```
playthrough/                          (folder, one per run)
  record.json                         world seed
  enrollment.json                     course registration draft
  saves/
    slot-<n>.json                     ≤9 manual + autosave slots
  replays/<date>/<slot>.json          word-for-word scene history
  photos/                             captured photos + sidecars
```

Source functions: `writeSlotSave`, `writeAutosave`, `listPlaythroughs`, `readPlaythroughRecord`,
`stampRecord`, `MAX_SLOT_SAVES = 9`. Settings + encrypted keys live in `data/settings.json`.

**Save points** (the VN loop): at every **slot crossing** (`writeSlotSave`) and mid-scene
(`writeAutosave`). A loop pushed on the stack **suspends** to a blob so a save mid-combat reloads.

## 6. Versioning & migrations

Every record carries `schemaVersion`:

```
CHARACTER_SCHEMA_VERSION = 3
SAVE_SCHEMA_VERSION      = 12
RECORD_SCHEMA_VERSION    = 3
ENROLLMENT_SCHEMA_VERSION= 2
SAVED_SCENE_SCHEMA_VERSION
CUSTOM_BACKGROUND_SCHEMA_VERSION = 1
```

On load, older shapes are upgraded (`upgradeEndpointModels`, `upgradeNsfwSound`, character
migration at `schemaVersion 2 → 3`). A validator rejects a shape the build doesn't read.

## 7. Determinism & journaling

- The **record + overlay** split makes saves small and migrations tractable.
- **Journal deltas** per plugin id so a combat/inventory kit's state coexists with the VN's.
- All transitions go through `REDUCE`; the journal is the delta log — the basis for **replays**
  (the source stores full scene transcripts per slot) and for crash recovery.

---

**Next:** [18 — Image pipeline](./18-image-pipeline.md).
