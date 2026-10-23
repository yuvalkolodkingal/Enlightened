# Bending — Screens and HUD

Everything here is built with Enhanced's UI kit, copied into `bending.client.screen`: `CreatorUi.BigButton` (the card
widget: icon or face, label, sub-line, colour, selected/lit states), `EditorTab.cards(...)` (the paged grid), `Popup`,
`PreviewPuppet` (a `RemotePlayer` drawn in the GUI with `EntityRenderDispatcher`, animated by the animator's preview
API), `CatalogSearch` (two-language search with Hebrew folding), `HudDraw.visual` (forced right-to-left layout with ICU
`Bidi`), the generated config screens (`ConfigScreen`, `ConfigSectionScreen`, `ConfigListScreen`), and the
`LayeredDraw.Layer` HUD pattern registered through `ClientRegistrar.guiLayer`. None of the screens pause the game.

**Hebrew.** Every screen has a right-to-left layout when `Language.getInstance().isDefaultRightToLeft()`: columns are
mirrored (the preview moves to the right, lists to the left, text right-aligned), lines are laid out with `HudDraw.visual`
so a line that starts with a Latin key name (`[X]`) is still right-to-left, and units live inside the translation
(`hud.bending.seconds` = `%s ש׳`), as Enhanced learned. Numbers stay Latin digits. Key names from `getTranslatedKeyMessage()`
are left as the game gives them.

---

## 1. Keys and how the screens open

