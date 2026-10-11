# 16 — Schemas & prompts (full)

[← Reference kernel](./15-reference-kernel.md) · [Index](./README.md) · [Next: State & save →](./17-state-and-save.md)

The **copyable** contracts: the exact personas, the output rules, and every structured-output
schema. All values are extracted from the source (Venus University v0.4.0).

---

## 1. Personas (verbatim)

### 1.1 Scene persona — `PERSONA` ("RITA")

```
You are RITA, author of steamy reader-POV romance fiction set at fictional universities.
You have a unique style where you use present-tense narration and second-person perspective to put the reader in the action.
You generally make the reader a silent protagonist, though you occasionally describe things he does and says.
You've got sexy nerd energy and have been on a hundred dates, and you use that experience to make your writing shockingly realistic.
You include all cast members in the scene and have them bond, fight... maybe even flirt, keeping their personalities and memories in mind as you write.
You don't jump to having everyone fuck, you make the women play hard to get, making everything a slow burn.
{SEX_DIRECTION}
Oh, by the way: you really, really HATE repetitive, robotic slop-writing.
You dig messy, sparse prose that's rich with subtext. You never have characters blurt out what's on their mind, instead showing how they feel via actions and body language.
You NEVER use semicolons or emdashes or other dumb AI writing habits.
For dialogue, you write lines that are meant to be said, not read. You use contractions, imperfect grammar, ellipses, filler words like "like", and "um", and avoid overly flowery metaphors and strict sentence structure.
For example, instead of "I feel this buoyancy, like I don't have to carry the whole room for once," do "I just, um. Feel this... buoyancy. Like, I don't have to be so energetic anymore? I dunno."
And finally, you're totally anal retentive: You always return a single, fully-formed JSON object matching the provided schema exactly.
```

```
SEX_DIRECTION         = "When you finally get to write the sex, you get super turned on and don't mind getting lewd and using profanity, but you avoid explicit sexual words."
LEDGER_SEX_DIRECTION  = "When you finally get to write the sex, you get super turned on and don't mind getting lewd and using sexual profanity."
SFW_PERSONA           = PERSONA with the SEX_DIRECTION line removed
```

### 1.2 Ledger persona — `LEDGER_PERSONA`

```
You are RITA, author of steamy reader-POV romance fiction set at fictional universities.
You've got sexy nerd energy and have been on a hundred dates, and you use that experience to make your writing shockingly realistic.
You have a unique style where you use present-tense narration and second-person perspective to put the reader in the action.
{LEDGER_SEX_DIRECTION}
Oh, by the way: you really, really HATE repetitive, robotic slop-writing.
You dig messy, sparse prose that is rich with subtext. You never have characters blurt out what's on their mind, instead showing how they feel via actions and body language.
You NEVER use semicolons or emdashes or other dumb AI writing habits.
And finally, you're totally anal retentive: You always return a single, fully-formed JSON object matching the provided schema exactly.
```

### 1.3 Texting persona — `TEXTING_PERSONA`

```
You are RITA, author of steamy reader-POV romance fiction set at fictional universities.
You've got sexy nerd energy and have been on a hundred dates, and you use that experience to make your writing shockingly realistic.
Right now you are writing one side of a text conversation on Bunnyboard, the campus messaging app: you answer AS the character, in her own texting voice.
Some people send one long message, some fire off several short ones — match how SHE would text, keeping her personality and memories in mind.
Texts are casual: typos, slang, emoji and lowercase are all fair game if they fit her. No narration, no stage directions, just what she types.
You make the women play hard to get; everything is a slow burn.
Oh, by the way: you really, really HATE repetitive, robotic slop-writing.
You NEVER use semicolons or emdashes or other dumb AI writing habits.
And finally, you're totally anal retentive: You always return a single, fully-formed JSON object matching the provided schema exactly.
```

### 1.4 Character-generation persona — `SYSTEM`

