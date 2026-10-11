# 20 — Build a Venus (from scratch)

[← Kits](./19-kits.md) · [Index](./README.md) · [Next: UI screens →](./21-ui-screens.md)

A step-by-step path from nothing to a playable Venus-like game, using the pieces in
[15](./15-reference-kernel.md)–[19](./19-kits.md). Each milestone is playable.

---

## 1. The minimum kit set

A Venus-like game is the **VN base** plus these kits (from [19 §4](./19-kits.md#4-the-base-venus-kit-set)):

```
base-romance   (the VN loop + scene + ledger + relationships)
characters     (character generation + the image pipeline)
texting        (Bunnyboard)
classes        (classes, projects, exams, grades)
jobs           (shifts, wages)
occasions      (holidays, weather, weekends, spring break)
social-sim     (rumors, crushes, friendships, NPC overlay)
endings        (graduation, epilogue, farewells)
```

`photos` is optional. Everything else is opt-in.

## 2. Milestones (each is playable)

| M | Goal | Uses |
|---|---|---|
| **M0** | Kernel boots; one hard-coded scene plays | [15](./15-reference-kernel.md) |
| **M1** | VN base: day/night `ScheduleLoop`, scene prompt + schema, presenter, save | [16](./16-schemas-and-prompts.md), [17](./17-state-and-save.md) |
| **M2** | Characters: `character.json` + art (ComfyUI or placeholders), manage UI | [18](./18-image-pipeline.md), [21](./21-ui-screens.md) |
| **M3** | Relationships: ledger → flags/memories/affection | [05](./05-resolver-engine.md), [17](./17-state-and-save.md) |
| **M4** | Slot intro + texting (Bunnyboard) | [04](./04-intent-engine.md), [16](./16-schemas-and-prompts.md) |
| **M5** | World: classes, jobs, occasions, weather | [06](./06-loop-engine.md) |
| **M6** | Social sim: rumors, crushes, friendships, NPC overlay | [06](./06-loop-engine.md) |
| **M7** | Endings: graduation, epilogue, farewells | [08](./08-presenter-engine.md) |
| **M8** | Polish: photos, settings, creator app | [13](./13-creator-app.md) |

## 3. Templates

### 3.1 `character.json` (schemaVersion 3)

```jsonc
{
  "schemaVersion": 3,
  "charId": "<uuid>",
  "firstName": "Min-ha", "lastName": "Shin",
  "personality": "…one colorful paragraph…",
  "behavior": {
    "withStrangers": "…", "withFriends": "…", "withCrush": "…",
    "withLover": "…", "withEnemy": "…"
  },
  "backstory": "…", "datingHistory": "…", "datingPreference": "…", "kinks": "…",
  "isVirgin": true,
  "likes": ["…"], "dislikes": ["…"],
  "giftPreferences": { "liked": ["romantic", "cozy"], "disliked": ["practical"] },
  "traits": ["Promiscuous"], "preferredStat": "heart",
  "appearance": {
    "hairColor": "brown", "eyeColor": "blue", "hairShade": [], "eyeShade": [],
    "hairLength": "long", "hairTexture": "straight",
    "hairStyle": ["swept_bangs"], "hairBangs": [], "headAccessories": [],
    "makeup": [], "breastSize": ["large"], "skin": []
  },
  "outfit": ["white_crop_top", "denim_shorts", "black_boots"],
  "peOutfit": ["…"], "swimOutfit": ["…"],
  "pose": "hand_on_hip",
  "expressions": { "neutral": ["…"], "happy": ["…"], "sad": ["…"], "angry": ["…"],
                   "surprised": ["…"], "embarrassed": ["…"], "aroused": ["…"] },
  "roomPrompt": "…",
  "voice": "medium",
  "baseAppearance": ["…booru tags…"],
  "generationSeed": 123456789
}
```

### 3.2 A class

```jsonc
{ "code": "COM 315", "name": "Public Relations Campaign Design",
  "description": "A project-based workshop…",
  "kind": "project", "category": "major", "major": "Communications",
  "professor": { "lastName": "Williams", "gender": "man", "age": "older", "temperament": "sarcastic" },
  "difficulty": "medium", "slot": 6 }
```

### 3.3 A job

```jsonc
{ "jobId": "diner", "employer": "Bobby's Diner", "locationId": "restaurant",
  "shifts": ["mon_pm", "wed_am", "fri_pm"], "pay": 90,
  "workplaceKey": "diner", "messages": { "raise": "…", "holiday": "…" } }
```

### 3.4 A location

```jsonc
{ "id": "cutetea", "kind": "interior", "blurb": "a bubble tea shop with borrow-able board games",
  "bg": "cute_tea_day" }
```

### 3.5 An occasion

```jsonc
{ "id": "valentines", "title": "Valentine's Day", "description": "…",
  "startDate": 26, "endDate": 26, "time": null, "cancelsClasses": false, "kind": "holiday" }
```

## 4. Step-by-step (M0 → M1)

1. **Kernel** ([15](./15-reference-kernel.md)): `Registry`, `World`, `EventBus`, `Clock`,
   `Rng`, `OpenAiGateway`, `DefaultSceneEngine`, `LoopStack`, `ScheduleLoop`.
2. **Personas & schemas** ([16](./16-schemas-and-prompts.md)): paste `PERSONA`, `jsonRules`, the
   `scene` and `ledger` schemas.
3. **Scene blocks**: `Protagonist`, `Cast`, `Now`, `Lore`, `SceneSoFar`, `Tail`.
4. **Presenter** ([08](./08-presenter-engine.md)): the stage reducer (`show/hide/sprite/cg/bg`)
   + a dialogue box.
5. **Save** ([17](./17-state-and-save.md)): record + overlay, slot save, autosave.
6. **Loop**: `ScheduleLoop` with `[day, night]`, per-phase resolve → the ledger.
7. **Play a turn**: player action → `classifyAction` → `castTurn` → `scene.run` → `ledger` →
   `reduce`.

Repeat with kits for M2–M8. Nothing here needs new kernel code — it's **content + kit wiring**.

## 5. The one rule

Keep every sub-system in the [decision → resolution → prose](./14-vn-plus-systems.md#3-the-one-design-rule-decision--resolution--prose)
shape. If you do, the VN base never changes and the kits stay pluggable.

---

**Next:** [21 — UI screens](./21-ui-screens.md).
