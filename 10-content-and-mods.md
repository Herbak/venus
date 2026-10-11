# 10 — Content & mods

[← Index](./README.md) · [Prompt system](./09-prompt-system.md) · [Next: Worked example →](./11-worked-example-combat.md)

A game is data + declared engines. This chapter covers **content packs** (the genre-neutral
entity model), the source's **user-content surfaces**, and the **ComfyUI image pipeline** setup.

---

## 1. Genre-neutral entity model

Rename the campus concepts to universal ones; keep the **record + overlay** persistence.

| Campus concept | Generic entity | Fields |
|---|---|---|
| character | **Actor** | stats, traits, tags, voice, schedule, flags, memories, appearance |
| class/job | **Group/Role** | membership, schedule, resources, progression |
| project/exam | **Quest/Challenge** | state, deadline, success criteria |
| location | **Place** | tags, adjacency, spawn tables |
| occasion/holiday | **WorldEvent** | trigger, duration, effects |
| plan | **Appointment** | actors, place, time, initiatedBy |
| memory/event/act | **History entry** | typed, dated, weighted |
| money/HP | **Resource** | scalar + bounds |
| affection | **Relationship** | derived score + flags |

The **affection formula** (`Σ points × recency-weight`) generalizes to trust, fear, reputation,
faction standing.

## 2. Content pack layout

```
pack/
  entities/*.json      # actors, places, groups, quests
  prompts/*.ts         # personas + blocks
  schemas/*.json       # structured-output contracts
  directives/*.ts      # new beat directives
  modifiers/*.ts       # genre/rule modifiers
  loops/*.ts           # new loops (e.g. combat)
  systems/*.ts         # scheduled side-systems
  assets/              # images / audio / workflows
  pack.json            # manifest: id, version, schemaVersion, requires[]
```

A pack **declares** engines and data; it does not run arbitrary code (sandbox rule).

## 3. The source's user-content surfaces

The game already ships a modding-ish surface worth generalizing:

| Surface | Source | Generalize as |
|---|---|---|
| Custom characters | `chars:import/export/duplicate`, character schema | **Actor pack** (portable JSON) |
| Custom outfits | `CUSTOM_OUTFIT_SLOTS` (custom1–5), `writeCustomOutfit`, `offeredCustomOutfits` | **Actor outfit slots** (named + trigger instructions) |
| Custom CGs | `CUSTOM_CG_SLOTS` (customcg1–4), `offeredCustomCgs` | **Actor CG slots** |
| Custom backgrounds | `backgrounds:list/remove`, `CustomBackgroundsModal` | **Place assets** |
| Custom rooms | `roomReady`, `assertRoomPicture`, `cropToPng` | **Place assets** |
| Created scenes (sandbox) | `useSceneCreatorStore`, `createdSceneFields`, `writeCreatedScene` | **Scenario editor** |
| Grab bags | `useGrabBagStore`, `draw(bag, pool)` | **Random pools** (names, handles, seed words, feed) |
| Import/export | `backup:export/import`, `chars:export/import` | **Save/pack transfer** |
| Scenes library | `scenes:list/read/write/delete` | **Saved scenarios** |

## 4. The image pipeline (ComfyUI)

Characters are generated once, then **derived** from a seed-locked base.

```
buildCharacterPrompt (danbooru tags) ─► character.json
      ▼
characterBase.json        text2img + ControlNet(openpose skeleton) + upscale + FaceDetailer + rembg
characterCutout.json      base → background cut → alpha channel
characterExpression.json  base → face pass per emotion
characterHands.json       hands repaint/detailer + rembg compositing
characterCg.json          full-screen CGs per stock position (+ "_after")
```

- **Poses:** `pose.json` maps a pose key → booru tags + description; `pose/skeletons/*.png` are
  the OpenPose ControlNet images.
- **Nodes used:** `CheckpointLoaderSimple`, `LoraLoader`, `CLIPTextEncode`, `KSampler`,
  `VAEDecode`, `ControlNetApplyAdvanced`, `ControlNetLoader`, `SeedNode`, `EmptyLatentImage`,
  `ImageUpscaleWithModel`, `UpscaleModelLoader`, `ImageScaleBy`, `FaceDetailer`, `DetailerForEach`,
  `UltralyticsDetectorProvider`, `InspyrenetRembgAdvanced`, `LoadImage`, `SaveImage`,
  `MaskComposite`, `InvertMask`, `JoinImageWithAlpha`, `BboxDetectorSEGS`, `ImpactDilateMask`,
  `ImpactSegsAndMask`, `SegsToCombinedMask`, `GrowMask`, `MaskToSEGS`, `ThresholdMask`,
  `ImageCompositeMasked`, `MaskToImage`.
- **Models installed by setup:** `novaAnimeXL_ilV190` checkpoint, `usnrStyle` LoRA,
  `illustriousOpenpose` ControlNet, `hand_yolov9c` detector, plus ComfyUI custom nodes (Impact
  Pack, Impact Subpack, Inspyrenet Rembg).

**Generalize:** the media gateway exposes `image(spec)`; a workflow is a **template** with
named node slots (`saveImage`, `faceSaveImage`, `detailerPositivePrompt`...). Providers: ComfyUI
(local), an images API (remote), or any plugin.

## 5. Audio content

Declarative track table (see [Presenter](./08-presenter-engine.md#6-audio)): `music`, `ambience`,
`nsfw`, `sfx` groups, each track `{ path, group, loop, trimDb, jitter }`. The source's music
files are external free downloads (documented in `assets/sound/README.md`).

## 6. Source map

| Concept | Source symbol |
|---|---|
| Assets | `getAssetsPath`, `assets/characters`, `assets/pose`, `assets/sound`, `assets/workflows` |
| Character I/O | `chars:export/import`, `MANIFEST_FORMAT`, `buildManifest` |
| Custom content | `CUSTOM_OUTFIT_SLOTS`, `CUSTOM_CG_SLOTS`, `backgrounds:*`, `assertRoomPicture` |
| Sandbox | `useSceneCreatorStore`, `writeCreatedScene`, `scenes:*` |
| Random pools | `useGrabBagStore`, `SEED_WORDS`, `FEED_HANDLE_POOL` |
| Image jobs | `runGenerationJob`, `generateSprite`, `generateCg`, `fixHands`, `BASE_NODE`... |
| Setup | `useSetupStore`, `getSetupStatus`, `PINNED_NODES`, model list |

---

**Next:** [11 — Worked example: combat](./11-worked-example-combat.md).