| Key | Default | Action |
|---|---|---|
| `key.bending.slot_1..4` | **X, C, Left Alt, K** | the loadout slots; Sneak + key is the variant (the client sends the Sneak flag with the press, Enhanced's `PRESS_SNEAK` capture) |
| `key.bending.menu` | **O** | opens the **Bending screen** (tabs); Sneak + O cycles the look at once |
| (none) | right-click | the Element Scroll (choice screen), the Water Skin (refill), the Air Glider (open/close) |
| `key.bending.menu` while a slot is held? | — | no chords beyond Sneak |

Category `key.categories.bending` = "Bending" / "כשפות". Context `IN_GAME` for all (so a clash with Enhanced's keys shows
in Controls: there is none by default).

---

## 2. Screens

### 2.1 Element Choice — בחירת אלמנט (`ElementChoiceScreen`)

Opened by the Element Scroll (server → `ElementScreenPayload`), or `/bending choose`. Four cards, a preview, a confirm.

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         Choose your element                              │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    ┌───────────┐  │
│   │  ▲ EARTH │ │  ◎ AIR   │ │  ≈ WATER │ │  ♦ FIRE  │    │           │  │
│   │  [Toph]  │ │ [Tenzin] │ │ [Katara] │ │  [Zuko]  │    │  (3D you  │  │
│   │ Rock Throw│ │ Air Blast│ │Water Shot│ │ Fireball │    │  in the   │  │
│   │ Earth Wall│ │Air Shield│ │ Healing  │ │Fire Whip │    │  element's│  │
│   │ Earth Arm.│ │Air Scoot.│ │Ice Knives│ │ Lightning│    │  default  │  │
│   │  Looks: 6 │ │  Looks: 6│ │  Looks: 4│ │  Looks: 5│    │   look)   │  │
│   └──────────┘ └──────────┘ └──────────┘ └──────────┘    └───────────┘  │
│        Strong, rooted. Earth Kingdom.  Water is strong against fire.      │
│                  [ Become an Earthbender ]        [ Not now ]             │
└──────────────────────────────────────────────────────────────────────────┘
```

* Cards are `BigButton`s 92 × 110 (clamped to the width: `cardW = clamp((width−24−3·GAP)/4 … )`, the `SerumScreen`
  recipe for three cards, widened to four), each with the element's **symbol icon** (6.1), its name bold in the element
  colour, the default look's **face** (`PlayerFaceRenderer`), three starter abilities in grey, "Looks: n"; a selected card
  gets the gold double outline; hovering plays a short puff of the element's particles behind the card (`CreatorUi`
  has no particles: a 6-mote `GuiGraphics` sprite flurry, cheap).
* The **preview** (`PreviewPuppet`, 96 px tall) shows **you** in the selected element's default look, playing that
  element's idle stance (`previewHold` of `EARTH_IDLE` etc.) and, every 4 s, its starter gesture (`previewCast`).
* The **description line** (2 lines max, `font.split`) and the **matchup hint**.
* **Become** sends `ElementChoicePayload(element)`; **Not now** closes (the scroll is kept). Keys 1–4 select, Enter
  confirms (as `SerumScreen`).
* **Hebrew (RTL):** the cards run right to left (Earth at the right), the preview sits at the **left**, text
  right-aligned:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           בחרו את האלמנט שלכם                            │
│  ┌───────────┐    ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │           │    │   אש ♦   │ │  מים ≈   │ │ אוויר ◎  │ │  אדמה ▲  │    │
│  │ (תצוגה    │    │  [זוקו]  │ │ [קטארה]  │ │ [טנזין]  │ │  [טוף]   │    │
│  │  תלת-ממד) │    │ כדור אש  │ │ יריית מים│ │ הדף אוויר│ │ זריקת סלע│    │
│  │           │    │ שוט אש   │ │  ריפוי   │ │ מגן אוויר│ │ קיר אדמה │    │
│  │           │    │  ברק     │ │סכיני קרח │ │קטנוע אווי│ │שריון אדמה│    │
│  │           │    │ מראות: 5 │ │ מראות: 4 │ │ מראות: 6 │ │ מראות: 6 │    │
│  └───────────┘    └──────────┘ └──────────┘ └──────────┘ └──────────┘    │
│                 .חזקה ומושרשת. ממלכת האדמה.  מים חזקים נגד אש             │
│              [ לא עכשיו ]        [ להפוך לכשף אדמה ]                      │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.2 The Bending screen — מסך הכשפות (`BendingScreen`, key O)

One screen, four tabs (Enhanced's `HeroEditorScreen` layout: tabs at y 16, a left preview column `left() =
clamp(width/3, 92, 160)`, the content page to the right of it, a status line in place of the title for 6 s after an
action). The **header** shows the element symbol, "Earthbender", **SP** and **ES**: `✦ 12 SP   ◆ ES 23`.

```
┌──────────────────────────────────────────────────────────────────────────┐
│ ▲ Earthbender                ✦ 12 skill points   ◆ strength 23           │
│ [ Skill Tree ] [ Loadout ] [ Looks ] [ Water ]            (tab buttons)  │
│ ┌────────────┐ ┌──────────────────────────────────────────────────────┐  │
│ │            │ │                                                      │  │
│ │  3D you    │ │                 (tab content)                        │  │
│ │  (turn by  │ │                                                      │  │
│ │  dragging) │ │                                                      │  │
│ │            │ │                                                      │  │
│ │ Stand Walk │ │                                                      │  │
│ │ Run  Cast  │ │                                                      │  │
│ └────────────┘ └──────────────────────────────────────────────────────┘  │
│                                   [ Done ]                                │
└──────────────────────────────────────────────────────────────────────────┘
```

The preview plays the element's idle; **Cast** plays the selected ability's gesture (`previewCast`) so Eilon sees the
move before unlocking it. The **Water** tab appears for waterbenders only (2.6).

#### 2.2a Skill Tree tab — עץ הכישורים

A **graph of cards** drawn on a scrollable canvas (drag to pan, wheel to zoom 0.75–1.25): the root at the top, tiers
below, **connectors** as 2-px lines in the element colour (lit when the child is unlockable, grey when locked, gold when
unlocked). Each node is a `BigButton` 76 × 40 with the ability's **icon** (6.2), name, and a **sub-line**: `Unlocked ·
M 4` or `3 SP` or `Needs Rock Throw M2`. Selecting a node fills the **detail panel** at the right (or below on narrow
screens):

```
┌──────────────────────────── Skill Tree ─────────────────────────────────┐
│                       ┌────────────┐                                     │
│                       │ ◉ Rock Throw│  M 4  ▮▮▮▮▯▯▯▯▯▯                   │
│                       └─────┬──────┘                                     │
│          ┌──────────────────┼──────────────────┬──────────────┐          │
│   ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐ ┌─────┴──────┐   │
│   │ ▌Earth Pillar│    │ ▆ Earth Wall│    │ ⛨ Earth Arm.│ │ ≋ Lava (6) │   │
│   │ Unlocked M2  │    │  3 SP       │    │  4 SP       │ │ Needs ES 20│   │
│   └─────────────┘    └──────┬──────┘    └─────────────┘ └────────────┘   │
│                      ┌──────┴──────┐                                     │
│                      │ ∴ Sand (3)  │   ┌─ Metal Bending (needs ES 30) ─┐ │
│                      └──────┬──────┘   │ ✕ Knives(5) ▦ Wall(5) ⚙ Wreck(6)│ │
│                      ┌──────┴──────┐   └───────────────────────────────┘ │
│                      │ ▽ Quicksand │                                      │
│                      └─────────────┘                                      │
├──────────────────────── Earth Wall ──────────────────────────────────────┤
│ Raise a wall of the ground in front of you. It blocks arrows and rocks   │
│ for 10 seconds. Sneak: Wall Push.                                        │
│ Mastery 1: 3 × 2 blocks    Mastery 10: 7 × 4 blocks, 2 thick             │
│ Cooldown 8 s   Range 3   Duration 10 s                                   │
│ [ Unlock: 3 SP ]      Mastery ▮▮▯▯▯▯▯▯▯▯ 2/10  [ +1 level: 2 SP ]  [ Cast ]│
└──────────────────────────────────────────────────────────────────────────┘
```

* **Unlock** is enabled when the parent is unlocked at the needed mastery and you have the SP (`BendingPayload.UNLOCK`).
  **+1 level** buys mastery up to 7 (`BUY_MASTERY`). **Cast** previews the gesture on the puppet. The mastery bar is 10
  cells (`▮▯` as Enhanced's `hexBarrier()` bar; green/yellow/grey by use). A **thin XP bar** under the cells shows the
  progress to the next level from use.
* Zaheer's two cards appear with a **red thread** icon when the path opens; before that a greyed card with "?" and the
  tooltip "A loss opens this path" (no spoiler of the pet rule unless `showZaheerHint` is on: default **off**).
* **Hebrew:** the tree is mirrored left-right, text right-aligned; connectors unchanged.

#### 2.2b Loadout tab — סידור הקרב

```
┌──────────────────────────── Loadout ────────────────────────────────────┐
│    Slot 1 [X]        Slot 2 [C]        Slot 3 [Alt]       Slot 4 [K]     │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐      ┌──────────┐      │
│  │ ◉ Rock   │      │ ▆ Earth  │      │ ▌Earth   │      │   empty  │      │
│  │   Throw  │      │   Wall   │      │   Pillar │      │  (drag   │      │
│  │ Sneak:   │      │ Sneak:   │      │ Sneak:   │      │   here)  │      │
│  │ Hold Rock│      │ Wall Push│      │ Launch   │      │          │      │
│  └──────────┘      └──────────┘      └──────────┘      └──────────┘      │
│                                                                           │
│  Unlocked abilities (drag onto a slot, or click a slot then an ability):  │
│  ┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐┌────────┐            │
│  │◉ Rock  ││▆ Wall  ││▌Pillar ││⛨ Armour││∴ Sand  ││▽ Quick.│   < 1/1 >  │
│  └────────┘└────────┘└────────┘└────────┘└────────┘└────────┘            │
│  [ Auto ]  [ Clear ]  [ Save as… ]  Presets: [Beginner] [Toph] [Bolin]    │
└──────────────────────────────────────────────────────────────────────────┘
```

* Four big `BigButton`s (the `KeysTab` cells, `stacked()`), each showing the ability's icon and name and its Sneak
  variant's small icon and name; a **cooldown shade** if it is cooling right now. The grid of unlocked abilities below is
  `EditorTab.cards("loadout", …, 62, 18)` with paging. **Drag and drop** (`mouseDragged` moving a ghost card) or
  click-click. **Auto** fills empty slots in tree order; **Clear** empties; **Save as…** stores up to 4 named presets
  per element (saved on the player, `TECH.md` 4); the element's built-in presets (`DESIGN.md` loadout examples) are
  one click. Changes send `BendingPayload.LOADOUT(slots)`.
* **Hebrew:** slots run right to left (slot 1 at the right); the key names stay as given.

#### 2.2c Looks tab — מראות

```
┌───────────────────────────── Looks ─────────────────────────────────────┐
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                      │
│  │ [face] Toph  │ │ [face] Bumi  │ │ [face] Lin   │   selected = gold    │
│  └──────────────┘ └──────────────┘ └──────────────┘                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐     │
│  │ [face] Suyin │ │ [face] Ghazan│ │ [face] Bolin │ │ [you] Own skin│     │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘     │
│                                                                           │
│  Toph Beifong — the greatest earthbender in the world. Bare feet.         │
│  Glider colour: ■■■■■■■■■■■■■■■■ (airbenders)   Water Skin: on belt [✓]   │
│                              [ Wear ]                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

* Face cards (`BigButton.face(skin)`, 20 × 20 faces at `flow`), the selected look drives the preview at once (the
  puppet's `PowerData`-equivalent `BenderData.look` is set locally). **Wear** sends `BendingPayload.LOOK(id)` (also
  Sneak + O cycles). A **draft** badge on Jae and Karthik until their pictures arrive. Airbenders see the **glider dye
  swatches** (16 `CreatorUi.Swatch`es; changing re-dyes the held glider: a crafting shortcut, `GLIDER_DYE` payload) and a
  **Jae board** toggle when Jae is worn (**suggestion**). Waterbenders: "Water Skin on belt" toggle (draw it or not).
* **Hebrew:** mirrored grid; the description right-aligned.

#### 2.2d Water tab — מים (waterbenders)

Shows the **water sources** in reach right now (a list: "River, 4 blocks", "Water Skin 6/8", "Plants (needs M7)"),
the Skin's charges as a bar, and two options: `preferSkin` (use the Skin before open water: default off) and
`showSourceLines` (draw a thin line to the current source in the world: default on).

### 2.3 Settings — הגדרות

Enhanced's generated config screens as they are, reached from the mod list's Config button (`ClientRegistrar.configScreen`)
and from a **Settings** button on the Bending screen's footer. Two files: `bending-client.toml` (sections `[hud]`,
`[effects]`, `[animation]`, `[sound]`) and `bending-server.toml` (every number: `TECH.md` 5), the latter editable only
in single player / LAN host (Enhanced's `not_online` tooltip).

Client options (the only ones not in the server file):

| Section | Option | Default | Meaning |
|---|---|---|---|
| `[hud]` | `position` | `BOTTOM_LEFT` | `BOTTOM_LEFT`, `BOTTOM_RIGHT`, `TOP_LEFT`, `TOP_RIGHT` |
| | `scale` | 1.0 | 0.75–1.5 |
| | `showMasteryBars` | true | |
| | `showElementStrength` | true | |
| | `moonIndicator` | true | the full-moon icon for waterbenders |
| | `sourceLines` | true | the thin line to the water source |
| `[effects]` | `density` | 1.0 | 0.25–1.0 multiplier on every particle count |
| | `screenEffects` | true | overlays (slit, vignettes, heat) |
| | `cameraShake` | true | (also off with reducedMotion) |
| | `fakeLight` | false | (suggestion) the warm disc under flames |
| `[animation]` | `animations` | true | the mod's poses (off: vanilla) |
| | `reducedMotion` | false | calm bob, no tremble, no shake, no camera yaw |
| | `firstPersonHands` | true | the hand motions |
| `[sound]` | `music` | true | the element themes |
| | `loopVolume` | 1.0 | the hold loops' master |

---

## 3. The HUD (`BendingHud`, a `LayeredDraw.Layer` registered `BELOW DEBUG_OVERLAY`, like Enhanced's `PowerHud`; the
overlays of `ANIMATION.md` 2.2 are separate layers `ABOVE CAMERA_OVERLAYS`)

