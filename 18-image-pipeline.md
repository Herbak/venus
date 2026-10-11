# 18 — Image pipeline (ComfyUI)

[← State & save](./17-state-and-save.md) · [Index](./README.md) · [Next: Kits →](./19-kits.md)

The character art pipeline: a **seed-locked base** with everything derived from it. Extracted
from the source's `assets/workflows/*.json` and `generateSprite` / `generateCg` / `fixHands`.

---

## 1. The chain

```
buildCharacterPrompt (danbooru tags) ─► character.json
      │  appearance, main/PE/swim outfit, personality, 5 behavior stages,
      │  love life, traits, expression tags, pose, room prompt
      ▼
characterBase.json        text2img + ControlNet(openpose skeleton) + upscale + FaceDetailer + rembg → base sprite
characterCutout.json      base → background cut → alpha channel
characterExpression.json  base → face pass per emotion (neutral, happy, sad, angry, surprised, embarrassed, aroused)
characterHands.json       hands repaint/detailer + rembg compositing
characterCg.json          full-screen CGs per stock position (+ "_after")
```

The **neutral base** is generated once per outfit set, then every expression/outfit/hand pass
**reuses it** (img2img + a face/hand detailer) so the girl stays consistent.

## 2. Workflows as templates

Each workflow JSON is a graph of nodes; the engine patches **named node slots** with prompts,
seed, and image names:

```ts
interface WorkflowSpec {
  workflowFile: string
  nodes: Record<string, string>     // slot name -> node id
  prompts?: { positive: string; negative: string }
  detailerPositive?: string
  seed?: number
  sourceImageName?: string          // for the derived passes
  poseImageName?: string            // the openpose ControlNet input
  paintImageName?: string           // for the hands pass
  destPath: string
  faceDestPath?: string
  regionDestPath?: string
}
```

The named slots per workflow (from the source):

| Workflow | Slots |
|---|---|
| `characterBase` | `saveImage`, `faceSaveImage`, `detailerPositivePrompt` |
| `characterCutout` | `saveImage` |
| `characterExpression` | `saveImage`, `faceSaveImage`, `detailerPositivePrompt` |
| `characterHands` | `saveImage`, `regionSaveImage`, `detailerPositivePrompt` |
| `characterCg` | `saveImage`, `faceSaveImage` |

The engine loads the JSON, patches `inputs` on those nodes, posts it to `${BASE_URL}/prompt`,
and reads the result from the websocket / output folder.

## 3. Nodes used

```
CheckpointLoaderSimple, LoraLoader, CLIPTextEncode, KSampler, VAEDecode, VAEEncode,
ControlNetApplyAdvanced, ControlNetLoader, SeedNode, EmptyLatentImage,
ImageUpscaleWithModel, UpscaleModelLoader, ImageScaleBy,
FaceDetailer, DetailerForEach, UltralyticsDetectorProvider, InspyrenetRembgAdvanced,
LoadImage, SaveImage, MaskComposite, InvertMask, JoinImageWithAlpha,
BboxDetectorSEGS, ImpactDilateMask, ImpactSegsAndMask, SegsToCombinedMask, GrowMask,
MaskToSEGS, ThresholdMask, ImageCompositeMasked, MaskToImage
```

## 4. Models & custom nodes (installed by setup)

| Component | Item |
|---|---|
| Checkpoint | `novaAnimeXL_ilV190.safetensors` |
| LoRA | `usnrStyle.safetensors` |
| ControlNet | `illustriousOpenpose` (`openpose_s6000.safetensors`) |
| Detector | `hand_yolov9c.pt` (Ultralytics) |
| Nodes | ComfyUI Impact Pack (FaceDetailer), Impact Subpack (Ultralytics), Inspyrenet Rembg |

`getSetupStatus` / `runInstall` / `verifyModel` download, verify (hash), and patch these.

## 5. Poses & skeletons

- `pose/pose.json`: `{ <poseKey>: { tags: [...], description: "..." } }` — a pose maps to booru
  tags + a human description (used both as a ControlNet hint and in the character prompt).
- `pose/skeletons/<key>.png`: the OpenPose ControlNet images.
- `pose/poseMaterial/`: supporting material.
- The chosen `pose` rides in `character.json`; `stagePose` selects the skeleton per generation.

## 6. Seeds & staging

- `character.generationSeed` is fixed at creation; every pass reuses it (or an override) so the
  same girl renders consistently.
- **Staging:** a new character generates to a `staging/` path, then is promoted to live once all
  passes succeed (`getStagedPath` vs the live path). The manage-characters UI shows per-emotion
  state (`unwritten` → `unfinished` → `ready`) and a "resume" for partial generations.

## 7. Prompt builders

| Builder | Produces |
|---|---|
| `buildBasePrompt` | the neutral base (appearance + main outfit + pose tags) |
| `buildOutfitBasePrompt` | a base in a given outfit set (PE/swim/custom) |
| `buildImagePrompt` | a per-emotion face pass (base + expression tags) |
| `buildCgPrompt` | a full-screen CG for a stock position (+ a detailer prompt) |
| `photoPrompt` | a photo |

Prompts are assembled from the character's tags (`baseAppearance`, outfit tags, `expressions`).

## 8. Generalization

- The media gateway exposes `image(spec)`; a **workflow is a template with named node slots**.
- Providers: ComfyUI (local), an images API (remote), or a plugin.
- The **base → derived** pattern is the reusable idea: generate one canonical asset, then derive
  variants (expressions, outfits, states) from it for consistency.

---

**Next:** [19 — Kits](./19-kits.md).
