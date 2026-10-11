# Appendix A — Source reference

[← Index](./README.md) · [Roadmap](./12-roadmap-and-rules.md)

Concrete values, schemas, and surfaces extracted from **Venus University v0.4.0**, kept here so
the design docs can stay conceptual.

---

## A.1 Enums

```
EMOTIONS         = neutral, happy, sad, angry, surprised, embarrassed, aroused
STOCK_POSITIONS  = nude_foreplay, nude_foreplay_after, sex, sex_after,
                   handjob, handjob_after, fellatio, fellatio_after
CUSTOM_CG_SLOTS  = customcg1..4
CUSTOM_OUTFIT_SLOTS = custom1..5
STOCK_OUTFIT_SETS   = pe, swim, nude
MEMORY_TYPES     = liked, disliked, loved, hated
ACT_KINDS        = kiss, sex
EVENT_GLOSS      = became_lovers, broke_up, friendzoned_by_reader,
                   friendzoned_reader, gave_contact_info, unblocked, agreed_to_harem
STAT_KEYS        = brain, body, heart
CHARACTER_TRAITS = Mood-swings, Disagreeable, Good-natured, Materialist,
                   Terminally Online, Promiscuous, High Standards
PROMPT_KINDS     = solo, examQuiz, slotIntro, texting
DEFAULT_SECONDARY_KINDS = solo, examQuiz
THINKING_LEVELS  = minimal, low, medium, high
TEMPERAMENTS     = strict, easygoing, old-fashioned, passionate, sarcastic,
                   absent-minded, gruff, warm, theatrical, dry, eccentric, funny
DIFFICULTIES     = easy, medium, hard
```

## A.2 The scene output contract (`jsonRules`)

```
beats[]: { speaker: "" | <actorKey>, text, bg?: <bg enum>, directives?: <enum>[] }
directives: show:<key> | hide:<key> | sprite:<key>,<emotion><_outfit> | cg:<position>
+ summary?: string   + end?: boolean
```

`bg` and `sprite` enums are built per scene (backgrounds + cast room bgs; emotions + outfit
suffixes the actor actually has).

## A.3 Schemas

### ledger (abridged)
```
ledger: {
  memories: [{ charKey, type ∈ MEMORY_TYPES, desc }],
  events:   [{ charKey, event ∈ EVENT_GLOSS }],
  acts:     [{ kind ∈ ACT_KINDS, inPublic, chars[] }],
  spent:    integer ≥ 0,
  expelled: boolean,
  plans?, stats? { brain, body, heart : boolean },
  classSummary?, classFactoid?
}
```

### texting
```
texting: { messages: string[], summary: string, blocked: boolean }
```

### hangout classifier
```
hangoutClassifier: { playerAsked: boolean, characterOffered: boolean, description: string }
```

### action classifier verdict
```
{ characters[], mentionedOnly[], goingToClass, classCode,
  goingToWork, workingOnProject, projectClassCode, inPublic, sceneLocation }
```

### exam
```
exam_quiz: { questions: [{ question, a, b, c, d, correct ∈ [A,B,C,D] }] }
```

### slot intro (`SCHEMA$1`)
```
{ lines[], hangouts[], breakups[], posts[] }
```

## A.4 Affection & stats

```
affection = Σ memories  POINTS[type] × weightFor(daysAgo)  + likedPostBonus
POINTS    = { loved:+2, liked:+1, disliked:-1, hated:-2 }
weightFor = ≤7d:×4, ≤30d:×2, else ×1
tiers     = friendly 15, trusted 30, devoted 50 (symmetric negatives)
dispositions = devoted, trusted, friendly, neutral, annoyed, hostile

stats: brain/body/heart, tiers 0..5, STARTING_TIER 1
AWARD 1 · CAST_STATS 1 · SOLO_BONUS 1
```

**Flags:** `hasMet, hasCrush, friendZoned, friendZonedBy, isLover, brokenUp(count), hasKissed,
hadSex, benefits, harem, gaveContactInfo, blocked, knowsTraits, knowsBackstory, knowsLoveLife`.

## A.5 Main-process IPC surface

```
settings:get/set
grabBags:get/set
app:quit
update:check/apply
log:write/export
setup:getStatus/runInstall/openModelFolder/verifyModel
assets:getPoseManifest/getQuickstart/readAudio
saves:playthroughs/list/enroll/enrollment/readEndingArt/exportEndingArt (…)
replays:list
photos:options/list/export (…)
comfy:start/stop/… (queue, ws)
jobs:cancelGroup/cancelKeys
llm:completeScene / classify / classifyHangout / generateQuiz /
     listModels / testWriter / listImageModels / testImages
chars:list/create/reference/update/delete/expressions/cgs/outfits/profileCrop/
      export/import/duplicate/saveExport/defaults/restoreDefaults/openFolder/room
backgrounds:list/remove
scenes:list/read/write/delete
backup:export/import
```

Renderer receives streamed scene deltas via `llm.onSceneDelta`.

## A.6 Renderer stores

```
useGameStore          the world (see A.7)
useUiStore            errors, modals, theme
useSettingsStore      settings + provider config
useSaveStore          playthroughs/saves
useAudioStore         playback, volumes, pitch
useAssetStore         backgrounds, poses
useCharacterStore     character CRUD + custom outfits/cgs
useBunnyboardStore    texting UI state
useJobStore           ComfyUI/LLM job progress
useComfyStore         ComfyUI status
usePhotoStore         photos
useSceneCreatorStore  created-scene sandbox
useGrabBagStore       random pools
useSetupStore         ComfyUI install/verify
useCrossingStore      transitions
useUpdateStore        app updates
```