**Placement.** Default **bottom-left**, so it never collides with Enhanced's top-left text panel when both mods run:
`x = 4`, `y = height − 4 − panelH`, above the vanilla left status bars when they would overlap (`HudHeights.left()` tells
how tall they are: the panel sits above `max(hotbar+bars, 0)` only when the position is bottom-centre; bottom-left is
clear of them since the bars start at the hotbar's left edge, `width/2 − 91`: on screens narrower than 2·(4+panelW+8)+182
the panel moves up by `HudHeights.left()`). Hidden when `hideGui`, the debug screen is open, the player is dead, or has no
element. The panel's width eases (Enhanced's 20-tick narrowing rule).

```
 ┌──────────────────────────────────────────────┐
 │ ▲ Earthbender · Toph        ✦ 12 SP  ◆ ES 23 │   <- header line (element colour)
 │ ┌────┐ ┌────┐ ┌────┐ ┌────┐                   │
 │ │ ◉  │ │ ▆  │ │ ▌  │ │    │   ⛨ Armour 18/25 │   <- 4 slot icons (24 px) + status column
 │ │ X  │ │ C  │ │Alt │ │ K  │   ◈ Metal ▮▮▮▯▯   │
 │ └────┘ └────┘ └────┘ └────┘                   │
 │  ▮▮▮▮▯    ▮▮▯▯▯    ▮▮▮▮▮    ·····             │   <- mastery bars (5 cells, scaled 10→5)
 │  ◇ Hold   ◇ Push   ◇ Launch  ·····            │   <- Sneak variant mini-icon + name (hover or Sneak held)
 └──────────────────────────────────────────────┘
```