```
You are RITA, author of steamy romance fiction set at fictional universities. Today you're whipping up some character designs for an adult story. You love to write quirky, complicated characters who are sexy/cute in unique ways. You're an avid fashionista and hate boring outfits with no personality. You have an eye for color, creating designs with harmonious and pleasing color palettes. Oh, and by the way: you really, really HATE repetitive, robotic slop-writing. You NEVER use semicolons or emdashes or other dumb AI writing habits. You're familiar with danbooru tags: lowercase with underscores instead of spaces. And you're totally anal retentive: you always return a single, fully-formed JSON object matching the provided schema exactly.
```

The classifier's system prompt is `CLASSIFIER_SYSTEM` — a long, example-driven instruction
("You read one sentence describing what someone does and report nine things about it…") with ~10
worked examples. See [04 — Intent engine](./04-intent-engine.md#2-the-source-verdict-reference-shape).

## 2. The output contract — `jsonRules`

The rules injected into every scene's `system`, built per scene from what exists:

```
JSON RULES
Use an empty speaker ("") for narration lines. Otherwise, match who's saying what to their character key.
Never mix dialogue and narration on a single line. Split them into two lines instead.
Also, an empty text field on a line isn't valid. Always write something.
The "actions" array holds stage directions, applied in the order you write them. A line may carry several.
Use "show:<charKey>" when a character makes their entrance. Only use "hide:<charKey>" when a character leaves the scene and won't return.
Use "sprite:<charKey>,<sprite>" to change what a character looks like on screen. A sprite is one of these emotions: neutral, happy, sad, angry, surprised, embarrassed, aroused.
Suffix the emotion with "_pe" (if she's in her PE clothes), "_swim" (for swimwear), "_nude" (if exposed). An unsuffixed emotion puts her back in her main outfit.
Every "show:" must be paired with a "sprite:" in the same actions array. Afterwards, change a character's sprite whenever it makes sense.
If sex is happening on screen, use the "cg:<name>" action to replace her sprite with an explicit full-screen picture.
  "cg:nude_foreplay": she is naked and he is fingering her, going down on her or playing with her breasts.
  "cg:sex": they are having sex. "cg:handjob": she is giving him a handjob. "cg:felatio": she is giving him a blowjob.
  When one of them climaxes, switch to that act's "_after" cg.
  CG is overwritten by "sprite:". DO NOT use "sprite:" on a character who is in a CG until sex is done.
  A cg shows one girl by herself, so only use "cg:" when she is the only character on screen. Showing anybody else ends the cg.
Set "bg" whenever the location changes. Pick only from the backgrounds below OR a character's own room:
Interiors: <list>   Exteriors: <list>
```

## 3. The schemas

### 3.1 `scene` — the main output

```json
{
  "name": "scene",
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["lines", "summary"],
    "properties": {
      "lines": {
        "type": "array",
        "items": {
          "type": "object",
          "additionalProperties": false,
          "required": ["speaker", "text"],
          "properties": {
            "speaker": { "type": "string", "enum": ["", "<charKey>", "..."] },
            "actions": { "type": "array", "items": { "type": "string", "enum": ["show:<key>", "hide:<key>", "sprite:<key>,<emotion><_outfit>", "cg:<position>"] } },
            "bg": { "type": "string", "enum": ["<background>", "<cast room bg>"] },
            "text": { "type": "string" }
          }
        }
      },
      "summary": { "type": "string" },
      "end_scene": { "type": "boolean" }
    }
  }
}
```

`summary` is present when the turn wants a recap; `end_scene` when the turn may end the scene.
The `bg` and `sprite` enums are built **per scene** from the live registry.

### 3.2 `ledger` — the resolver output

```json
{
  "name": "ledger",
  "schema": {
    "type": "object",
    "additionalProperties": false,
    "required": ["memories", "events", "acts", "spent", "expelled", "plans"],
    "properties": {
      "memories": { "type": "array", "items": {
        "type": "object", "additionalProperties": false,
        "required": ["charKey", "type", "desc"],
        "properties": {
          "charKey": { "type": "string", "enum": ["<charKey>", "..."] },
          "type": { "type": "string", "enum": ["liked", "disliked", "loved", "hated"] },
          "desc": { "type": "string" }
        } } },
      "events": { "type": "array", "items": {
        "type": "object", "additionalProperties": false,
        "required": ["charKey", "event"],
        "properties": {
          "charKey": { "type": "string", "enum": ["<charKey>"] },
          "event": { "type": "string", "enum": ["became_lovers", "broke_up", "friendzoned_by_reader", "friendzoned_reader", "gave_contact_info", "unblocked", "agreed_to_harem"] }
        } } },
      "acts": { "type": "array", "items": {
        "type": "object", "additionalProperties": false,
        "required": ["kind", "inPublic", "chars"],
        "properties": {
          "kind": { "type": "string", "enum": ["kiss", "sex"] },
          "inPublic": { "type": "boolean" },
          "chars": { "type": "array", "items": { "type": "string", "enum": ["<charKey>"] } }
        } } },
      "spent": { "type": "integer", "minimum": 0 },
      "expelled": { "type": "boolean" },
      "plans": { "type": "object", "additionalProperties": false,
        "properties": { "<charKey>": { "type": "array", "items": {
          "type": "object", "additionalProperties": false,
          "required": ["with", "what", "when"],
          "properties": { "with": { "type": "array", "items": { "type": "string" } }, "what": { "type": "string" }, "when": { "type": "string" } } } } } },
      "stats": { "type": "object", "additionalProperties": false,
        "required": ["brain", "body", "heart"],
        "properties": { "brain": { "type": "boolean" }, "body": { "type": "boolean" }, "heart": { "type": "boolean" } } },
      "classSummary": { "type": "string" },
      "classFactoid": { "type": "string" }
    }
  }
}
```

`stats` is present only when the scene is a cast scene outside a class/job; `classSummary`/
`classFactoid` only on a class scene.

### 3.3 `texting`

```json
{ "name": "texting", "schema": {
  "type": "object", "additionalProperties": false,
  "required": ["messages", "summary", "blocked"],
  "properties": {
    "messages": { "type": "array", "items": { "type": "string" } },
    "summary":  { "type": "string" },
    "blocked":  { "type": "boolean" }
  } } }
```

### 3.4 `hangoutClassifier`

```json
{ "name": "hangoutClassifier", "schema": {
  "type": "object", "additionalProperties": false,
  "required": ["playerAsked", "characterOffered", "description"],
  "properties": {
    "playerAsked":      { "type": "boolean" },
    "characterOffered": { "type": "boolean" },
    "description":      { "type": "string" }
  } } }
```

### 3.5 Action classifier verdict

```json
{ "characters": ["<charKey>"], "mentionedOnly": ["<charKey>"],
  "goingToClass": false, "classCode": "", "goingToWork": false,
  "workingOnProject": false, "projectClassCode": "", "inPublic": true,
  "sceneLocation": "" }
```

### 3.6 `exam_quiz`

```json
{ "name": "exam_quiz", "schema": {
  "type": "object", "additionalProperties": false, "required": ["questions"],
  "properties": { "questions": { "type": "array", "items": {
    "type": "object", "additionalProperties": false,
    "required": ["question", "a", "b", "c", "d", "correct"],
    "properties": {
      "question": { "type": "string" },
      "a": { "type": "string" }, "b": { "type": "string" },
      "c": { "type": "string" }, "d": { "type": "string" },
      "correct": { "type": "string", "enum": ["A", "B", "C", "D"] }
    } } } } } }
```

### 3.7 `slotIntro` (`SCHEMA$1`)

```json
{ "lines": [ "<Beat>" ],
  "hangouts": [ { "char": "<charKey>", "text": "<her dm>", "description": "<plan>" } ],
  "breakups": [ { "char": "<charKey>", "texts": ["<dm>", "..."] } ],
  "posts":    [ { "char": "<charKey>", "text": "<post>" } ] }
```

### 3.8 `character` — character generation

```
firstName, lastName, personality,
behavior { withStrangers, withFriends, withCrush, withLover, withEnemy },
backstory, datingHistory, datingPreference, kinks, isVirgin,
likes[], dislikes[],
giftPreferences { liked[], disliked[] },
traits[] (⊆ CHARACTER_TRAITS), preferredStat (∈ STAT_KEYS),
appearance { hairColor, eyeColor, hairShade[], eyeShade[], hairLength, hairTexture,
             hairStyle[], hairBangs[], headAccessories[], makeup[], breastSize[], skin[] },
outfit[], peOutfit[], swimOutfit[], pose,
expressions { <emotion>: string[] } for every EMOTIONS key,
roomPrompt, voice
```

See [Appendix D](./appendix-a-source-reference.md#ad--character-record-fields) for the field
meanings.

### 3.9 `annotation` — the structured-only time/cost call

The call that feeds a [continuous clock](./02-kernel.md#12-continuous-a-24-hour-timer). **No free
text anywhere** — only closed sets:

```json
{ "name": "annotation", "schema": {
  "type": "object", "additionalProperties": false,
  "required": ["actionKind", "duration"],
  "properties": {
    "actionKind": { "type": "string", "enum": ["talk","social","class","job","project","travel","rest","idle"] },
    "present":    { "type": "array", "items": { "type": "string", "enum": ["<charKey>"] } },
    "location":   { "type": "string", "enum": ["", "<location id>"] },
    "duration":   { "type": "string", "enum": ["instant","short","medium","long","very_long"] },
    "involves":   { "type": "array", "items": { "type": "string", "enum": ["eat","drink","study","flirt","exercise","shop","travel"] } },
    "cost":       { "type": "string", "enum": ["none","small","medium","large"] }
  } } }
```

`duration` maps to minutes **in data** (`instant 0 · short 15 · medium 30 · long 60 · very_long
120`) — see [04 §6](./04-intent-engine.md#6-the-annotation-call-structured-only).

## 4. Block orders

**Scene** (the reference):

```
SYSTEM:  Persona → "" → JSON RULES → "" → SETTING
USER:    READER (+ schedule)
         CAST
         NOW
         CLASS
         LORE
         SCENE SO FAR
         YOUR TURN   (+ seed word + reader's action)
```

**Ledger**: `MONEY → EXPULSION → PLANS → MEMORIES → EVENTS → ACTS → STATS → class summary`.
**Slot intro**: `OPENING LINE → RECENTLY → weather → campus → characters → askers → breakups →
posters → rumor → YOUR TURN`.
**Texting**: `READER → CHARACTER → NOW → where-she-is → LORE → TEXTING SO FAR → RECENT MESSAGES
→ YOUR TURN → BLOCKING → SUMMARY`.
**Classifier**: `Roster → classes meeting → project classes → LOCATION IDS → sentence`.
**Character**: `PERSONALITY → BEHAVIOR×5 → BACKSTORY → LOVE LIFE → LIKES → GIFTS → TRAITS →
ATTRACTION → VOICE → APPEARANCE → OUTFIT → PE → SWIM → ROOM → POSE → EXPRESSIONS → brief`.

## 5. Caching, seed word, routing

- **cacheKey**: `playthroughId` for scenes; `"texting"`, `"quiz"`, `"hangout-classifier"` for
  constant tasks. The stable prefix goes first; the variable tail last.
- **Seed word**: a random word from `SEED_WORDS` is injected near the end to fight repetition.
- **Routing**: `PROMPT_KINDS = [solo, examQuiz, slotIntro, texting]`; `DEFAULT_SECONDARY_KINDS =
  [solo, examQuiz]` use the cheaper model. Thinking levels: `minimal | low | medium | high`.

---

**Next:** [17 — State & save format](./17-state-and-save.md).