Plus a module-level `loopState` scratch object (not persisted).

## A.7 `useGameStore` shape (abridged)

```
playthroughId, date, time, chars, playerFirstName, playerLastName,
stats { brain, body, heart }, money, bio, tallies,
charInfo {}, classes {}, playerSchedule {}, history {}, replays {},
bunnyboard {}, events [], occasions [], occasionsDeclined [], job, jobsClosed,
jobClosures {}, weather [], inventory [], classRecords {}, gradesStanding,
expelled, npcRelationships {}, npcFriendships [], npcOverlay, slotRumor,
lastSlotCast, weekendOutings {}, outingSlots {}, springBreakAway, feedExtras,
graduationSeen, farewellsDone [], characters {}, charKeyToId {},
cast [], scene* fields..., currentSceneTranscript [], sceneSummary,
sceneSummaries [], sceneEnding, endingInFlight, awaitingInput, busy, streaming,
waitingForLine, ...error fields, pendingLines [], currentLine,
cgReady {}, outfitReady {}, customCgReady {}, roomReady {},
...stage { bg, slots, emotions, flipped, departed, offStage, sceneLog,
           stageOverride, bgOverride, outfitLock, cgLock, brightened }
```

## A.8 ComfyUI workflows & nodes

Workflows: `characterBase.json`, `characterCutout.json`, `characterExpression.json`,
`characterHands.json`, `characterCg.json`.

Nodes seen: `CheckpointLoaderSimple, LoraLoader, CLIPTextEncode, KSampler, VAEDecode,
ControlNetApplyAdvanced, ControlNetLoader, SeedNode, EmptyLatentImage, ImageUpscaleWithModel,
UpscaleModelLoader, ImageScaleBy, FaceDetailer, DetailerForEach, UltralyticsDetectorProvider,
InspyrenetRembgAdvanced, LoadImage, SaveImage, MaskComposite, InvertMask, JoinImageWithAlpha,
BboxDetectorSEGS, ImpactDilateMask, ImpactSegsAndMask, SegsToCombinedMask, GrowMask, MaskToSEGS,
ThresholdMask, ImageCompositeMasked, MaskToImage`.

Models (installed by setup): `novaAnimeXL_ilV190` checkpoint, `usnrStyle` LoRA,
`illustriousOpenpose` ControlNet, `hand_yolov9c` detector; custom nodes: Impact Pack, Impact
Subpack, Inspyrenet Rembg.

## A.9 Character record fields (`character.json`, schemaVersion 3)

```
charId, firstName, lastName, personality,
behavior { withStrangers, withFriends, withCrush, withLover, withEnemy },
backstory, datingHistory, datingPreference, kinks, isVirgin,
likes[], dislikes[],
giftPreferences { liked[], disliked[] },
traits[] (⊆ CHARACTER_TRAITS), preferredStat (∈ STAT_KEYS),
appearance { hairColor, eyeColor, hairShade[], eyeShade[], hairLength, hairTexture,
             hairStyle[], hairBangs[], headAccessories[], makeup[], breastSize[], skin[] },
outfit[], peOutfit[], swimOutfit[], pose,
expressions { <emotion>: string[] } for every EMOTIONS key,
roomPrompt, voice, baseAppearance[], generationSeed
```

## A.10 Prompt roster (source builders)

`buildScenePrompt`, `buildSoloPrompt`, `buildContinuationPrompt`, `buildClosingPrompt`,
`buildSlotIntroPrompt`, `buildLedgerPrompt`, `buildTextingPrompt`,
`buildHangoutClassifierPrompt`, `buildClassifierPrompt`, `buildQuizPrompt`,
`buildCharacterPrompt`, `buildClassPrompt`, `buildProfilePrompt`, `buildOccasionPrompt`,
`buildEndingPostsPrompt`, `photoPrompt`. Personas: `PERSONA`, `LEDGER_PERSONA`,
`TEXTING_PERSONA`, `SFW_PERSONA`.

## A.11 Scene playback & interjection

```
playback        advance, advanceLine, isSilentLine, deliverSceneLines, writeSceneLines,
                queueLines, owedFloor (REPLY_FLOOR_MS = 3000)
interject offer interjectOfferOf, lockOf, replyRowOfferOf, playbackOffered
interject run   interject, takeTurnFromReply, truncateUnread, reachDecisionPoint,
                abortSceneCall, dropEnding
divider words   DIVIDER_WORDS = { hover: "HOVER TO SHOW ACTION BOX",
                                  locked: "CLASS IN SESSION",
                                  ending: "SCENE IS ENDING" }
lock rule       lockOf = sceneClass !== null && acted <= 1 ? "locked" : "open"
unread policy   truncateUnread() drops pendingLines and slices the transcript to the read point
ending guard    InterruptEndingModal ("Interrupt the ending?") when sceneEnding & warnEndingInterrupt;
                runEnding fires closing + ledger + text-ledger in PARALLEL (the last two are the
                "bookkeeping"); interrupting -> dropEnding() cancels ENDING_LLM_GROUP and clears
                pendingLedgerResult, discarding the already-sent bookkeeping results
edit guard      EditEndingModal ("Edit ending scene?") when editing an ending whose bookkeeping is
                already in flight & warnEndingEdit ("changes won't be reflected unless you insert
                another action")
```

---

[← Back to index](./README.md)