* **Slot icon** (24 × 24): the ability icon (6.2); a **cooldown sweep**: a dark radial wedge (`GuiGraphics` triangles
  via a 24-segment fan, as `StoneWheelScreen.ring` draws fills) covering the remaining fraction clockwise, with the
  seconds (`hud.bending.seconds`, `%.1f`) in the corner; **ready flash**: a white 3-tick outline when it comes off
  cooldown (+ a soft `hud_ready` click, optional); **held**: an aqua outline while a hold ability runs; **on**: a green
  dot for toggles; **dimmed** when it cannot work now (no earth/water/metal in reach: the client runs the same reach
  checks as prediction) with a tiny reason glyph (✗ earth / ✗ water / ✗ metal / ✗ moon); the **key letter** under it in
  yellow (`getTranslatedKeyMessage()`). When **Sneak is held**, the icons **swap** to the Sneak variants (the small
  diamond row becomes big and vice versa) so a kid always sees what the key will do.
* **Header**: the element symbol and "Earthbender · Toph" (element name key `element.bending.<id>.bender`, the look's
  name after a middle dot), **SP** (`✦ 12 SP`, white; flashes gold and counts up when earned: a `+2` rises and fades for 20
  ticks), **ES** (`◆ ES 23`, grey; hidden if `showElementStrength` is off).
