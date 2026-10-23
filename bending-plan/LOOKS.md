# Bending — Looks

Every look is **drawn for the mod**: a 64×64 skin plus the mod's own `ModelPart` layers (hair, robes, armour plates,
crowns, hoods), the way Enhanced draws Agatha's hat, Heimdall's helm or Hela's headdress (`AgathaHatLayer`-style
`RenderLayer<AbstractClientPlayer, PlayerModel>` classes, a `LayerDefinition` in head-pixel space, attached per skin
model through `ClientRegistrar.addLayers`). No skin is copied from a skin site; the references below are **descriptions
from the shows** for the artist (the art lanes draw from them with the same `gen_<lane>.py`-style pixel scripts Enhanced
used). Nothing here changes rules: a look changes how you look and the colour of your element's effects (Azula). Every
element also has **Your own skin** (`own`): your skin, the element's effects.

**Status of the research:** the ATLA and Korra characters are well documented and the descriptions below are from the
shows. *Avatar: Seven Havens* (Paramount+, first episodes released October 2025) is new: only text sources were
reachable from this session, so **Jae's and Kartik's entries list exactly what is known and what needs a picture from
Eilon** (section 2.5, 2.6). Eilon's spellings ג'יי and קארתיק are kept in Hebrew; in English the show spells them **Jae**
and **Karthik** (the plan uses those in ids: `jae`, `karthik`; the Hebrew display names are Eilon's).

## 0. The skin layout plan (every look)

The base is the vanilla **64×64** skin (wide or slim arms per look). The mod's looks use the overlay regions for what
sits close to the body and separate layers for anything that sticks out more than the 0.25-pixel hat layer allows.

| Region (vanilla UV) | Base layer | Overlay layer (second skin) | Used for |
|---|---|---|---|
| Head `(0,0)–(32,16)` | face, skin tone, eyes, scars, tattoos (Air arrows on the brow), hairline | `(32,0)` hat: short hair volume, headbands, bangs, beards close to the face, masks (Amon), hoods' inner rim | all |
| Body `(16,16)–(40,32)` | tunic, sash, belt, necklace (Katara), chest emblem (Jae) | `(16,32)` jacket: open vests (Bolin's collar), robe folds, shoulder wraps (Tenzin's cape at the shoulders), armour plates close to the body (Lin) | all |
| Arms `(40,16)` / `(32,48)` | sleeves, bracers, wristbands, bare skin (Toph, Ghazan) | `(40,32)` / `(48,48)`: puffed sleeves, bracers that stand off (Azula's cuffs), the Air Nomad arm arrows | all |
| Legs `(0,16)` / `(16,48)` | trousers, boots, **bare feet (Toph)** | `(0,32)` / `(0,48)`: boot tops, leg armour (Jae's cuisses), robe hems, knee wraps | all |

**Layers (`ModelPart`, drawn by the mod):** each is a small `LayerDefinition` in head space (top of the head y = −8,
front z = −4) or body space, textured on its own PNG (64×32 or 64×64), attached to `head` / `body` / legs with
`translateAndRotate`, hidden when the player wears a helmet in that slot (hair/crowns) or is invisible, exactly as
`AgathaHatLayer.wears` does. Shared layer kinds, drawn once and textured per look:

| Layer kind | Attached to | Geometry | Looks that use it (texture each) |
|---|---|---|---|
| `TopknotLayer` | head | a 3×3×3 bun 1.5 px above the crown + an optional 1×4×1 pin and a 2×2×2 tie; variants `bun_high`, `bun_back` (Katara: at the back), `bun_double` (Toph: a wide bun with the ornament loops: two 1×3×1 arcs) | Toph, Bumi, Katara, Pakku, Iroh, Jeong Jeong, Ozai, Azula, Ghazan, Zaheer |
| `CrownLayer` | head (over a topknot) | the Fire Nation **flame crown**: a 1 px thick, 5 px tall flame silhouette (3 cubes stepped) in gold, in front of the bun; Ozai's has five prongs (a wider 7 px flame), Azula's three; Bumi's **Earth crown**: a green 7×2×7 band with a gold 2×2 gem | Ozai, Azula, Bumi |
| `HairLayer` | head | long hair falling behind the head: a 8×6×1 slab at the back (−4..4, −2..4, 4..5) + two 2×4×1 side locks; variants `shaggy` (Zuko: slabs at the sides and back 1 px out, no slab below the ears), `bob` (Jinora: a 9×5×9 shell 0.6 px out), `braid` (Katara: a 1×6×1 braid at the back plus two loops `(2×3×1)` beside the face) | Zuko, Jinora, Katara, Ming-Hua (ponytail: a 2×7×2 tail), Suyin |
| `BeardLayer` | head | a 6×3×1 slab under the chin (−3..3, 0..3, −4.5..−3.5) + a 4×2×1 moustache; variants `goatee` (Ozai, Ghazan: a 2×3×1 point), `full` (Zaheer, Iroh), `moustache_long` (Gyatso: two 1×5×1 drooping strands), `pointed` (Tenzin: a 4×4×1 tapering slab) | Tenzin, Zaheer, Gyatso, Iroh, Pakku, Jeong Jeong, Ozai, Ghazan, Karthik |
| `RobeLayer` | body (hips) | a **skirt of robes**: four 1-px plates (front/back 8×8×1, sides 4×8×1) hanging from the hips (y 12..20 in body space), swinging ±8° with the legs' walk (driven by `rightLeg.xRot`), split at the front; variants `long` (to the ankles, 12 px), `short` (to the knees, 6 px), `cape_wrap` (Tenzin: a back plate only, 8×14×1, from the shoulders) | Tenzin, Gyatso, Jinora, Bumi, Suyin, Pakku, Iroh, Jeong Jeong, Ozai, Amon, Azula (short) |
| `ShoulderLayer` | body (shoulders) | two 5×3×5 pads above the shoulders (pointed: an extra 1 px spike) | Azula, Ozai, Lin (flat plates), Jae (brigade pads) |
| `BackLayer` | body (back) | a cable spool (Lin: a 6×6×3 drum with a 1 px rim), a glider staff when stowed (airbenders with a glider in the inventory: a 1×14×1 staff diagonal across the back), the Water Skin (a 4×6×2 pouch at the right hip) | Lin, airbenders, waterbenders |
| `HoodLayer` | head | Amon's hood: a 10×10×10 shell open at the face, drooping to the shoulders (two 3×6×1 flaps) | Amon |
| `MaskLayer` | head | a 8×8×1 plate on the face 0.5 px out with the painted features (Amon) | Amon |
| `ArmBandLayer` | arms | 4×2×4 bands on the upper arms (Air Nomad arm wraps, Ghazan's bands, Bolin's bracers) | several |
| `EarthArmourLayer` | whole body | the stone suit: an inflated `PlayerModel` mesh (`CubeDeformation(0.35)`, built as `LayerDefinition.create(PlayerModel.createMesh(…))` like Enhanced's `ENERGY_WIDE/SLIM`) with the **eye slit** cut from the head cube (the head is 4 cubes: brow 8×3×8, cheeks 8×2×8, jaw 8×3×8, and two 2×2×8 side pieces leaving a 8×2 slit), textured `earth_armour.png` (stone plates with cracks; tinted by the ground's map colour at forming time: `argb` tint); split into **6 slice parts** for `LayerReveal` (`ANIMATION.md` 14.10); worn over the look, hiding the skin | Earth Armour |
| `SteelArmourLayer` | whole body | the same mesh, `steel_armour.png` (riveted iron plates, Lin's police armour's colour), 5 slice parts | Steel Armour |
| `OctopusLayer` | whole body + 8 chains | the water body veil (`MATTER_TEXTURED(water_flow)` over the inflated mesh at α 0.55) and the tentacles (ribbon chains, `ANIMATION.md` 5.3): not a `ModelPart`, a renderer | Water Octopus |
| `WaterArmsLayer` | arms | inflated arm parts (`CubeDeformation(0.25)`) textured by the water sprite + the ribbon extensions | Water Arms |
| `GliderLayer` | hands (open) / back (stowed) | a staff 1×16×1 with two wings: each a 10×1×6 plate hinged at the staff's top, folding from z ±5° (closed) to ±85° (open), textured `glider_<dye>.png` (16 colours: one texture tinted by the dye's `MapColor`), a cloth seam line | Air Glider |
| `WaterSkinLayer` | right hip | a 4×6×2 leather pouch with a cork, its fullness shown by the pouch's darkness (3 texture stages) | Water Skin |

**Slim vs wide:** slim (3-px arms): Toph, Lin, Suyin, Jinora, Katara, Ming-Hua, Azula. Wide: everyone else. (Bumi,
Ghazan, Bolin, Zaheer, Iroh, Ozai are broad: the wide model; Tenzin, Gyatso, Zuko, Jeong Jeong, Pakku, Amon, Jae, Karthik:
wide.)

**Faces:** every face is drawn in the shows' flat style: almond eyes 2 px wide with a 1 px iris of the character's eye
colour (Earth Kingdom green, Water Tribe blue, Fire Nation gold, Air Nomad grey), a 1 px mouth, eyebrows 1 px. Skin tones
per nation: Earth Kingdom light-medium `#E8C39E` / `#D9A878`; Water Tribe medium-dark `#A8744F` / `#8B5A3C`; Fire Nation
light `#F0D2B4`; Air Nomads light `#EBCDA8`; Seven Havens' South-Asian-inspired Ellora: medium `#B98156` / `#A06A45`.

**Approval line:** each look ends with one or two sentences Eilon can say yes or no to (Hebrew in `SUGGESTIONS.md`).

---

## 1. Earth — the looks and the Earth layers

### 1.1 Toph Beifong — טוף בייפונג (default Earth look)

* **Reference.** *ATLA*, Books 2–3: a short girl of twelve; **black hair in a large, high bun** held by a **green
  headband** with the family's **hair ornament**: a pale green band with two loops over the bun, and thick bangs
  hanging over her **pale, unfocused grey-green eyes** (blind). Outfit: a **yellow-green sleeveless tunic** (`#C8C06A`)
  with **dark green trim** (`#3F6B3A`), a **cream long-sleeved undershirt**... (in Book 2 her arms are covered to the
  wrist in pale green; Book 3 Fire Nation disguise is not used), **dark green trousers** to the shin, a **dark green
  belt**, pale green wristbands, and **bare feet** (she sees through her feet). Earth Kingdom light skin.
* **Skin.** Slim. Base: the tunic and undershirt, the belt, the trousers ending at the shin, **bare feet** drawn as skin
  with toes (the legs' lower 4 rows), the bangs on the face's top 3 rows with the eyes beneath them pale (`#B6C9A6`
  irises, no pupils: the blind look), small mouth. Overlay: the hat layer carries the bangs' volume and the headband;
  the jacket layer the tunic's loose edges.
* **Layers.** `TopknotLayer bun_double` (`toph_bun.png`: black bun, the green band, the two pale loops).
* **Effects.** Element colours as Earth's default (ochre). **(suggestion)** Toph's rocks are a shade darker (`#6B5B3E`
  dust): no, keep one Earth palette; **her ambient master effect** is the grain ring at bare feet (no change).
* **Approval:** Toph as in Book 2: green and yellow tunic, big bun with the headband loops, bangs over pale eyes,
  **bare feet**. The default look for a new earthbender.

### 1.2 King Bumi — המלך בומי

* **Reference.** *ATLA*: the ancient king of Omashu, 112, wiry and then hugely muscular under the robes: **white hair
  in a tall, messy topknot**, **bushy white eyebrows**, a **white goatee** with a thin moustache, **one eye wider than
  the other**, a mad grin with a few crooked teeth; **green royal robes** (`#4F7A3F`) with **gold trim** (`#D4AF37`), a
  wide gold-trimmed collar, and a small **gold Earth Kingdom crown** (a green band with a gold leaf). Light skin, lined.
* **Skin.** Wide. Base: the green robe with the gold collar and trim down the front, gold cuffs, green boots; the face
  with the heavy brows (2 rows), the uneven eyes (one 2×2 open, one 2×1 squint), the grin (3 px with a dark gap). Overlay:
  the robe's folds; the jacket layer carries the collar's standing edge.
* **Layers.** `TopknotLayer bun_high` (white, tall: the bun is 3×4×3), `CrownLayer earth` (the green band with the gold
  gem), `BeardLayer goatee` (white), `RobeLayer long` (`bumi_robe.png`).
* **Effects.** Default Earth. **(suggestion)** his idle hint includes a slow head tilt (the madness): a `head(0, 0, 6)`
  ambient at weight 0.3: client-only, cheap, flagged for Eilon.
* **Approval:** old King Bumi in green and gold robes with his tall white topknot, crown and crazy grin.

### 1.3 Lin Beifong — לין בייפונג

* **Reference.** *Korra*: Chief of Republic City's metalbending police, in her fifties: **short grey hair** swept
  back and up, **two parallel scars on her right cheek**, **light green eyes**, a stern mouth. The **metalbending police
  armour**: **dark grey-green metal** (`#4A5550`) plates over a **dark olive uniform** (`#2F3A32`), a wide armoured
  collar, segmented shoulder and chest plates, armoured forearms and boots, and the **cable spool** on the back (two
  spools of metal cable at the lower back). Light skin.
* **Skin.** Slim. Base: the dark uniform with the plates painted on the body and limbs (lighter plate edges `#7C8A82`),
  the collar, the boots; the face with the two scars (two 1-px diagonal lines `#B88A78` on the right cheek), grey hair on
  the head's top and back (the hat layer adds its upswept volume). Overlay: the plates' relief on the jacket and sleeve
  layers.
* **Layers.** `ShoulderLayer flat` (`lin_shoulders.png`: squared plates), `BackLayer spool` (`lin_spool.png`).
* **Effects.** Default Earth; **her metal effects are the police steel grey** `#8C96A0` instead of the iron `#B8C0CC`
  (the metal filings and plates tint by look: a `metalColor` per look, as Enhanced's variants carry a `color`/`glow`).
* **Approval:** Chief Lin in the dark metalbender armour with the cable spool, short grey hair and the two cheek scars.

### 1.4 Suyin Beifong — סויין בייפונג

* **Reference.** *Korra*, Books 3–4: Zaofu's matriarch: **dark hair pulled back** into a low bun with a **silver and
  green metal headpiece** (a flat metal clip arcing over the hairline), **light green eyes**, a small beauty mark under
  the right eye (**check against a picture**), an elegant **green and silver Zaofu robe** (`#3E7F5A` green with
  `#B9C4CC` silver panels and a dark green sash), long sleeves, a high collar, the **metal clan's silver accents** at
  the collar and cuffs. Light skin.
* **Skin.** Slim. Base: the green robe with silver panels, the sash, dark boots; the face with the mark; dark hair at
  the top and sides. Overlay: the robe's collar and cuffs in silver.
* **Layers.** `TopknotLayer bun_back` (dark, low) with the headpiece painted on the bun's front, `RobeLayer long`
  (`suyin_robe.png`, split at the sides).
* **Effects.** Metal effects in **Zaofu platinum-silver** `#C8D2DA`.
* **Approval:** Suyin in a green and silver Zaofu robe with the metal headpiece and a low bun.

### 1.5 Ghazan — גאזאן

* **Reference.** *Korra*, Book 3: the Red Lotus lavabender: a big, calm man with **dark brown hair tied back in a
  topknot/ponytail** with **shaved or short sides**, a **moustache and a pointed goatee**, a **lazy half-smile**, dark
  eyes, **tan Earth Kingdom skin** (`#C99667`); **a sleeveless dark green tunic** (`#2F4F3A`) open at the chest over a
  **brown undershirt**, **brown trousers** (`#5A4634`), **bracers** and arm bands, a red-brown sash; **(check:** whether
  he has a tattoo on an arm).
* **Skin.** Wide. Base: bare upper arms (tan), the tunic and bands, the trousers and boots; the face with the goatee
  (the hat layer gives the moustache its stand-off). Overlay: the open tunic's edges on the jacket layer.
* **Layers.** `TopknotLayer bun_high` (dark, a tail: the `tail` variant adds a 1×4×1 drop behind), `BeardLayer goatee`,
  `ArmBandLayer`.
* **Effects.** Default Earth; **his lava is the show's orange-red** (the one Lava palette: no change). **(suggestion)**
  Ghazan's look makes the Lava Path's crust a bit darker (`#1E120C`): a per-look lava tint, trivial.
* **Approval:** Ghazan: sleeveless green tunic, bare tanned arms, a topknot, a moustache and goatee.

### 1.6 Bolin — בולין

* **Reference.** *Korra*: a stocky, cheerful teen: **black hair swept back** with a **curl at the front**, **bright
  green eyes**, thick eyebrows, a big grin, fair skin; Book 1 outfit: a **dark green sleeveless vest/jacket** (`#2E5A3A`)
  with a **grey fur collar** (`#8E8E8A`) over a **cream shirt**, **brown wrist wraps**, a brown belt, **dark trousers**
  (`#3A3A3A`) and boots. (Pabu, the fire ferret, is a companion suggestion, not part of the look.)
* **Skin.** Wide. Base: the vest and shirt, fur collar painted at the neck, the belt, the trousers; the face with heavy
  brows and the grin; the hair's curl on the brow's top row. Overlay: the fur collar's volume on the jacket layer, the
  curl on the hat layer.
* **Layers.** none beyond the overlays (Bolin reads well with the second skin alone).
* **Effects.** Default Earth and Lava.
* **Approval:** Bolin in his green vest with the grey fur collar and his swept-back hair with the curl.

### 1.7 The Earth layers' art

* **Earth Armour** (`earth_armour.png`, 64×64 for the inflated mesh): large rough plates with dark seams, lighter
  chipped edges, three overlay crack textures (`earth_armour_cracks_1..3.png`, alpha only). Tinted at forming by the
  ground: grass → `#6E5B3D`, stone → `#8A8A86`, sandstone → `#C9B37C`, deepslate → `#4E4E52`, netherrack → `#7A3B3B`,
  end stone → `#D8D6A8` (the block's `MapColor`).
* **Steel Armour** (`steel_armour.png`): riveted plates, a seam down the chest, the police grey `#6E7A80` with
  highlights `#B8C4CC`; a dent overlay (`steel_dents_1..3.png`); a highlight mask for the glint sweep.
* **Rock** (block displays: no texture). **Quicksand block** (`quicksand.png`: 4-frame rippling sand). **Wrecked
  devices** (`wrecked_*.png`: one "bent metal" tile set with the device's base colour tint: 8 textures for the device
  families). **Steel plate block** (`steel_plate.png`). **Lava path crust** (`temp_lava` uses vanilla lava; `lava_crust.png`
  for the cooling overlay). **Ice spike**, **ice wall** (vanilla ice). **Sand grains** (`particle/grain.png`, 8×8),
  **droplets** (`particle/droplet.png`).

---

## 2. Air — the looks and the Air layers

### 2.1 Tenzin — טנזין (default Air look)

* **Reference.** *Korra*: Aang's son, in his fifties: **bald** with the **blue Air Nomad arrow tattoo** (`#2E6FBF`)
  from the brow over the crown and down the back, arrows on the backs of the hands; a **dark brown-grey pointed beard**
  (greying, `#5A4A3E`) and a moustache; **grey eyes**; a serious, kind face. Korra-era airbender robes: a **yellow-orange
  robe** (`#E9A23B`) with **maroon-red wraps** (`#8E2F2F`) over the shoulders and a **red cape-like shawl** down the
  back, a **red sash** at the waist, **yellow trousers**, **brown boots**, orange arm wraps.
* **Skin.** Wide. Base: the robe and wraps, the sash, the trousers and boots; the head bald with the arrow drawn from
  the brow (a 2-px wide arrow point between the eyes widening to 4 px at the crown) over the top to the nape; grey eyes.
  Overlay: the shoulder wraps' volume, the hand arrows on the arm overlays.
* **Layers.** `BeardLayer pointed` (`tenzin_beard.png`), `RobeLayer cape_wrap` (`tenzin_cape.png`: the maroon shawl
  down the back, swinging), `BackLayer glider` when a glider is carried.
* **Effects.** Default Air (saffron/sky).
* **Approval:** Tenzin: bald with the blue arrow, the pointed beard, yellow robes with the red shoulder wraps and shawl.
  The default look for a new airbender.

### 2.2 Zaheer — זהיר

* **Reference.** *Korra*, Book 3: the Red Lotus leader: a lean, muscular man around forty: **dark hair tied back**
  low (**check:** in some episodes he is shown with shaved head and beard; the plan uses the tied-back hair of his
  escape), a **full dark beard** cropped short, **grey eyes**, tan-light skin, intense brows. Outfit: a **dark olive
  sleeveless tunic** (`#4A4A33`) with a **dark red wrap** (`#7A2A2A`) across the chest and waist, **brown trousers**,
  **bracers**, bare upper arms. After gaining flight: the same clothes, a wind always moving them.
* **Skin.** Wide. Base: the tunic, wrap, trousers, bracers, bare arms; the face with the beard's shape (the hat layer
  gives the beard volume), the tied hair at the back. Overlay: the wrap's folds.
* **Layers.** `BeardLayer full`, `TopknotLayer tail` (a low tail), `ArmBandLayer` (bracers).
* **Effects.** Default Air; with **Zaheer's Path** open, every airbender's pushes get a faint **red thread** motif in
  the HUD only (no world effect change). **(suggestion)** Zaheer's look makes the True Flight wake slightly darker
  (`#D8D8D8`): cosmetic per-look `color`.
* **Approval:** Zaheer: tied-back dark hair, a full short beard, an olive sleeveless tunic with the dark red wrap.
  Confirm the hair (tied back, not shaved).

### 2.3 Jinora — ג'ינורה

* **Reference.** *Korra*: Tenzin's daughter; Book 4 (an airbending master at 14): **short brown hair** (`#5B3F2A`)
  in a bob with a side part, the **blue arrow tattoo** on her brow (and hands), **brown eyes**, a calm, bookish face;
  the Korra-era **airbender wingsuit/robes**: **yellow-orange** (`#E9A23B`) top with a **red** (`#8E2F2F`) collar and
  sash, yellow trousers to the ankle, brown soft boots. Light skin.
* **Skin.** Slim. Base: the robes, the sash, boots; the face with the arrow point on the brow and the brown bob around
  it; brown eyes. Overlay: the bob's volume on the hat layer (the `HairLayer bob` adds the rest).
* **Layers.** `HairLayer bob` (`jinora_hair.png`), `RobeLayer short`.
* **Effects.** Default Air; **(suggestion)** her **Spiritual Projection spirit glows brighter** (`#BFE8FF` α 0.55
  instead of 0.45): she is the show's spirit expert; cosmetic.
* **Approval:** Jinora as a young master: brown bob, the arrow on her forehead, yellow and red airbender robes.

### 2.4 Monk Gyatso — הנזיר גיאטסו

* **Reference.** *ATLA* flashbacks: Aang's elderly mentor: **bald** with the **arrow tattoo**, a **long white
  moustache** drooping past the chin, bushy white eyebrows, warm **grey eyes**, laugh lines; the classic **Air Nomad monk
  robes**: an **orange** (`#E07A1F`) over-robe with a **yellow** (`#F2C14E`) under-robe and a **saffron shawl**, a
  **wooden Air Nomad necklace** (a brown cord with a round pendant), bare feet or simple sandals (the show: bare
  feet with the arrows).
* **Skin.** Wide. Base: the robes and shawl, the necklace, the trousers in yellow, bare feet with arrows; the face with
  the brows and the moustache's root (the layer gives its length). Overlay: the shawl.
* **Layers.** `BeardLayer moustache_long` (`gyatso_moustache.png`), `RobeLayer long`.
* **Effects.** Default Air.
* **Approval:** Gyatso: old bald monk with the arrow, the long white moustache and orange and yellow robes.

### 2.5 Jae — ג'יי (*Avatar: Seven Havens*)

* **What is known (from reviews, interviews and fan wikis; the series' art book was not reachable):** an **airbender
  of Ellora's Bending Brigade**, 19, a platoon leader who becomes the twin Avatars' friend, mentor and airbending
  master, expressive, calm under pressure; the first on-screen gay male lead of the franchise (his boyfriend is Datu).
  Uniform: the **Bending Brigade's form-fitting, futuristic soldier's uniform in the Air Nation's colours: dark yellow,
  dark orange and light grey**, with a **triangular emblem in blue and dark orange on the chest plate** that echoes the
  Air Nomad arrow, **a running-style belt with pockets** on each side, **loose brown trousers**, **grey cuisse-style leg
  armour**, **knee-high boots in grey and dark orange**, and **shoulder pads** (the Brigade uniforms resemble Korra's
  pro-bending uniforms without helmet or arm guards). Gear: a **fold-up board with a hole in the middle that an air
  scooter fills** (a hover-board: a lovely glider alternative, 2.7), and he flies **gliders**.
* **What needs a picture from Eilon (open question 4):** hair colour and style, skin tone, whether he has arrow
  tattoos, the exact colours of the uniform panels and boots, any facial marks. **Until then the art lane draws a
  placeholder** (dark hair, short; Ellora medium skin tone; the uniform from the description) and marks the look
  "draft" in the look picker.
* **Skin (plan).** Wide. Base: the yellow/orange/grey uniform with the triangular chest emblem, the belt with pockets,
  brown trousers, grey leg armour on the legs' overlays, the boots. Overlay: the uniform's panel relief.
* **Layers.** `ShoulderLayer brigade` (`jae_shoulders.png`), optional `BackLayer board` (the folded board on the back
  when the scooter ability is carried: **(suggestion)**).
* **Effects.** Default Air; **(suggestion)** his Air Scooter is drawn **inside Jae's board** (a flat ring mesh around
  the ball) when his look is on: a per-look scooter style.
* **Approval:** Jae in the Bending Brigade uniform (yellow, orange, grey, the triangle emblem); **send a picture for the
  face and hair.**

### 2.6 Karthik — קארתיק (*Avatar: Seven Havens*)

* **What is known:** an **airbender of the Order of the White Lotus**, "Captain Karthik", a tall man in his forties:
  **brown hair with white streaks coming in underneath**, **scruff / a short beard**, **tired eyes**, gentle and calm
  but stern and forceful with orders; Nisha's surrogate father; nicknamed "the Human Hurricane" and "Warrior of the
  Wind"; voiced by Akshay Khanna; fans debate whether he is a villain.
* **What needs a picture:** his **outfit and its colours** (White Lotus of the Havens era: unknown whether white and
  blue), his skin tone, hair length, any emblem. Placeholder until then: a **white and pale-blue long coat** with the
  **white lotus tile** on the chest (the Order's symbol from *ATLA*), brown hair with white at the temples and nape,
  a short beard, Ellora medium skin.
* **Skin (plan).** Wide. Base: the coat, a dark under-tunic, trousers, boots; the face with the beard's root and the
  white-streaked hair. Overlay: the coat's lapels.
* **Layers.** `BeardLayer full` (short, brown with grey), `RobeLayer long` (the coat's skirt), `HairLayer short_back`
  (the white streaks at the nape).
* **Effects.** Default Air; **(suggestion)** his Tornado is 10% wider (the Human Hurricane): no, cosmetic only: a
  per-look `tornadoTexture` with a darker swirl.
* **Approval:** Karthik as a White Lotus airbender: brown hair with white streaks, a short beard, a white and blue coat
  with the lotus; **send a picture for the outfit.**

### 2.7 The Air layers' art

* **Glider** (`glider_base.png`, tinted by dye): a dark wooden staff, two cloth wings with a seam; 16 dyes → the dye's
  `MapColor` tint; the default saffron (`#E9A23B`). Stowed: the staff on the back. **(suggestion)** `jae_board.png`: Jae's
  fold-up board as an alternate glider skin, chosen in the look picker when Jae is the look.
* **Air textures:** `air_lattice.png` (a soft swirl lattice, white on transparent, 64×64 tileable) for the shield and
  scooter; `air_swirl.png` (a streaky wisp texture, 64×64) for tornadoes and spouts; `eyes_closed.png` (two 2×1 lid
  strips in the face's UV, tinted at runtime).
* **Spirit:** no texture: the player's skin tinted blue.

---

## 3. Water — the looks and the Water layers

### 3.1 Katara — קטארה (default Water look)

* **Reference.** *ATLA*, Book 3 (her most familiar look): a girl of fourteen: **dark brown hair** (`#3A2518`) in a
  **high bun at the back** with two **"hair loopies"** (loops of hair framing the face from the temples to the chin)
  and bangs parted at the middle; **blue eyes** (`#3B7DD8`); **medium-dark Water Tribe skin** (`#A8744F`); her
  **mother's necklace**: a **blue choker band with a carved pale blue pendant**. Outfit: a **blue Water Tribe tunic**
  (`#2F5F9E`) with **white fur trim** at the collar and hem (`#EDEDE4`), **dark blue sleeves**, a **lighter blue sash**,
  dark blue trousers tucked into **fur-trimmed boots**, white wristbands. (Book 1's parka with the thick fur is a
  **(suggestion)** second variant `katara_parka`.)
* **Skin.** Slim. Base: the tunic, sash, trousers, boots with white trim; the necklace at the neck (1 row of blue with
  a 2×1 pale pendant); the face with the bangs; blue eyes. Overlay: the fur trim's volume, the sleeves.
* **Layers.** `TopknotLayer bun_back` + `HairLayer braid` (the loops: `katara_hair.png`), `WaterSkinLayer` when a skin
  is carried.
* **Effects.** Default Water (sea blue). Her healing glow is the default `#8FE3FF`.
* **Approval:** Katara in blue with the white fur trim, her bun and hair loops, and her mother's necklace. The default
  look for a new waterbender.

### 3.2 Amon / Noatak — אמון / נואטק

* **Reference.** *Korra*, Book 1: the Equalist leader: a **white-cream face mask** (`#EDE6D6`) with a **red circle on
  the forehead** (`#B22222`), narrow dark eye slits, painted brows and a thin red line for the mouth, under a **dark
  hood** (`#1F2228`); a **long dark coat** with a **high collar**, a **dark red lining** (`#4A1A1A`) showing at the
  cuffs and the hem, dark gloves, boots. **Noatak** unmasked (Book 1 finale): a man around forty with **dark brown hair
  swept back**, **blue eyes**, a strong jaw, a faint painted "scar" washed off: **(suggestion)** a second variant
  `noatak` without the mask and hood, same coat.
* **Skin.** Wide. Base: the coat with the collar and red lining, gloves, boots; the head: under the mask, Noatak's face
  (so the `noatak` variant is the same skin with the mask and hood layers off). Overlay: the collar.
* **Layers.** `MaskLayer` (`amon_mask.png`), `HoodLayer` (`amon_hood.png`), `RobeLayer long` (the coat's skirt).
* **Effects.** Default Water; **his blood-bending threads are darker** (`#5A0A14`) and his Severance pulse is **black**
  (`#0A0608`): a per-look `bloodColor`. His Water Shot is the default blue (he is still a waterbender).
* **Approval:** Amon with the white mask, red circle, dark hood and coat; and, if you want, Noatak's face without the
  mask as a second look.

### 3.3 Ming-Hua — מינג-הואה

* **Reference.** *Korra*, Book 3: the Red Lotus waterbender **born without arms**; a pale woman with **long black
  hair** falling loose with a **thin braid** at each side (**check:** a high ponytail in some scenes), **dark eyes**,
  **dark lips**, a sharp face; a **dark blue sleeveless tunic** (`#1F3A66`) with **grey wraps** around the torso, dark
  trousers, bare shoulders where her water arms root.
* **Skin.** Slim. Base: the tunic and wraps, trousers, boots; the face with the dark lips; **the arm regions are drawn
  transparent above the shoulder stumps** (the top 2 rows of each arm are skin: the stumps; the rest of each arm is
  **transparent**), so without Water Arms the model's arms are invisible but for the stumps: Ming-Hua has no arms. This
  is the one look that changes the body's silhouette; it is cosmetic (the hitbox is the player's).
* **Layers.** `HairLayer long` + braids (`minghua_hair.png`). With **Water Arms on**, the `WaterArmsLayer` draws her
  arms as water from the shoulders (the inflated arm parts fully textured in water, as the show): the look is made for
  the ability. Without Water Arms she **punches with invisible arms**: **(suggestion)** her look draws **thin water
  stubs** (0.4-block ribbons) from the stumps whenever a source is in reach, as a hint; decide.
* **Effects.** Default Water; her ice tips are the default.
* **Approval:** Ming-Hua with long black hair, dark blue tunic, and **no arms** (water arms when the ability is on).
  Say whether the arms should stay invisible when Water Arms is off, or show short water stubs.

### 3.4 Master Pakku — מאסטר פאקו (White Lotus)

* **Reference.** *ATLA*, Book 3 finale: the Northern Water Tribe master, old and stern: **grey hair in a high
  topknot**, a **grey moustache and a short grey goatee**, wrinkles, **sharp blue eyes**, medium Water Tribe skin.
  The **Order of the White Lotus** robes of the finale: **white** (`#F2F0E8`) long robes with **pale blue** (`#9CC4E8`)
  trim at the collar, cuffs and hem, a **white lotus tile emblem** (a white lotus on a blue disc) on the chest, a blue
  sash, white boots. (**Check:** the exact trim colour; the Order's soldiers wear white with a tall white hat; the
  elders' robes are white with blue.)
* **Skin.** Wide. Base: the white robes with blue trim and the emblem, the sash, boots; the face with the brows and the
  goatee's root; grey hair. Overlay: the collar.
* **Layers.** `TopknotLayer bun_high` (grey), `BeardLayer goatee` (grey), `RobeLayer long` (`white_lotus_robe.png`:
  **shared by Pakku, Iroh and Jeong Jeong** with a per-look tint of the trim: Pakku blue, Iroh red-gold, Jeong Jeong red).
* **Effects.** Default Water.
* **Approval:** old Master Pakku in the white and blue White Lotus robes with the lotus emblem, grey topknot and goatee.

### 3.5 The Water layers' art

* **Water Skin** (`water_skin.png`: a leather pouch, 3 fullness stages), the item icon (16×16, 9 stages for the bar).
* **Water textures:** vanilla's `block/water_still` and `block/water_flow` sprites (looked up from the block atlas,
  tinted by the biome's water colour), `water_foam.png` (a white foam ribbon), `ice_frost.png` (frost lines overlay),
  `wet.png` (a dark wet decal), `quicksand`... (Earth's).
* **Octopus / Arms:** no textures beyond the water sprites; the tentacle ribbons use `water_flow` with a highlight.
* **Blood:** the thread colour per look (`bloodColor`), the moon overlay (`moon_full.png`, 32×32, for the HUD and the
  vignette corner).

---

## 4. Fire — the looks and the Fire layers

### 4.1 Zuko — זוקו (default Fire look; "after he joined Team Avatar")

* **Reference.** *ATLA*, Book 3 (Western Air Temple onward): a sixteen-year-old with **shaggy black hair** grown out to
  the jaw, falling over his brow, the **burn scar over his left eye** (`#A0564A` and `#B86E5C`, from the brow to the
  ear, the eye narrowed), **gold eyes** (`#D8A62A`), pale Fire Nation skin. Outfit: the **dark red Fire Nation tunic**
  (`#7A1F1F`) with a **darker, almost black** (`#2A1A1A`) under-layer at the shoulders and a **dark brown belt**, **gold
  trim** (`#C9A24A`) at the collar, **red trousers**, dark boots, red arm wraps with gold edges.
* **Skin.** Wide. Base: the tunic and under-layer, belt, trousers, boots, wraps; the face with the scar (a 3×3 area of
  darker tones around the left eye, the eye 1 px narrower), gold eyes, the hair's fringe. Overlay: the hair's shag on the
  hat layer (the `HairLayer shaggy` adds the sides).
* **Layers.** `HairLayer shaggy` (`zuko_hair.png`).
* **Effects.** Default Fire (orange/gold).
* **Approval:** Zuko with the shaggy hair and his scar, in the dark red and gold tunic of Book 3. The default look for a
  new firebender.

### 4.2 Uncle Iroh — הדוד אירו (White Lotus)

* **Reference.** *ATLA*: the Dragon of the West, heavyset and jolly: **grey hair in a topknot**, a **full grey beard**
  and moustache, **warm gold eyes** with laugh lines, pale skin. The **White Lotus robes** of the finale (white with the
  Order's trim; Iroh's **(check)** carries a red-gold trim as the Grand Lotus), the lotus emblem, wide sleeves. (His
  Book 2 green Earth Kingdom traveller's robe is a **(suggestion)** second variant `iroh_traveller`; his tea.)
* **Skin.** Wide. Base: the white robes with the trim and emblem, a wide sash, boots; the round face with the beard's
  root and the brows; grey hair. Overlay: the wide sleeves.
* **Layers.** `TopknotLayer bun_high` (grey), `BeardLayer full` (grey, wide: `iroh_beard.png`), `RobeLayer long`
  (`white_lotus_robe.png`, red-gold trim).
* **Effects.** Default Fire; **his Fire Breath is the default** but **(suggestion)** his look makes the breath's jet
  wider by the "dragon" texture: cosmetic no; keep default.
* **Approval:** Uncle Iroh in the white White Lotus robes with the lotus, his grey topknot and full beard.

### 4.3 Princess Azula — הנסיכה אזולה (blue fire)

* **Reference.** *ATLA*: **black hair in a high topknot** with the **gold flame crown** (three prongs) and **two long
  bangs** framing her face to the chin, **gold eyes** sharp and lined, **red lipstick**, pale skin, a cruel smile. The
  **Fire Nation royal armour**: **dark red/maroon** (`#6E1B1B`) with **black** panels, **gold trim**, **pointed gold
  shoulder pads**, armoured cuffs, a gold belt plate with the Fire Nation flame, dark trousers and boots.
* **Skin.** Slim. Base: the armour's panels, gold trim, the belt plate, the cuffs, boots; the face with the bangs (two
  1-px columns beside the face on the hat layer), the eyes and lips; black hair. Overlay: the armour's relief.
* **Layers.** `TopknotLayer bun_high` (black), `CrownLayer flame` (gold, three prongs), `ShoulderLayer pointed`
  (`azula_shoulders.png`), `RobeLayer short` (the armour's skirt plates).
* **Effects. Every fire effect is blue**: the FLAME `BeamStyle` becomes `BLUE_FLAME` (core `#F4FDFF`, inner `#CFF2FF`,
  main `#6FC4FF`, body `#3FA4FF`, fringe `#1B4FCC`), every `FLAME` particle becomes `SOUL_FIRE_FLAME`, fireballs' cores
  `#E8FBFF` / rims `#3FA4FF`, scorch discs stay dark, embers `#9FD8FF`; sounds get a hissing top layer (pitch 1.15) for the
  jets. Lightning stays white-blue. The look's `color` is `#3FA4FF`, `glow` `#CFF2FF`.
* **Approval:** Azula in the dark red royal armour with pointed gold shoulders, the flame crown, two bangs and red lips;
  **all her fire is blue.**

### 4.4 Jeong Jeong — ג'ונג ג'ונג (White Lotus)

* **Reference.** *ATLA*: the Deserter: an old, gaunt master with **long white hair** swept back and falling to the
  shoulders, a **long white moustache** and a **wispy white beard**, a **scar across his right eye** (a vertical line from
  brow to cheek), **gold eyes**, pale weathered skin. The **White Lotus robes** (white with red trim as a firebender of
  the Order) over his old dark red rags **(check)**.
* **Skin.** Wide. Base: the white robes with red trim and emblem; the face with the scar line, the brows; white hair.
  Overlay: the robes.
* **Layers.** `HairLayer long_back` (white, `jeongjeong_hair.png`), `BeardLayer moustache_long` + a wispy beard slab,
  `RobeLayer long` (`white_lotus_robe.png`, red trim).
* **Effects.** Default Fire.
* **Approval:** Jeong Jeong: long white hair and moustache, the scar over his eye, white White Lotus robes with red trim.

### 4.5 Fire Lord Ozai — אדון האש אוזאי

* **Reference.** *ATLA*: the Fire Lord: **long black hair** in a **high topknot** with the **five-pronged golden flame
  crown** of the Fire Lord, a **pointed black goatee** and a thin moustache, **cold gold eyes**, pale skin, sharp
  cheekbones. The **royal robes**: **deep red** (`#7A1515`) with **black** and **gold** (`#D4AF37`) armour at the
  shoulders (large pointed pauldrons), a gold-trimmed black mantle, a wide gold belt with the flame, long red robes,
  black boots. (Bare-chested Phoenix King of the finale: a **(suggestion)** second variant `phoenix_king`.)
* **Skin.** Wide. Base: the robes, mantle, belt, boots; the face with the goatee's root and the moustache, the
  brows; black hair at the back. Overlay: the mantle.
* **Layers.** `TopknotLayer bun_high` (black, tall), `CrownLayer flame_five` (gold, five prongs: 7 px wide),
  `ShoulderLayer pointed` (`ozai_shoulders.png`, larger), `BeardLayer goatee` (black), `RobeLayer long` (`ozai_robe.png`).
* **Effects.** Default Fire; **(suggestion)** Ozai's Fire Jet and great fireball are **10% bigger** cosmetically (no
  rule change: the mesh scale only). His lightning is the default.
* **Approval:** Fire Lord Ozai: long black topknot with the five-pronged crown, pointed goatee, deep red and gold royal
  robes with big shoulder armour.

### 4.6 The Fire layers' art

* `white_lotus_robe.png` (shared, trim tinted), the flame crowns (`crown_flame_3.png`, `crown_flame_5.png`), the
  shoulder pads, the beards and topknots as above; `fire_mouth.png` (the breathing glow mask); `heat.png` (a heat
  shimmer noise texture, 64×64, tileable); `scorch.png` (a dark burn decal).

---

## 5. Items drawn on the body

| Item | Where | Texture | Notes |
|---|---|---|---|
| Air Glider (stowed) | across the back | `glider_base.png` tinted | only for airbenders; hidden when a cape/robe layer would clip (the robe wins) |
| Water Skin | right hip | `water_skin.png` (3 stages) | waterbenders with a skin in the inventory; fullness from the fullest skin |
| Element Scroll (held) | hand | a vanilla-style item model `element_scroll.json` (a rolled scroll with the four symbols) | |

## 6. Icons (`UI.md` 6 has the full list)

Every look has a **face icon** for the look picker: the skin's face region (8×8) blown up, as Enhanced's
`PlayerFaceRenderer.draw` does from the skin texture; no separate art.

## 7. Checklist for the art lane (per look)

1. Draw the 64×64 skin (wide or slim as listed) from the reference text; check it with `SkinCheck` (64×64, PNG, valid).
2. Draw each layer texture and build its `LayerDefinition` (head space; test with `/bending_model look <id>` in a client).
3. Register the `Variant` (`id`, skin, `slim`, `color`, `glow`, and the look's extras: `metalColor`, `bloodColor`,
   `fireStyle`) in the element's `Looks` list; the lang keys `look.bending.<element>.<id>` in both languages.
4. Screenshot in a real client: front, back, side, first-person hands, with the Earth/Steel armour over it, with the
   octopus, with the glider; English and Hebrew HUD.
5. The `LooksTests` pin: every look has a skin PNG of 64×64, every layer texture exists, every look has both names, the
   face icon renders.