* **Status column** (right of the icons, one line per active state, Enhanced's `hudStatus` style): the **Earth Armour /
  Steel Armour shell** (`⛨ 18/25` with a 5-cell bar, green/yellow/red), **Metal charge** (`◈ ▮▮▮▯▯ 12/30`), **Air Shield**
  strength (`◎ 9/13`), **Scooter / Spout time** (`⟲ 42 s`), **Spiritual Projection** (`☽ 48 s · body 64 m`), **Water
  supply** (3.3), **Blood** (`✶ 4.2 s` hold left), **Lightning** (`⚡ charging…` / `⚡ redirect ready`), **Zaheer's
  path** (a small broken-thread glyph in the header), **Severed** (`⊘ bending gone 1:52`, red), **Paralysed** (`⚡ 1.6 s`).
* **Mastery bars**: 5 cells per slot (level/2 rounded), tooltip-less; `showMasteryBars`.
* **Hebrew:** the panel's lines go through `HudDraw.visual`; the header reads `✦ 12 נק׳ כישור  ◆ חוזק 23 · טוף · כשף
  אדמה ▲` right-aligned; the icons stay in slot order **right to left** when the position is a right corner, left to
  right in a left corner (icons are not text; the key letters under them say which is which).

### 3.1 Earth: nothing more. **3.2 Air:** the glider's state (`⌇ glider open`) when open.

### 3.3 Water supply — אספקת מים (waterbenders only)

```
 ┌───────────────────────────────────────────────┐
 │ ≈ Waterbender · Katara       ✦ 8 SP  ◆ ES 14  │
 │ ┌────┐ ┌────┐ ┌────┐ ┌────┐   ~ River 4 m     │   <- the current source
 │ │ ≈  │ │ ✧  │ │ ❄  │ │ ⌁  │   ◍ Skin ▮▮▮▮▮▮▯▯ │   <- the water skin's charges
 │ │ X  │ │ C  │ │Alt │ │ K  │   ☾ full moon      │   <- 3.6
 │ └────┘ └────┘ └────┘ └────┘                   │
 └───────────────────────────────────────────────┘
```

* **Source line**: `~ River 4 m` / `~ Rain` / `◍ Water Skin` / `❀ Plants` / **`✗ No water`** (red). The `sourceLines`
  option draws a thin blue line in the world from the hand to the source block (an `ENERGY_VEIL` ribbon α 0.2).
* **Skin bar**: 8 cells (`▮▯`, blue), the fullest skin; empty: red; refilling in rain: a drip glyph.

### 3.4 Spiritual projection mode — מצב השלכה רוחנית

While the spirit is out, the HUD **changes**: the slot row is replaced by `☽ Spirit · 48 s left · body 64 m → ↑`
(the time, the distance and an arrow toward the body; range ring: a thin arc fills as you near the limit), the overlay
vignette (`ANIMATION.md` 2.2), and a hint `[X] return` (the slot key of the ability returns you). Damage to the body shows
a red flash on the panel.

### 3.5 Severed — נשללו הכשפות

Every icon greys (`0x60000000` fill over them), the header turns grey, the status line `⊘ Your bending is gone · 1:52`
counts down; nothing else changes.

### 3.6 The full-moon indicator — מחוון הירח המלא (waterbenders)

A **moon glyph** in the status column: `☾` grey on ordinary nights, **`●` white** on a full-moon night (phase 0, night)
with "full moon" / "ירח מלא" and a faint glow; by day the phase of the coming night (`☽ tonight: full` from 10000 ticks
on). For a blood bender below M10 the Blood slot icon is dimmed with the ✗ moon glyph until the full moon. The icon is
`moon_full.png` (16 × 16).

### 3.7 Other players

Above another bender's head, no HUD. Their looks, stances and effects say everything (the design's rule, as Enhanced).

---

## 4. Action-bar messages (the `message.bending.*` keys, both languages)

`no_earth` "No earth here" / "אין פה אדמה"; `no_sand`, `no_metal` "No metal" / "אין מתכת"; `no_water` "No water" /
"אין מים"; `protected` "Protected ground" / "קרקע מוגנת"; `stronger_earthbender` "A stronger earthbender" / "כשף אדמה
חזק יותר"; `stronger_waterbender`; `no_moon` "Only under a full moon" / "רק בירח מלא"; `in_water` "Not in water" / "לא
בתוך מים"; `empty_slot` "Empty slot: press O" / "משבצת ריקה: לחצו O"; `severed` "Your bending is gone" / "הכשפות שלכם
נשללו"; `mastery_up` "%1$s: mastery %2$s" / "%1$s: רמת שליטה %2$s"; `points` "+%s skill points" / "+%s נקודות כישור";
`unlocked` "%s unlocked!" / "%s נפתחה!"; `zaheer_path` "You have cut your earthly tethers" / "ניתקתם את הקשרים
הארציים שלכם"; `immune` "Immune" / "חסין".

---

## 5. Hebrew copy rules (for every lane)

* Natural Hebrew, never transliterated English words where a Hebrew word exists; the dub's terms: כשף/כשפת, כשפות אדמה /
  אוויר / מים / אש, כשפות דם, כשפות ברק, כשפות מתכת, כשפות לבה, כשפות קרח, כשפות חול (the last three are the mod's,
  in the dub's pattern).
* Second person plural for instructions ("לחצו", "בחרו"), as Enhanced's wiki; ability names are nouns (זריקת סלע).
* Sneak = התכופפות, hold = החזקה, on/off = פועל/כבוי, seconds = `%s ש׳`, mastery = רמת שליטה, skill points = נקודות
  כישור, element strength = חוזק האלמנט, loadout = סידור הקרב, skill tree = עץ הכישורים, look = מראה.
* Character names in Hebrew as Eilon spells them (ג'יי, קארתיק) and the common Hebrew spellings for the rest
  (`LOOKS.md` headings).
* `DataTests` rules as Enhanced: identical key sets, identical `%` placeholders, Hebrew letters present, no double
  spaces.

---

## 6. Icons (16 × 16 for the HUD at 24 px it is drawn scaled; 32 × 32 masters, `textures/gui/icons/`)

### 6.1 Elements and states

| Icon | Name | Drawing |
|---|---|---|
| `earth` | ▲ | a stepped mountain in ochre with a moss-green base |
| `air` | ◎ | three nested wind swirls in white on sky blue |
| `water` | ≈ | two waves, sea blue with a white crest |
| `fire` | ♦ | a three-tongued flame, orange with a gold core (Azula: a blue variant `fire_blue`) |
| `sp` | ✦ | a four-point gold star |
| `es` | ◆ | a grey diamond |
| `moon_full`, `moon` | ● / ☾ | a white disc with craters; a grey crescent |
| `thread_cut` | ⌇ | a red thread snapped |
| `severed` | ⊘ | a grey circle with a bar |
| `paralysed` | ⚡ | a white-blue bolt |
| `no_source` glyphs | ✗ | a small red cross with the source's glyph |

### 6.2 Abilities (one per card; the Sneak variant reuses the icon with a small ◇ badge)

| Earth | Air | Water | Fire |
|---|---|---|---|
| `rock_throw` a flying rock with a dust tail | `air_blast` a palm with wind lines | `water_shot` a water stream | `fireball` a fireball |
| `earth_armour` a stone chest plate with an eye slit | `air_shield` a wind sphere | `healing` glowing hands over a heart | `fire_punch` a burning fist |
| `earth_pillar` a column rising | `air_glider` the open glider (item) | `water_octopus` a figure with 8 arms | `flamethrower` two jets |
| `earth_wall` a wall of three blocks | `air_blades` a crescent | `water_spout` a vortex column | `fire_whip` a curling flame rope |
| `sand_bending` a sand swirl | `spiritual_projection` a seated figure with a blue spirit | `water_arms` two long water arms | `fire_circle` a ring of flame |
| `quicksand` a sinking figure | `air_bubble` a head in a bubble | `ice_knives` three shards | `fire_healing` gold hands over a heart |
| `lava_bending` a lava path with a shuriken | `air_scooter` a figure on a ball | `freeze` a frozen footprint | `lightning` a forked bolt |
| `steel_knives` three knives | `tornado` a funnel | `ice_wall` an ice wall | `lightning_redirection` a bolt bent in a U |
| `steel_wall` riveted plates | `true_flight` a floating figure (red thread) | `blood_bending` a red claw hand with threads | |
| `wreck_metal` a bent hopper | `suffocation` a hand over a gasping mouth (red thread) | `bending_severance` a thumb on a brow | |

Sneak variants with their own art where the badge is not enough: `hold_rock`, `shed_blast`, `earth_launch`, `wall_push`,
`sand_wall`, `sand_cloud`, `lava_shuriken`, `metal_pull`, `steel_armour`, `metal_grip`, `air_jump`, `shield_burst`,
`air_slice`, `deep_meditation`, `air_cushion`, `air_spout`, `air_wheel`, `hover`, `air_vacuum`, `water_whip`, `cleanse`,
`octopus_lash`, `wave_ride`, `arm_grab`, `ice_spikes`, `snow_throw`, `ice_dome`, `blood_fling`, `small_fireball`,
`fire_kick`, `fire_breath`, `whip_lash`, `fire_jet`, `burn_out`, `lightning_arc`, `ground_it`: **38** small icons.

### 6.3 Items and blocks

`element_scroll`, `water_skin` (9 fill stages), `air_glider` (16 dyes, one base tinted), `bending_scroll_<element>`
(suggestion), `training_dummy` (suggestion), `quicksand`, `steel_plate`, `ice_spike`, `wrecked_*` (8), `temp_lava`
(vanilla lava), the advancement icons (one per element: the symbol in a frame).

### 6.4 Where icons are drawn

Icons are `GuiGraphics.blit` sprites from one **atlas** `textures/gui/bending_icons.png` (256 × 256: 16 × 16 cells,
addressed by an `Icon` enum with u, v), so the HUD costs one texture bind.

---

## 7. Tests for the UI (server-side, as Enhanced's client tests: no client class in a signature)

`HudTests`: the panel's lines for a given `BenderData` (keys present, cooldown formatting, the Hebrew seconds line keeps
the unit next to the number through `java.text.Bidi`), the source line for each source, the moon line for each phase and
time; `ScreenDataTests`: every ability has an icon cell, a name and a description in both languages, every look a face;
`LoadoutTests`: slot payload round trip, presets, auto-fill order; `TreeTests`: unlock rules (parent, mastery, SP, ES
gates), buying mastery caps at 7. What they look like: a real client, English and Hebrew, three render rounds per lane,
screenshots per screen (`TECH.md` 8).
