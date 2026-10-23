# Bending — Animation and Effects Bible

This is the build reference for every movement and every effect of the mod. It is written against **Enhanced's
animation system**, which Bending copies into `com.kolodkingal.bending.client.render` (see `TECH.md` 2 for the exact
class list): gestures (`GestureId` + `Gestures.define` keyframes), stances (`Stance` + `Stances` rows), flight and run
profiles, `FirstPersonArms.Offset`, `HeroAnimator` blending, `GestureEffects` flourishes, `HandTracker`, and the FX
toolkit (`EnergyGeometry`, `FxShapes`, `SandShapes`, `EnhancedRenderTypes`, `ShieldFx`, `BeamRenderer`, `EnergyParticle`).
Where a building block does not exist in Enhanced, section 14 lists it as **NEW** and names the lane that builds it.

## 0. How to read this document

### 0.1 Units and signs (Enhanced's conventions, unchanged)

* **Ticks.** 20 per second. A keyframe's time is the tick it is reached.
* **Angles are degrees**, written for a **right-handed** player; the pipeline mirrors a left-handed one. `M` is the
  **main** arm (right), `O` the **off** arm (left). Arm angles are `(x, y, z)`:
  * `x`: negative raises the arm **forward and up** (−90 is level forward, −176 is straight overhead); positive swings it
    **back**. 0 hangs at the side.
  * `y`: turns the arm about its own axis / inward-outward in the horizontal plane (for `M`, +y swings the hand across
    the body).
  * `z`: for `M`, **+z spreads the arm outward** (away from the body); for `O`, **−z** spreads outward. So a symmetric
    pose is written `m(x, y, z)` and `o(x, −y, −z)`.
* **Legs** in gesture keys drive **x only** (`legs(right, left)`: negative lifts the leg forward, positive back). Leg
  **z** (spread) is available in **stance rows** (`c.leg(right, x, z, w)`), and Bending adds it to gesture keys (**NEW**,
  14.1), so a horse stance and a kick can be keyed.
* **Head** `(x, y, z)` is **additive** on top of the look (x down positive).
* **Whole body:** `lean` (degrees, forward positive; a pose-stack pitch about a pivot 0.40 blocks up on the ground),
  `roll` (bank), `yaw` (a visual turn; must start at 0 and end on a multiple of 360), `drop` (blocks the body sinks;
  positive is down), `twist` (shoulder twist in degrees: becomes `body.yRot`).
* **Ease codes** (the curve of the segment *arriving* at the key): `S` smoothstep (default), `O` ease-out cubic (snap
  then settle), `I` ease-in cubic (accelerate into an impact), `L` linear, `E` sine in-out, `B` ease-out-back (10%
  overshoot). The gesture's **weight envelope** is `envelope(t, attack, dur, release)`: cubic rise over `attack`, hold,
  sine fall over `release`.
* **Modes:** a plain `m(x,y,z)` is ABS (lerped toward by the weight); `mAdd` is ADD (added to what is below); `mWorld(phi,
  y, z)` holds the arm at a **world elevation** (0 hangs, 90 level, 180 up) regardless of body lean; `mAim(Target, dx,
  dy, dz)` points the arm at a target (`LOOK`, `HELD`, `PROJECTILE`, `ANCHOR`, `STRIKE`...) plus an offset.
* **Tremble:** `mTremble(amplitude°, speed rad/tick)` on a key; off under *Reduced motion*.
* **Groups:** at most one gesture per group plays; a new one fades the old over 3 ticks. Bending's groups: `CAST`
  (every ability), `TAKEOFF`, `LAND`, `BRAKE`, `RUN`, `RIDE` (**NEW**: mount/dismount of scooter and spout), `HIT`
  (**NEW**: hit reactions, so they never cancel a cast).
* **`force`**: the gesture replaces the vanilla punch on the arms it holds. Every attack gesture below is `force = true`;
  stances, rides and reactions are not.

### 0.2 Keyframe tables

Each ability has a keyframe table in this shape:

| t | ease | M (x,y,z) | O (x,y,z) | legs R/L x (z) | head | body (lean / twist / drop / yaw) | notes |

An empty cell means the channel is **not keyed** at that time (the layer below shows through: walking legs keep
walking). A dash in a whole column means the channel is never driven by this gesture. `REST` means the arm's rest
pose of the current stance (`EARTH_REST_M` etc., 1.2). After the table: **timing** (`dur / attack / release`, group,
force), **lock** (does it stop movement? how: a `MovementInputUpdateEvent` lock bound to a `Live` bit, never the gesture
itself), **blend** (what it does over walking, flying, swimming, sneaking), **started by** (the `GestureRoutes` arm:
the ability's cooldown edge, a `Live` bit, a toggle flip, or an FX payload kind), **prediction** (the client's own key
press starts it at once when the client can tell it will work).

### 0.3 First person

Hand motion is `FirstPersonArms.Offset` per hand: `(dx, dy, dz, rx, ry, rz)` in view space (blocks: +x right, +y up,
+z away from the camera; degrees) in right-hand signs; the left hand mirrors `dx`, `ry`, `rz`. The empty off hand
slides in when used (`used > 0.01`). Offsets are capped at 0.8 blocks. **Camera** effects go through
`ViewportEvent.ComputeCameraAngles` (shake: pitch/roll sine with distance falloff and the reduced-motion gate) and
`ComputeFovModifierEvent` (FOV). **Overlays** are `LayeredDraw.Layer`s above `CAMERA_OVERLAYS`. All of this exists in
Enhanced and is reused as is.

### 0.4 Effects

Particles are Enhanced's `EnergyParticle` (`bending:energy`, colour + scale, fullbright, 10–21 ticks, friction 0.9) plus
vanilla types (`BLOCK` dust of a state, `DUST`, `SPLASH`, `FLAME`, `SMALL_FLAME`, `SOUL_FIRE_FLAME` for blue fire,
`ELECTRIC_SPARK`, `CLOUD`, `SNOWFLAKE`, `ITEM_SNOWBALL`, `FALLING_WATER`, `DRIPPING_WATER`, `BUBBLE`, `POOF`, `CAMPFIRE_COSY_SMOKE`,
`LAVA`, `DRIPPING_LAVA`, `ASH`, `WHITE_ASH`, `GLOW`, `END_ROD`, `SONIC_BOOM`, `GUST`, `SMALL_GUST`, `TRIAL_SPAWNER_DETECTION`),
and **two new particle types** (**NEW**, 14.5): `bending:grain` (a tinted sand/earth grain with gravity, 20–40 ticks) and
`bending:droplet` (a water bead with gravity and a splash on landing). Meshes are drawn in `RenderLevelStageEvent.AFTER_PARTICLES`
into the copied render types: `ENERGY_BEAM` (additive, for fire, lightning, glows), `ENERGY_VEIL` (translucent, for air
shapes and water bodies), `MATTER_FLAT` and `MATTER_TEXTURED` (Enhanced's `SandRenderTypes.FLAT` / `textured`: normal
alpha, for earth, sand, water and ice surfaces), `energyField(tex)` (textured additive lattice: the air shield's
lines). Rule from Enhanced: **matter is not light**: earth, sand, water and ice use normal alpha; fire, lightning and
spirit glows use additive.

**Budgets.** Every effect row gives *particles per tick (or per event)* at density 1.0. The client option *Effect
density* (0.25–1.0, default 1.0; `bending-client.toml [effects] density`) multiplies every count; vanilla's Particles
option (All / Decreased / Minimal = 1.0 / 0.5 / 0.15) multiplies again, as Enhanced's `SandFx.density()` does. Nothing
is spawned beyond **48 blocks** from the camera, meshes are not drawn beyond **96**, and each player's effects are
capped (per-player caps in the rows). The design target: **4 benders fighting at once stay under ~600 live particles
and ~40 mesh draws per frame.**

**Lighting.** No dynamic world light exists in Enhanced and none is added (no light-block tricks: they would be temp
blocks with light updates, the one thing that lags). Fire, lightning and spirit forms are drawn **fullbright and
additive**, which reads as emissive; fire casts no light on the ground. **(suggestion, 14.7):** a client-only "fake
light" pass is possible later (a translucent warm disc under a flame, `MATTER_FLAT`), cheap and loader-free.

### 0.5 Sound cues

Each ability lists **wind-up / release / impact (or loop)** cues by event name; the synthesis recipe and both subtitles
are in `AUDIO.md`. Loops follow the player through the copied `PowerLoopSounds` (a `Kind` per loop with a predicate on
`BenderData`).

---

## 1. The four styles, as poses

### 1.1 What the shows do, in Minecraft's joints

| Element | Martial art | Body | Arms | Legs | Rhythm |
|---|---|---|---|---|---|
| **Earth** | Hung Gar | upright, heavy, **low**: horse stance (`drop` 0.18–0.32), lean forward 4–8°, shoulders square | fists and **open palms pushed from the hip**, elbows bent, slow wind-up then a hard stop (`I` into the hit, `O` out) | **wide**: legs spread z ±14, stomps (a leg lifts to x −35 then slams, `drop` dips 0.1) | wait, root, strike: long attack ticks, short release |
| **Air** | Ba Gua Zhang | light, upright, often **turning** (`yaw` 90–360), lean with the circle, `drop` 0 | **open palms**, arms drawing **circles** (y and z sweep through 60–120° over 10–16 ticks), one arm leads, the other follows a quarter turn behind | **stepping**, never planted: legs x alternate ±18 as the body turns (walk cycle keeps running under most gestures) | continuous: `S` and `E` easings, weights overlap, no hard stops except the blast's release |
| **Water** | Tai Chi | upright, **soft knees** (`drop` 0.06–0.10), slow weight shifts (`lean` ±4 alternating) | **flowing**: both arms move together in **S-curves**, wrists lead (y leads x by 2 ticks), palms turn over (y sweeps), pulls and pushes | shoulder-width, one leg forward (x −12 / +12), weight shifts between them | slow in, quick out: a 12-tick gather, a 4-tick release; `E` everywhere, `O` on the release |
| **Fire** | Northern Shaolin | upright, **long** bow stance (front leg x −30, back leg x +25, `lean` 10), chest forward | **straight punches** and chops: arms snap to level forward (x −90) in 2–3 ticks with `O`, pulled back with `I`; fists | **kicks** (leg x −80 to −110, z ±20) and stamps | explosive: attack 1–2 ticks, hold 2, release 4–6; the **breath**: a 6-tick inhale (chest up: head −6, lean −3) before big moves |
| **Blood** | (Hama / Amon) | hunched, lean 12–18°, head down 6, shoulders in (twist ±6 alternating) | **claw hands**: arms forward and up at −70, y ±25 inward, z toward the centre, **fingers** implied by a 3° tremble at 2.2 rad/tick and small y pulses; one hand pulls, the other pushes | feet planted, knees bent (`drop` 0.12) | slow, trembling, the puppet's limbs follow the hands by 2 ticks |
| **Lightning** | Iroh's form | upright and still, lean 0, breath in | the **two-finger** path: both arms draw **slow circles** (14 ticks) that **meet at the stomach** (x −40, y ±40, z toward the centre), then the leading arm **extends** to level (x −90, y 0) with the other braced at the hip; redirection runs the path **in reverse** through the stomach and out the other arm | bow stance, planted | 24-tick wind-up, 2-tick release |

### 1.2 Pose library (the `StancePoses` of Bending)

These are the constants every table below refers to (degrees, right-handed; `_O` is the mirror unless given).

| Name | M (x,y,z) | O (x,y,z) | legs R/L x (z) | head | lean / drop | Meaning |
|---|---|---|---|---|---|---|
| `NEUTRAL` | 0,0,8 | 0,0,−8 | 0/0 (±2) | 0 | 0 / 0 | vanilla rest |
| `EARTH_REST` | 8,0,14 | 8,0,−14 | 0/0 (±6) | 0 | 2 / 0.04 | heavy shoulders |
| `EARTH_HORSE` | −38,12,40 | −38,−12,−40 | 0/0 (±16) | 2 | 4 / 0.24 | horse stance, fists at the hips' height, forearms level |
| `EARTH_PUSH` | −88,−4,6 | −60,10,−30 | 0/0 (±14) | 0 | 8 / 0.18 | palm thrust, rear arm chambered |
| `EARTH_LIFT` | −120,0,30 | −120,0,−30 | 0/0 (±12) | −6 | −6 / 0.10 | both palms up as the rock rises |
| `EARTH_CLAW` | −72,18,10 | 20,0,−20 | 0/0 (±10) | 0 | 4 / 0.12 | the holding hand, fingers down |
| `EARTH_STOMP_UP` | −30,0,30 | −30,0,−30 | −36/0 | −4 | −4 / 0 | the right leg raised |
| `EARTH_STOMP_DOWN` | −20,0,34 | −20,0,−34 | 0/0 (±16) | 6 | 6 / 0.30 | the slam |
| `EARTH_GUARD` | −100,30,20 | −100,−30,−20 | 0/0 (±14) | 0 | 2 / 0.20 | forearms crossed before the chest (the wall rises) |
| `AIR_REST` | −10,0,10 | −10,0,−10 | 0/0 (±3) | −2 | 0 / 0 | light, palms open |
| `AIR_PALMS` | −70,−30,−10 | −70,30,10 | — | 0 | 0 / 0 | both palms forward, close together |
| `AIR_CIRCLE_A` | −120,−40,40 | −30,20,−10 | −10/10 | 0,−10,0 | −2 / 0 | a quarter of the circle: leading hand high |
| `AIR_CIRCLE_B` | −60,40,−30 | −100,−40,30 | 10/−10 | 0,10,0 | 2 / 0 | the opposite quarter |
| `AIR_GUARD` | −60,−20,−24 | −60,20,24 | 0/0 (±6) | 0 | 0 / 0.02 | palms up and out: the shield |
| `AIR_SIT` | −20,30,−40 | −20,−30,40 | −90/−90 (±55) | 4 | 0 / 0.72 | meditation: cross-legged, hands on the knees (**NEW** leg z in keys) |
| `WATER_REST` | 4,6,10 | 4,−6,−10 | −6/6 | 0 | 0 / 0.06 | soft knees, one foot forward |
| `WATER_GATHER` | −40,30,−20 | −40,−30,20 | −12/12 | 2 | −3 / 0.10 | both hands drawn in at the belly, palms up |
| `WATER_PUSH` | −86,10,0 | −86,−10,0 | −14/16 | 0 | 6 / 0.08 | both palms out, level |
| `WATER_WAVE` | −110,−30,30 | −50,30,−20 | −10/10 | 0 | 2 / 0.08 | "wave hands like clouds": high hand leads |
| `WATER_WHIP` | −70,40,−10 | −30,−20,−20 | −10/10 | 0 | 4 / 0.08 | one arm cocked, wrist leading |
| `WATER_HEAL` | −60,20,−10 | −60,−20,10 | −6/6 | 6 | 2 / 0.08 | palms down, hovering |
| `FIRE_REST` | 6,0,12 | 6,0,−12 | −8/8 (±6) | 0 | 2 / 0.02 | ready, fists loose |
| `FIRE_BOW` | −30,0,34 | −30,0,−34 | −30/25 (±8) | 0 | 10 / 0.14 | bow stance, fists chambered at the hips |
| `FIRE_PUNCH` | −90,0,0 | −10,0,−30 | −30/25 (±8) | 0 | 12 / 0.12 | straight punch, rear fist chambered |
| `FIRE_DOUBLE` | −90,−6,4 | −90,6,−4 | −30/25 (±8) | 0 | 12 / 0.12 | both fists forward (flamethrower) |
| `FIRE_BREATH` | −40,0,50 | −40,0,−50 | −20/20 (±10) | −14 | −8 / 0.04 | arms back and wide, head back, chest open |
| `FIRE_KICK` | −60,0,40 | 40,0,−20 | −105/15 (+22/0) | 0 | −10 / 0.02 | right leg at head height (**NEW** leg z) |
| `LIGHTNING_MEET` | −40,38,−24 | −40,−38,24 | −24/22 (±8) | 4 | 4 / 0.10 | fingers meeting at the stomach |
| `LIGHTNING_POINT` | −92,0,2 | −20,10,−16 | −30/25 (±8) | 0 | 6 / 0.10 | the two fingers extended |
| `REDIRECT_IN` | −92,0,2 | −30,0,−30 | −24/22 (±8) | 0 | 2 / 0.12 | one arm receives (level), the other low |
| `REDIRECT_OUT` | −30,0,30 | −92,0,−2 | −24/22 (±8) | 0 | 2 / 0.12 | the bolt leaves the other arm |
| `BLOOD_CLAW` | −72,26,−6 | −66,−22,8 | 0/0 (±8) | 6 | 14 / 0.12 | both claws forward, hunched |
| `BLOOD_HIGH` | −120,10,−10 | −60,−24,10 | 0/0 (±8) | 2 | 10 / 0.10 | one claw raised (the puppet lifts) |
| `PUPPET` | −100,−40,−60 | −100,40,60 | −20/30 (+25/−25) | −10,20,0 | −10 / −0.4 (lifted) | the victim: arms wrenched up and back, legs apart, head back |
| `PARALYSED` | −20,0,46 | −20,0,−46 | 0/0 (±14) | −8 | −6 / 0 | rigid, arms out, 2° tremble at 4.0 |

---

## 2. Shared first-person and screen rules

### 2.1 Hands

Every bender's first person follows Enhanced's `FirstPersonArms`: the gesture's hand motion in view space, the ambient
stance motion, and the element's **idle hands** when a slot key with a hold or ready stance is held (section 9). Blue
fire, water ribbons, rock chips and sparks are spawned at the drawn hand position (`FirstPersonArms.mainHand(player,
out)`), as Enhanced's beams are, so a Water Whip really hangs from your hand.

### 2.2 Overlays (`LayeredDraw.Layer`s above `CAMERA_OVERLAYS`)

| Overlay | When | What is drawn | Fade |
|---|---|---|---|
| **Earth Armour eye slit** | Earth or Steel Armour on, first person | a stone (or steel) frame covering the screen except a horizontal slit 22% of the height centred at 42% from the top, with rough inner edges (a 32-segment jagged polyline, `MATTER_FLAT`, dark ochre `#3A2E1C` at α 0.92; steel: `#2B2F36`); cracks (white lines) grow as the shell's health falls below 50%, 30%, 15%; the slit widens by 6% at Earth Armour M8+ | forms over 20 ticks from the edges inward (the armour's forming time); crumbles out in 6 ticks with chips |
| **Spiritual projection** | the spirit is out | a cool blue vignette (28 nested outlines as Enhanced's `PhoenixOverlay`, `#7FD0FF` α from 0.35 at the edge to 0), a faint waving ribbon border (sine 0.6 Hz), saturation ×0.7 and a blue tint via a post shader `bending:shaders/post/spirit.json` (vanilla `color_convolve`: Saturation 0.7, ColorScale 0.85/0.95/1.2), a thin **tether line** at the bottom centre with the distance to the body ("64 m") and a compass tick toward it; through-wall glows (`EntityGlow`: hostile mobs light blue, players white, villagers green) | in over 10 ticks at departure, out over 6 at return (with a white flash 0.3 at the snap-back) |
| **Blood-bending tint** | a Blood Bending hold is active (caster) | a red vignette (`#7A0E1E` α 0.30 at the edges), pulsing with a slow heartbeat (0.9 Hz: two beats), the moon drawn large and pale at the top-left corner behind the HUD | in 8 ticks, out 12 |
| **Being blood-bent** (victim) | `Live.BLOODBENT` on you | a stronger red vignette (α 0.5), the view **tilts** ±3° with the puppet's sway (camera roll), a heartbeat at 1.4 Hz | out over 20 ticks when freed |
| **Full-moon glow** | a waterbender outdoors on a full-moon night | a soft silver sheen (`fillGradient` top third `#E8F0FF` α 0.08 → 0) and the HUD moon icon glows (`UI.md` 3.6) | — |
| **Paralysed** | `bending:paralysed` on you | a white-blue flash 0.4 then a static-noise tint: 6 horizontal jittering bars (random y each 2 ticks, α 0.15) for the duration | out at once |
| **Quicksand** | you stand in quicksand | the bottom 30% of the screen is a sand gradient (`#D8B76A` α 0.5 → 0) that rises as you sink | — |
| **Underwater bubble** | Air Bubble on, under water | the water fog is removed (`ViewportEvent.RenderFog` set far plane 128, cancel) and a thin circular bubble rim is drawn (a ring at the screen's edge, α 0.25, with 6 slow highlights) | — |
| **Severed** | bending severed | every HUD icon greys (`UI.md` 3.5); no vignette (the loss is quiet) | — |
| **Fire Breath** | breathing | a heat ripple: the bottom half of the screen's hue shifts +3° every 2 ticks (a cheap wobble via two alternating `fillGradient`s at α 0.06) | — |

### 2.3 Camera

* **Shake** (never with *Reduced motion*): Earth stomps and landings (`strength 0.6`, 8 ticks, within 12 blocks),
  boulder impacts (0.8, 10, 16), Earth/Steel Wall rising (0.3, 6, 8), Shed Blast (0.7, 8, 12), Lava Path (0.2, 4, 6),
  Air Shield burst (0.5, 8, 10), a Tornado within 6 blocks (0.15 continuous), Water Spout ram (0.4, 6, 8), Lightning
  hit (0.9, 6, 20: a sharp high-frequency jolt: pitch `sin(5.1·age)`), Fire Circle (0.3, 6, 8), Shatter of your own
  armour (0.5, 8). Formula as Enhanced's `ArenaFx.shake`: `pitch += s·fade·sin(2.3·age)`, `roll += 0.6·s·fade·sin(1.77·age+1)`.
* **FOV**: Air Scooter and Air Wheel ×1.12 (eases in 6 ticks); True Flight at full speed ×1.08; Water Spout ride ×1.05;
  Fire Jet ×1.1; Lightning charge **−** ×0.96 (a slight pull-in over the 24 ticks, then snaps to 1.0 on release);
  Spiritual projection ×1.05 constant.
* **Roll**: Air Scooter banks the camera 4° into turns (lerp with `turn`), glider 6°; True Flight 5°.

---

## 3. Earth

### 3.0 Earth idle: the ready stance (held slot key, or the element's `IDLE`)

| Stance | tauIn / tauOut | Rows |
|---|---|---|
| `EARTH_IDLE` (ambient, on while a bending key is held with nothing to fire, or 40 ticks after any Earth gesture) | 5 / 6 | `EARTH_HORSE` at weight w: arms lerp to the horse, legs spread z ±16·w, `drop` 0.24·w, lean 4·w; a slow **breath**: arms x ±2.5 at 0.09 rad/tick; every 60 ticks a tiny weight shift (lean alternates ±1.5 over 20 ticks). Head looks along the camera (the shared head rule). |
| `EARTH_GRIP` (aimed, while `Live.EARTH_HOLDING`) | 2 / 4 | `EARTH_CLAW`: M aims at the held rock (`aim(HELD, −20, −6, 0)`) with tremble 2.4 at 1.9; O braced back; lean 4; `occupy(M)`. |
| `EARTH_ARMOURED` (ambient, while Earth/Steel Armour is on) | 6 / 6 | arms +10° x heavier swing, z ±4 wider, `drop` 0.03; the run character `EARTH_ARMOUR` (0.9 arms, 0.8 legs, lean 6, bob 0.035: a stomping walk). |

**Ambient particles of a master (Earth ES ≥ 60):** 1 `bending:grain` (the ground's colour) every 10 ticks rising 0.3
blocks around the feet and settling; at ES 100 a faint ochre dust ring (radius 1.2, `MATTER_FLAT`, α 0.08) under the feet
while standing still. Budget: ≤ 2 particles/s per master.

**Earth first-person idle (key held):** both hands low and wide, palms down (`M dx +0.10 dy −0.14 dz +0.05 rx +20`, O
mirrored), breathing ±0.01 in dy.

**Earth sound bed:** none while idle; a master's ambient grain has no sound.

### 3.1 Rock Throw — זריקת סלע

**Third person, `EARTH_ROCK_THROW`** (dur 16 / attack 2 / release 6, group CAST, force):

| t | ease | M | O | legs R/L (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | 0 | lean 2, drop 0.04 | |
| 3 | O | −40,0,36 | −40,0,−36 | −34 / 0 | −6 | lean −6, drop 0 | the right leg lifts: the stomp begins |
| 5 | I | −118,0,28 | −118,0,−28 | 0 / 0 (±16) | −4 | lean 2, **drop 0.30** | **stomp**: the foot slams, the rock jumps out of the ground (particles 3.1e), both palms rise under it |
| 9 | S | −120,−6,24 | −30,10,−26 | 0 / 0 (±16) | 0 | lean 0, drop 0.22 | the rock floats at chest height (tick 5–9: 4 ticks of float, the "hold" of Hung Gar) |
| 11 | I | **−92,−8,4** | −10,0,−30 | 0 / 0 (±14) | 2 | **lean 10**, drop 0.16, twist 12 | **the push**: the palm drives forward from the hip, the body turns in; the rock leaves at tick 11 (the server's launch tick) |
| 16 | S | `EARTH_REST` | `EARTH_REST` | (±6) | 0 | lean 2, drop 0.04, twist 0 | recovery |

Timing notes: the server lifts the rock at the press (tick 0) and holds it 11 ticks before launching, so the look
matches: the rock entity is spawned "rising" at tick 0 with a 5-tick rise (0.0 → 1.2 blocks) and a 6-tick float, then
flies. **Lock:** none (you may walk; the legs show the walk except for the stomp key at tick 3–5, which drives the right
leg only). **Blend:** over walking the stomp becomes a step; over flying (Creative) the stomp is skipped (legs not
keyed when `wAir > 0.5`: the `legs` channel is masked by `1 − wAir`, **NEW** 14.3 channel masks); never under water
(refused). **Started by:** the slot's cooldown edge (`cooldownGesture`: Rock Throw's configured 30 vs Hold Rock's 0: a
hold starts no cooldown, so the Sneak use is told apart by `Live.EARTH_HOLDING` rising). **Prediction:** on the press
when earth is in reach (the client raycasts the ground within 6) and the slot is ready.

**Mastery variants:** M4–6 (two rocks): the O arm mirrors the M push 2 ticks later (`O` at tick 13: −92,8,−4) and the
twist goes −12; M7–9: a third rock from the first palm again at tick 15 (the gesture is `EARTH_ROCK_THROW_3`, dur 20).
M10 boulder (`EARTH_BOULDER`, dur 26 / 3 / 8): the stomp at 5 is deeper (drop 0.36), both arms rise to −150 by tick 12
(the boulder floats over the head, `EARTH_LIFT` with x −150), then **both palms thrust** at tick 18 (`FIRE_DOUBLE`-like
−92,±6,±4) with lean 14 and a 0.1 step forward (`drop` 0.20 → 0.10); recovery to 26.

**Sneak: Hold Rock, `EARTH_ROCK_LIFT`** (dur 12 / 2 / 5, not force): the stomp keys 0–5 as above, then at tick 8 the M
settles into `EARTH_CLAW` aimed at the rock and hands over to the `EARTH_GRIP` stance (the gesture ends on the stance's
pose: no pop). Throwing from the hold plays `EARTH_ROCK_THROW` from tick 9 (a `param` of 9 skips the stomp). Setting
down: `EARTH_ROCK_DOWN` (dur 10): the claw opens (y 18 → 0) and lowers to x −60 with lean 6, as the rock sinks back.

**First person:**

| t | M hand (dx, dy, dz, rx, ry, rz) | O hand |
|---|---|---|
| 0–3 | 0, −0.06, 0, +10, 0, 0 (hands drop as the leg lifts) | mirrored |
| 5 | +0.02, **+0.22**, +0.06, −30, 0, +8 (both hands fly up with the stomp, palms up) | mirrored |
| 9 | 0, +0.16, +0.10, −25, 0, +4 (float) | −0.02, +0.04, 0, −10, 0, −4 (the off hand drifts back) |
| 11 | **+0.04, +0.02, +0.30, −8, −6, 0** (the thrust) | −0.08, −0.04, −0.08, 0, 0, 0 |
| 16 | rest | rest |

The camera dips 0.04 blocks at the stomp (the `drop` of the body is applied to the first-person camera too:
Enhanced applies drop to the pose stack only; **NEW** 14.4: a small camera-drop hook, reduced-motion gated).

**Effects:**

| Where | What | Count | Colour |
|---|---|---|---|
| Stomp (t 5) | `bending:grain` burst at the foot + a flat dust ring (`MATTER_FLAT`, radius 0.3 → 1.4 over 6 ticks, α 0.5 → 0) | 10 grains, 1 ring | the ground block's tint (`BlockParticleOption` of the state) |
| Rock rising (t 0–5) | the **rock entity** (`RockEntity`, a block display: `renderSingleBlock` of the torn state, scale 0.9, rising with `easeOutBack`; the hole fills with a dark `MATTER_FLAT` disc that fades as the temp block reverts) + 6 grains falling off it per tick for 5 ticks | 6/tick | ground tint |
| Float (t 5–11) | 1 grain per 2 ticks dropping from the rock; the rock turns 4°/tick | 0.5/tick | |
| Flight | the rock tumbles (Y 11°, X 7° per tick), a trail of 1 grain per tick; no glow (matter) | 1/tick | |
| Impact (creature) | 14 grains + `BLOCK` dust ×8 of the rock's state + a 0.6-block `MATTER_FLAT` puff disc; the rock vanishes | 22 | |
| Impact (block) | the same, plus a 0.2 shake within 8 blocks | 22 | |
| Boulder (M10) | the rock is 2×2×2 (8 block displays as one mesh, `renderSingleBlock` ×8 or one scaled cube with the block texture), trail 3 grains/tick, impact 40 grains + shake 0.8 | | |
| Per-player cap | 6 rock entities, 60 grains alive | | |

**Sounds:** wind-up `earth_stomp` (t 3–5: a deep thud with a gravel crunch) → release `earth_rock_throw` (t 11: a
low whoosh with grit) → impact `earth_rock_hit` (a cracking thump; on a creature a duller `earth_rock_hit_flesh`);
boulder: `earth_boulder_throw` / `earth_boulder_hit` (bigger, a sub-bass drop). Hold: `earth_rock_lift` on lift, a
quiet rumble loop `earth_hold_loop` while held (volume 0.3), `earth_rock_down` on set-down.

### 3.2 Earth Armour — שריון אדמה

**Forming, `EARTH_ARMOUR_FORM`** (dur 20 / 2 / 6, CAST, force; **locks movement** for its 20 ticks via `Live.EARTH_FORMING`):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | 0 | | |
| 4 | O | −30,0,40 | −30,0,−40 | −36 / 0 | −4 | lean −6 | the right leg lifts |
| 6 | I | −20,0,38 | −20,0,−38 | 0 / 0 (±18) | 6 | lean 6, **drop 0.32** | the **stomp** (horse stance): earth answers |
| 12 | S | −60,30,30 | −60,−30,−30 | (±18) | 0 | lean 2, drop 0.26 | both arms rise, elbows out: the stone climbs the body (the model layer shows from the feet up: 3.2e) |
| 16 | E | `EARTH_GUARD` | `EARTH_GUARD` | (±16) | 0 | lean 2, drop 0.22 | forearms cross before the chest: the chest plate closes |
| 20 | O | −40,0,50 | −40,0,−50 | (±16) | 0 | lean 2, drop 0.20 | arms **snap** out as the helmet seals: hand over to `EARTH_ARMOURED` |

**Shedding / breaking, `EARTH_ARMOUR_SHED`** (dur 10 / 1 / 4): arms to −60,0,60 / −60,0,−60 at tick 3 with `B`
(a shrug), lean −6, then rest; the layer crumbles (3.2e). **Shed Blast, `EARTH_ARMOUR_BLAST`** (dur 14 / 1 / 5,
force): the same shrug at tick 3, then at tick 5 both arms **slam down** to 20,0,30 / 20,0,−30 with drop 0.28 and lean
12 (`I`), a stomp of both feet; recovery to 14.

**Lock:** forming locks input (20 ticks); shedding does not. **Blend:** forming is refused while airborne, so no
flight blend; while armoured the walk is the `EARTH_ARMOUR` run character (heavier) and jumping is 0.9 height (server:
jump boost −; **(added)** the shell weighs). Under water the shell crumbles (`EARTH_ARMOUR_SHED` plays). **Started by:**
`Toggle.EARTH_ARMOUR` flip (on → FORM, off → SHED) and the FX payload kind `ARMOUR_BREAK` (→ SHED with a bigger
crumble); the blast by the slot's cooldown edge (20 s) while the toggle goes off in the same tick. **Prediction:** on the
press when standing on earth and not airborne.

**First person (forming):** both hands rise with the stone: t 6 (stomp) `dy −0.10, rx +15`; t 12 `dx ±0.12, dy +0.10,
dz +0.05, rx −20, rz ±15` (elbows out); t 16 hands cross in front of the face `M dx −0.14, dy +0.12, dz +0.12, ry −35`,
O mirrored; t 20 hands snap outward `dx ±0.25`. Then the **eye slit overlay** (2.2) closes over 20 ticks, from tick 8 to
28 (it lags the body by 8 ticks so you see the stone climb). The hands are drawn **in stone** while armoured: the
first-person arm uses the armour layer's texture via a `RenderArmEvent` override (Enhanced's `DeadpoolGearLayer`
pattern).

**Effects:**

| Where | What | Count | Colour |
|---|---|---|---|
| Stomp (t 6) | 14 grains + dust ring r 0.3 → 1.6 + shake 0.6 | 15 | ground tint |
| Climb (t 6–20) | the **armour layer** (`LOOKS.md` 1.1: a `ModelPart` suit of plates over the inflated player model, `CubeDeformation(0.35)`) is revealed **bottom-up** by a clip plane: `y = feet + 2.1·(t−6)/14` (an alpha mask in the texture coordinates is not available on the cutout type; use a scissor in model space: render the layer in 6 vertical slices as separate parts and show each when the plane passes it) | — | the ground's stone: the layer's texture is tinted by the torn block's map colour (grass → brown-green dirt plates; stone → grey; sandstone → yellow) |
| Climb particles | 4 grains/tick spiralling up the body; 2 `BLOCK` dust/tick | 6/tick | ground tint |
| Seal (t 20) | a puff of 10 grains at the head; the slit glows for 4 ticks | 10 | |
| Worn | every 40 ticks 1 grain falls from a plate; walking leaves 1 grain per step; the model's footstep particles are block dust | 0.1/tick | |
| Hit on the shell | 6 grains + `BLOCK` dust ×4 at the hit, a crack appears on the layer's texture overlay (3 crack stages as overlay textures `armour_cracks_1..3.png`, `entityTranslucent`) | 10 | |
| Break / shed | the slices fall off in 6 ticks top-down (each slice becomes 2 `FallingBlock`-style client particles: 12 grain bursts) + dust ring + shake 0.5; Shed Blast adds a stone-shard ring: 24 `bending:grain` at speed 0.6 outward + a `MATTER_FLAT` expanding ring r 0.5 → 4 over 8 ticks (α 0.6 → 0) | 40–60 | |
| Seismic Sense (M8) | hostile mobs within 16 get `EntityGlow` ochre `#C9B47A` while you stand still armoured | — | |
| Per-player cap | 1 armour layer, 40 grains | | |

**Sounds:** `earth_armour_form` (t 6–20: a stomp, then stone sliding and grinding up, ending in a heavy "clunk" seal),
`earth_armour_step` (replaces the footstep: a stony tread, volume 0.4), `earth_armour_hit` (a chip and a dull knock),
`earth_armour_crack` (a sharp split at each crack stage), `earth_armour_break` (rubble collapsing with a rising cloud),
`earth_shed_blast` (the same with a crack and a wide hiss of shards).

### 3.3 Earth Pillar — עמוד אדמה

**`EARTH_PILLAR`** (dur 14 / 2 / 5, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 3 | O | −20,0,40 | −20,0,−40 | 0/0 (±18) | 4 | lean 4, **drop 0.26** | both knees bend (the gather) |
| 6 | I | **−150,0,20** | **−150,0,−20** | 0/0 (±16) | −10 | **lean −8**, drop 0.06 | both palms **drive upward** past the shoulders: the ground rises under you (the rise starts at tick 5) |
| 10 | S | −120,0,26 | −120,0,−26 | (±14) | −6 | lean −4, drop 0.08 | riding up, arms held high |
| 14 | S | `EARTH_REST` | `EARTH_REST` | (±8) | 0 | lean 2, drop 0.04 | settle on top (the pillar reaches its height at tick 5 + height/0.5 ≈ 11–21; the gesture ends before, the `EARTH_RIDE_UP` stance holds the arms at −60,0,30 while the pillar still rises, `tauIn 2 / tauOut 4`, on while `Live.EARTH_PILLAR_RIDING`) |

**Lowering** (press again): `EARTH_PILLAR_DOWN` (dur 10): both palms **press down** from −60 to 10,0,30 with lean 6 at tick
4 (`I`), the pillar sinks; rest at 10. **Earth Launch, `EARTH_LAUNCH`** (dur 12 / 1 / 4, force): a deeper crouch at
tick 3 (drop 0.34, legs ±20, arms back 20,0,30 / 20,0,−30), then at tick 5 the arms **throw up** to −176,0,10 /
−176,0,−10 with lean −14 and drop 0 as the stub fires you; the air pose hands over to the shared `LAUNCHED` stance (11.3:
arms wide, legs trailing) until landing, which plays `LAND_SOFT` with a 0.12 drop kick.

**Lock:** none, but while riding the pillar the server moves you with it (`EARTH_PILLAR_RIDING`: your position is set
to the top each tick, as a passenger of a `PillarEntity`... see `TECH.md` 3: the pillar is temp blocks, the player is
moved by `setPos` with `hurtMarked` each tick: smooth because the client interpolates). **Blend:** the gesture's legs
are keyed only at 3 (bend) so a walking player bends mid-step; flying: refused. **Started by:** the slot's cooldown
edge (3 s vs the Launch's 4 s, told apart by length as Thanos's are), and `Live.EARTH_PILLAR_RIDING` for the ride stance.
**Prediction:** standing on earth (client check) and ready.

**First person:** t 3 `dy −0.12, rx +20` (crouch); t 6 both `dx ±0.06, dy +0.34, dz +0.04, rx −45, rz ±10` (palms up,
driving); t 10 hold `dy +0.22`; t 14 rest. The camera rides up smoothly (interpolated position). Launch: t 5 both hands
`dy +0.40` then they trail down `dy −0.10, rx −10` in the air (the `LAUNCHED` first-person: hands slightly out and
back, wobbling ±0.01 at 0.3 rad/tick).

**Effects:**

| Where | What | Count |
|---|---|---|
| Rise | the **pillar** is temp blocks of the ground's state placed 1 per 2 ticks (0.5 block/tick), each new block "pops up" with 4 grains and `BLOCK` dust ×2; a `MATTER_FLAT` dust ring at the base r 0.6 → 1.5 at the start | 6 per block |
| Top | the player rides; 1 grain falls off the top edge per 2 ticks while rising | 0.5/tick |
| Standing | nothing | |
| Sink | blocks removed top-down 1 per 2 ticks with 3 grains each; the rider descends | 3 per block |
| Aimed pillar (M5) | a dust ring at the aimed spot for 6 ticks before it rises (the tell: `MATTER_FLAT` ring r 1.0 pulsing α 0.3 → 0.6) | 1 ring |
| Launch | a 3-block **column of dust** (`FxShapes.cylinder` r 0.5 → 0.2, h 0 → 3 over 6 ticks, α 0.5 → 0, `MATTER_FLAT`) + 20 grains upward + shake 0.4 | 21 |
| Cap | 1 pillar (8–16 temp blocks), 24 grains | |

**Sounds:** `earth_pillar_rise` (a rising rumble with stone clunks every 2 ticks for the duration; built from a loop
`earth_rumble_loop` + one-shot `earth_clunk` per block), `earth_pillar_sink` (the same descending), `earth_launch` (a
stomp and a sharp stone "crack-whoosh"), the landing `earth_land_soft`.

### 3.4 Earth Wall — קיר אדמה

**`EARTH_WALL`** (dur 14 / 1 / 5, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 2 | O | 10,0,36 | 10,0,−36 | −34 / 0 | −4 | lean −4 | right leg lifts (a quick stomp) |
| 4 | I | −40,10,30 | −40,−10,−30 | 0/0 (±18) | 4 | lean 6, **drop 0.28** | **stomp**: the wall jumps up in 4 ticks (server: rows placed at ticks 4, 5, 6, 7) |
| 8 | O | **`EARTH_GUARD`** | **`EARTH_GUARD`** | (±16) | 0 | lean 2, drop 0.22 | forearms **cross and rise** before the chest: the wall tops out |
| 14 | S | `EARTH_HORSE` | `EARTH_HORSE` | (±16) | 2 | lean 4, drop 0.24 | hand over to `EARTH_IDLE` (the bender stays in the horse behind their wall) |

**Wall Push, `EARTH_WALL_PUSH`** (dur 18 / 1 / 6, force): keys 0–8 as above (if a wall stands, the gesture starts at
tick 8 via `param`), then at tick 10 (`I`) **both palms thrust** to −88,±6,±4 with lean 14, drop 0.16, twist 0, a step:
right leg x −20 / left +20; hold to 14; recovery at 18. The wall slides from tick 10 over 12 ticks.

**Lock:** none. **Blend:** walking continues (the legs are keyed only at the stomp); flying: allowed in Creative (the
wall rises where aimed; the stomp is masked, 14.3). **Started by:** the slot's cooldown edge (8 s vs the push's 10 s).
**Prediction:** earth under the footprint (client raycast) and ready.

**First person:** t 2 `dy −0.06`; t 4 both `dx ±0.04, dy −0.14, rx +25` (the stomp pulls the hands down and in); t 8
both hands **rise crossed** before the face `M dx −0.16, dy +0.14, dz +0.12, ry −30, rz +10`, O mirrored (the wall rising
fills the lower half of the view); t 14 lower to the idle hands. Push: t 10 both `dz +0.32, dy +0.02, rx −10` (a
two-palm shove), kick back `dz −0.06` at 12.

**Effects:**

| Where | What | Count |
|---|---|---|
| Tell (t 0–4) | a `MATTER_FLAT` dust line along the wall's footprint, α 0.3 pulsing | 1 |
| Rise (t 4–7) | each row of temp blocks appears bottom-up (row n at tick 4+n) with `BLOCK` dust ×2 and 3 grains per block, and a dust curtain: a `SandShapes.wall`-like translucent plane in front of the wall, α 0.5 → 0 over 8 ticks, `MATTER_TEXTURED(dust.png)` | 5 per block (≈ 30–140) |
| Standing | 1 grain falls from the top every 20 ticks per 3 blocks of width; when a projectile hits: 6 grains + `BLOCK` dust ×3 at the hit | low |
| Sink (t 200) | rows vanish top-down over 8 ticks with 2 grains per block | |
| Push | the wall's blocks move 1 cell per 2 ticks (temp blocks re-placed: at most 28 blocks × 6 steps); a dust plume behind the wall (`FxShapes.cylinder` lying along the wall, α 0.4) and 8 grains per step; things hit get 6 grains | 8/step |
| Cap | 1 wall (≤ 28 blocks, 56 at M10), 80 grains | |

**Sounds:** `earth_wall_rise` (a stomp into a fast rising stone slide, 0.4 s, four clunks), `earth_wall_hit` (a knock
with grit, pitched by the projectile's damage), `earth_wall_sink` (a slow grinding fall), `earth_wall_push` (a heavy
scrape loop `earth_scrape_loop` for the 12 ticks + a thud at the end).

### 3.5 Sand Bending — כשפות חול

**Sand Stream, `EARTH_SAND_STREAM`** (dur 16 / 2 / 6, CAST, force): a lighter Hung Gar, no stomp:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 4 | E | **−130,20,20** | **−130,−20,−20** | 0/0 (±14) | −8 | lean −6, drop 0.12 | both arms **scoop up** from the sides: the sand lifts in a sheet |
| 8 | S | −100,−30,40 | −100,30,−40 | (±14) | −4 | lean −2, drop 0.14, twist −10 | the arms **circle out** (a wide gather) |
| 11 | I | **−86,−10,−6** | **−86,10,6** | (±14) | 2 | **lean 10**, drop 0.10, twist 8 | both palms **push** together: the stream flies |
| 16 | S | `EARTH_REST` | `EARTH_REST` | (±8) | 0 | lean 2, drop 0.04, twist 0 | |

**Sand Wall, `EARTH_SAND_WALL`** (dur 12 / 1 / 5): like `EARTH_WALL` without the stomp: arms **sweep up from the
knees** (t 2: −10,0,36 / −10,0,−36 with drop 0.20; t 6: `EARTH_GUARD` with `E` and lean −2: the sand pours upward),
hand-over to `EARTH_IDLE` at 12.

**Lock:** none. **Blend:** walking continues. **Started by:** the slot's cooldown edges (2.5 s stream vs 6 s wall).
**Prediction:** sand in reach (or 4 sand in the inventory) and ready.

**First person:** stream: t 4 both `dx ±0.20, dy +0.22, dz +0.02, rx −30, rz ±20` (the scoop), t 8 both `dx ±0.28, dy
+0.12, ry ±20` (circle out), t 11 both `dx ±0.02, dz +0.30, rx −6` (push together), rest. Wall: t 2 `dy −0.16, rx +20`
(hands at the knees), t 6 `dy +0.18, dz +0.10, rz ±12` (rising palms), idle.

**Effects:**

| Where | What | Count |
|---|---|---|
| Stream lift (t 0–11) | 20 `bending:grain` (sand tint `#D8B76A`, red sand `#A95B2E`, soul sand `#4E3A2C`) rise from the sand within 3 blocks into a sheet 1.5 wide before the bender (a hanging `MATTER_TEXTURED(sand_swirl)` sheet, α 0 → 0.5), 3 grains/tick | 3/tick |
| Stream flight | one `SandStreamEntity`: a cone drawn as 2 `SandShapes.funnel`-like frustums lying forward (r 0.3 → 1.2 over the length), α 0.6, scrolling, plus 6 grains/tick along it; blinded targets get 4 grains at the eyes | 6/tick |
| Sand Wall | rows of temp **sand** blocks poured in from below: a `SandShapes.ridge` crest runs up the front face as it rises; 4 grains per block; standing: 1 grain/20 ticks per 2 blocks slides down; collapse: a `MATTER_FLAT` sand slump at the foot | 4 per block |
| Cap | 1 stream entity, 1 wall, 60 grains | |

**Sounds:** `sand_lift` (a dry hiss rising, 0.5 s), `sand_stream` (a sustained grainy rush 0.6 s + a soft "pelt"
loop), `sand_hit` (a sandy pelt on a creature), `sand_wall_rise` (pouring sand reversed: a swelling hiss), `sand_wall_fall`
(the slump).

### 3.6 Quicksand — חול טובעני

**`EARTH_QUICKSAND`** (dur 20 / 2 / 6, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 4 | O | **−70,−10,20** | −30,10,−30 | 0/0 (±16) | 0 | lean 4, drop 0.20 | the M palm **points down at the target's feet** (`mAim(LOOK, 30, 0, 0)`: 30° below the look) |
| 8 | E | aim(LOOK, 20, −20, 0) | aim(LOOK, 20, 20, 0) | (±16) | 4 | lean 8, drop 0.22 | both palms, fingers spread (y out), **tremble 2.0 at 2.4** (the pull) |
| 14 | I | aim(LOOK, 40, −10, 0) | aim(LOOK, 40, 10, 0) | (±16) | 8 | **lean 14**, **drop 0.30** | both hands **press downward** slowly (`I`): the target sinks |
| 20 | S | `EARTH_HORSE` | `EARTH_HORSE` | (±16) | 2 | lean 4, drop 0.24 | hand over to the idle |

**Sand Cloud, `EARTH_SAND_CLOUD`** (dur 14 / 2 / 5): arms **swirl up**: t 4 `AIR_CIRCLE_A`-like −120,−40,40 / −30,20,−10
with twist 10; t 8 the mirror with twist −10 (a figure-eight); t 11 both palms **open outward** at −80,±40,∓40 (the cloud
blooms); rest.

**Lock:** none. **Blend:** walking continues. **Started by:** cooldown edges (10 s each: the two are told apart by the
FX payload kind `QUICKSAND` vs `SAND_CLOUD`, `startedElsewhere`). **Prediction:** a creature in sight within 12 on sand.

**First person:** t 4 `M dz +0.22, dy −0.08, rx +30` (point down); t 8 both `dx ±0.10, dy −0.04, dz +0.22, rx +25, rz
±10` with tremble 0.012; t 14 both `dy −0.18, dz +0.18, rx +40` (press down); idle. Cloud: hands swirl (a circle of r
0.08 over 8 ticks) then open `dx ±0.26`.

**Effects (target side):** the 3×3 patch's temp blocks swap to `bending:quicksand` (a sand texture with a slow
**ripple** animated texture, 4 frames); 6 grains rise and fall at the patch per tick; the sinking creature gets a
`MATTER_FLAT` sand collar (a ring r 0.6 at its chest height, α 0.6) and 2 grains/tick sliding down it; the victim's
first-person overlay (2.2). Victim pose (12.1). **Sand Cloud:** a `SandShapes.funnel`-like **sphere** of swirl texture r 6,
rotating, α 0.45, with 10 grains/tick drifting inside; blinded targets get 4 grains at the eyes. **Cap:** 2 patches, 1
cloud, 60 grains.

**Sounds:** `quicksand_cast` (a low sandy groan that drops in pitch, 0.6 s), `quicksand_sink_loop` (a wet, grainy
churning loop at the victim, volume 0.5), `quicksand_free` (a sucking pop), `sand_cloud` (a broad hiss swell 0.8 s, then
a loop `sand_cloud_loop` at 0.3).

### 3.7 Lava Bending — כשפות לבה

**Lava Path, `EARTH_LAVA_PATH`** (dur 22 / 2 / 7, CAST, force): a slow, heavy Hung Gar with heat in it:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 4 | O | 10,0,40 | 10,0,−40 | −36 / 0 | −4 | lean −6 | the right leg lifts high |
| 6 | I | −30,0,36 | −30,0,−36 | 0/0 (±20) | 6 | lean 8, **drop 0.34** | the **deep stomp**: the ground under the foot glows (the first path cell melts at tick 6) |
| 12 | E | **−80,0,30** | **−80,0,−30** | (±18) | 2 | lean 6, drop 0.26 | both palms **press forward and down** slowly (`E`), fingers spread, **tremble 1.6 at 1.4**: the path runs out ahead 1 cell/tick |
| 18 | S | −70,0,34 | −70,0,−34 | (±18) | 2 | lean 6, drop 0.26 | hold (the path keeps running: the gesture hands over to `EARTH_LAVA_HOLD` stance for the remaining ticks, aimed, tremble 1.6) |
| 22 | S | `EARTH_HORSE` | `EARTH_HORSE` | (±16) | 2 | lean 4, drop 0.24 | |

**Lava Shuriken, `EARTH_LAVA_SHURIKEN`** (dur 14 / 1 / 5, force): t 3 the M arm **winds across the body** (−60,50,−30,
twist −18, lean 2), the O arm forward low (−40,−10,−20); t 5 the disc forms at the M hand (effects); t 8 (`I`) a
**horizontal throw**: M to −80,−40,30 with twist +20 and lean 8, a step (right leg −16 / left +16); recovery at 14. M8
(two discs): the O arm throws 3 ticks later mirrored.

**Lock:** none (walking continues; the path follows your look as you walk). **Blend:** flying: refused. **Started by:**
cooldown edges (12 s vs 4 s). **Prediction:** earth ahead and ready (path); ready (shuriken).

**First person:** path: t 4 `dy −0.06`; t 6 both `dy −0.16, rx +25` (stomp); t 12 both `dx ±0.14, dy −0.06, dz +0.26,
rx +30, ry ∓10` (palms pressing down and out) with tremble 0.012 (the heat shimmer below the hands: 2.2 Fire Breath
style ripple in the lower 20% of the screen, warm); hold; idle. Shuriken: t 3 `M dx −0.28, dz +0.02, ry −40` (wound
across), t 8 `M dx +0.30, dz +0.24, ry +30` (the flat throw), rest.

**Effects:**

| Where | What | Count |
|---|---|---|
| Stomp | 14 grains + ring + shake 0.2; the first cell cracks: `BLOCK` dust ×6 then **glows** (a `MATTER_FLAT` disc `#FF7A1F` α 0 → 0.8 over 4 ticks) before becoming temp lava | 20 |
| Path running | each cell: 4 ticks of crack glow, then the temp **lava** state (vanilla lava's animated texture; `bending:temp_lava` reuses `minecraft:lava` textures); 2 `LAVA` + 1 `SMOKE`... (`CAMPFIRE_COSY_SMOKE` ×1) per cell per 2 ticks while it burns; heat shimmer: a `MATTER_TEXTURED(heat.png)` translucent sheet 0.5 above the path, scrolling, α 0.25 | ≈ 1.5/tick per cell, capped at 20/tick for the path |
| Cooling | the cell darkens over 20 ticks (an overlay disc `#2A1A10` α 0 → 0.9 drawn on top, `MATTER_FLAT`) then the original block returns with 3 `WHITE_ASH` | 3 per cell |
| Shuriken | a **disc** entity: `FxShapes.disc` r 0.5 (0.65 at M8) `#FF6A1F` core + `annulus` rim `#8C2410`, spinning 30°/tick, additive glow halo (`FxShapes.glow` r 0.7 α 0.3), trail 2 `LAVA` + 1 `FLAME`/tick; impact: 16 `LAVA` + 8 `FLAME` + a 1-block `MATTER_FLAT` scorch disc for 60 ticks | 3/tick |
| Lava passive | at M10 while immune: 2 `FLAME` rising from the feet per tick | |
| Cap | 1 path (≤ 16 cells), 2 discs, 40 lava particles | |

**Sounds:** `lava_path_cast` (stomp + a deep molten rumble swelling 0.8 s), `lava_path_loop` (a bubbling hiss while
cells burn, follows the path's head, volume 0.5), `lava_cool` (a sizzling tick as cells cool), `lava_shuriken_throw` (a
hot whoosh with a spinning flutter), `lava_shuriken_hit` (a splat-sizzle), `lava_shuriken_spin_loop` (a thin flutter
while it flies, volume 0.3).

### 3.8 Metal Bending — כשפות מתכת

The metal gestures are **Lin Beifong's**: sharper and smaller than earth, wrists snapping, little stomp.

**Steel Knives, `EARTH_STEEL_KNIVES`** (dur 12 / 1 / 4, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `EARTH_REST` | `EARTH_REST` | | | | |
| 3 | O | **−50,20,−10** | −30,0,−26 | 0/0 (±12) | 0 | lean 2, drop 0.10 | the M forearm **snaps up** across the chest, wrist cocked: the knives form on the forearm (effects) |
| 5 | I | **−88,−20,20** | −30,0,−26 | −12 / 12 (±12) | 2 | **lean 8**, twist 10 | the **flick**: wrist snaps forward and out, knives fly (t 5, 6, 7 one each) |
| 8 | S | −84,−10,16 | −30,0,−26 | (±12) | 2 | lean 6, twist 8 | held out |
| 12 | S | `EARTH_REST` | `EARTH_REST` | (±8) | 0 | lean 2, twist 0 | |

M5+ (5 knives): the O arm flicks too, 2 ticks after (t 7: −88,20,−20), knives alternate hands. **Recall** (Sneak-key
within 5 s of stuck knives): `EARTH_KNIFE_RECALL` (dur 8): the M hand **opens toward the target** (aim(LOOK, −10, 0, 0),
`B`) and **closes** (y −20 → 0) as the knives streak back into charge.

**Metal Pull, `EARTH_METAL_PULL`** (hold; `EARTH_METAL_PULL_START` dur 8 / 1 / 3 then the stance `EARTH_PULLING`
(aimed, tauIn 2 / out 4) while `Live.METAL_PULLING`): the start: both arms **reach out and up** to −70,∓30,±20 with
fingers spread (y) by tick 4 (`E`), lean 4, drop 0.12; the stance: arms at −60,∓20,±16 with **tremble 2.2 at 2.8**
(the strain), head 4, lean 6; every 10 ticks the hands **pull back** 8° in x (a tug, `Stances.drift` with a sawtooth).
Releasing plays `EARTH_METAL_PULL_END` (dur 8): fists close (y 0) and drop to rest.

**Steel Wall, `EARTH_STEEL_WALL`** (dur 12 / 1 / 4, force): a Lin **chop**: t 2 both arms raised to −140,∓10,±10 (`O`),
drop 0.10; t 5 (`I`) both forearms **chop down** to −60,0,±14 with lean 10 and a short stomp (right leg −20 → 0 at 6): the
plates **shoot up** from the charge at the bender's feet toward the footprint (effects); t 12 `EARTH_HORSE`-lite (drop
0.14) hand-over to idle.

**Steel Armour, `EARTH_STEEL_ARMOUR_ON`** (dur 16 / 2 / 5; **locks** 16 ticks via `Live.EARTH_FORMING`): t 3 both arms
cross the chest (`EARTH_GUARD` with x −90: hands at the collar), drop 0.14; t 6 (`O`) arms **sweep down the body** to
−20,∓20,±26 as plates **clap** onto the body top-down (the reverse of earth: steel comes from the hands, not the
ground); t 10 arms snap out to −40,0,±50 (`B`) as the last plates lock; t 16 `EARTH_ARMOURED` hand-over. **Off,
`EARTH_STEEL_ARMOUR_OFF`** (dur 10): a shrug at 3 (`B`), plates fly off into the hands (charge returns) as the arms
gather at the belly (`WATER_GATHER`-like) at 7; rest.

**Wreck Metal, `EARTH_WRECK`** (dur 14 / 1 / 5, force): t 3 the M hand **reaches** (aim(LOOK, 0, 0, 0), fingers spread
y 20, `O`); t 6 (`I`) the hand **clenches and twists**: y 20 → −30, z 0 → 24, with a wrist roll implied by `ry` in first
person, twist −12, lean 6: the device crumples at tick 6; t 10 the hand **flicks away** (aim(LOOK, −20, 30, 0), `O`);
t 14 rest. At M5+ (3×3) both hands reach and clench (O mirrored).

**Metal Grip, `EARTH_METAL_GRIP`** (dur 10 / 1 / 4, then the stance `EARTH_METAL_HOLD` aimed at the held entity
(`aim(HELD, −16, 0, 0)` with tremble 1.8 at 2.0, O braced at −20,0,−30, lean 6, `occupy(M)`) while `Live.EARTH_HOLDING`
on a metal target): the start: M **thrusts open-handed** at the target (aim(LOOK, 0, 0, 0), `O` at tick 3), **closes**
(y −18) at tick 5 as the grip takes; hand-over. Fling: `EARTH_METAL_FLING` (dur 10): the arm **whips** from the hold to
aim(LOOK, −30, 10, 0) with twist 14 and lean 10 (`I` at tick 4), rest. Set down: `EARTH_ROCK_DOWN` reused.

**Lock:** Steel Armour on locks 16 ticks; nothing else. **Blend:** metal gestures key no legs except the chop's stomp,
so they ride over walking cleanly; flying: knives and wreck allowed in Creative flight (the chop is masked). **Started
by:** cooldown edges (2.5 s knives, 6 s pull, 8 s wall, 20 s armour-break, 4 s wreck, 8 s grip); `Toggle.STEEL_ARMOUR`
flips; `Live.METAL_PULLING`, `Live.EARTH_HOLDING`; payload kinds `KNIVES_RECALL`, `WRECK`. **Prediction:** charge > 0
(the client knows the synced charge) and ready; grip: a metal target in sight.

**First person:** knives: t 3 `M dx −0.10, dy +0.14, dz +0.06, ry −30, rz +20` (the forearm across, knives visible on
it: 3.8e), t 5 `M dx +0.16, dy +0.04, dz +0.28, ry +20, rz −10` (flick), rest; the O hand mirrors 2 ticks later at M5+.
Pull: both hands out `dx ±0.16, dy +0.02, dz +0.22, rx +10, ry ∓15`, trembling 0.012, tugging back `dz −0.05` every 10
ticks; filings stream into the palms (effects). Steel Wall: t 2 both `dy +0.30, rx −40` (raised), t 5 both `dy −0.10,
dz +0.14, rx +30` (chop). Steel Armour on: hands at the collar `dx ∓0.10, dy +0.16, dz +0.06` then sweep down `dy −0.14`,
snap out `dx ±0.24`; the hands turn to steel (the arm texture swap) at tick 8; the **steel eye slit** overlay (2.2)
closes over 16 ticks. Wreck: `M dz +0.26, ry +10` (reach), then `ry −40, rz +25, dz +0.18` (clench-twist, tremble
0.01), then `dx +0.14, dz +0.10` (flick). Grip: reach `dz +0.26`, close `ry −20`, hold with tremble; fling `dx +0.22, dz
+0.22`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Knives form (t 3–5) | 3–7 **knife quads** (`FxShapes.box`-like thin boxes 0.5 long, `MATTER_FLAT` steel grey `#B8C0CC` with a lit edge `#E8EEF5`) appear along the forearm (`HandTracker.hand` + the arm direction) one per tick with 2 `ELECTRIC_SPARK`... no: 2 `CRIT`-style white motes (energy `#E8EEF5` scale 0.5) each | 2 per knife |
| Knife flight | `SteelKnifeEntity`: the same box, spinning about its long axis 25°/tick, a 0.6-block faint `ENERGY_BEAM` streak behind (α 0.2); no particles | 0 |
| Knife hit | 4 white motes + the knife **stays** in the target (drawn stuck, offset to the hit point, 3 s) or in the wall (a temp decoration: the entity rests) | 4 |
| Recall | each knife streaks back as a `ENERGY_BEAM` ribbon 1.2 long `#D8E0EA` α 0.5 over 6 ticks, 2 motes at the hand on arrival | 2 |
| Metal Pull | **filings**: 6 `bending:grain` (steel grey, tinted by the source block: iron `#B8C0CC`, gold `#F2D15A`, copper `#D08A5A`) per tick streaming from each metal block within 8 along a curve to the hands (`EnergyGeometry.flowingPath` sampled: spawn at parametric positions), + a faint `ENERGY_BEAM` thread per block α 0.12; the charge bar fills | 6 per block per tick, capped 24/tick |
| Steel Wall | plates (`bending:steel_plate` temp blocks, an iron-block texture with rivets) **slide up** from the ground: each row appears with 4 white motes and a metallic flash (a `MATTER_FLAT` quad α 0.6 → 0 in 3 ticks); a hit: 6 `ELECTRIC_SPARK`-like white motes and a ring on the plate | 4 per plate |
| Steel Armour | the steel suit layer (`LOOKS.md` 1.2) appears **top-down** in 5 slices (head, chest, arms, legs, boots) at ticks 6, 7, 8, 9, 10 with 6 white motes and a clap each; worn: a glint sweep (a `RenderType.eyes` pass over a highlight mask every 80 ticks for 6 ticks); hit: 6 sparks + a dent decal stage (3 stages); break: slices fall off top-down as 5 × 8 grey grains + a clang | 30 / 6 |
| Wreck | the device swaps to its **wrecked** temp state (a crumpled texture variant, `LOOKS.md` 6: the `bending:wrecked_*` blocks share one "bent metal" atlas with the device's base colour tinted) with a 6-tick **crunch**: 10 white motes + 4 `BLOCK` dust of iron + a 0.3 shake within 6; mending (t 60): 6 motes and a soft chime | 14 |
| Metal Grip | the held thing gets a steel-grey `ENERGY_VEIL` outline (an inflated copy of its box, α 0.25, 4 corner "clamps" as thin boxes) and 2 motes/tick at the hand; fling: a 1-block `ENERGY_BEAM` streak and 8 motes on impact | 2/tick |
| Cap | 8 knives, 1 wall, 1 armour, 1 grip, 40 motes/grains | |

**Sounds:** `metal_knives_form` (three quick steel "shinks" 0.4 s), `metal_knife_throw` (a thin whistle), `metal_knife_hit`
(a "tink" into stone / a thud into flesh), `metal_knife_stick` (a quivering twang), `metal_recall` (a reversed shink
glide), `metal_pull_loop` (a scraping, grinding hum with filings hiss, volume 0.5, pitch rises with the charge),
`metal_pull_end` (a clink), `metal_wall_rise` (plates sliding and locking: four clanks), `metal_wall_hit` (a resonant
plate ring), `metal_armour_on` (five claps of plates and a final lock), `metal_armour_off` (plates peeling: five
reversed claps), `metal_armour_hit` (a dented clang), `metal_armour_break` (a cascade of falling plates), `metal_wreck`
(a crunching, groaning twist of metal 0.6 s), `metal_mend` (a soft reversed crunch with a chime), `metal_grip` (a clamp
clank), `metal_fling` (a swung-metal whoosh).

### 3.9 Earth hit reactions (what a target shows)

See 12.1: **rock** hits stagger (a `HIT_STAGGER` reaction gesture on players: lean −14 over 2 ticks, back to 0 over 8,
head +6; mobs: the shared `LivingEntityRenderer` hit tilt, 12.1), **boulders** knock down (players: `HIT_KNOCKDOWN`,
lean −40 and drop 0.5 for 10 ticks, then up), **quicksand** gives the sinking pose, **wall push** staggers, **Steel
Knives** flinch (head −4 twist 6 for 4 ticks), **Metal Grip** gives the `LIFTED` pose (Enhanced's `Live.LIFTED`: arms
wide, legs trailing, drop −0.3).

---

## 4. Air

### 4.0 Air idle: the ready stance

| Stance | tauIn / tauOut | Rows |
|---|---|---|
| `AIR_IDLE` (ambient, key held / 40 ticks after an Air gesture) | 4 / 5 | `AIR_REST` plus a **slow circle**: the M hand draws a 12° circle (x −10 + 6·sin(0.12·age), z 10 + 6·cos(0.12·age)), the O hand the same a half-turn behind; weight shifts: lean 2·sin(0.06·age), legs x ±4 alternating with it (the Ba Gua "mud-wading" sway); head level. |
| `AIR_GUARDING` (aimed, while the Air Shield is up) | 3 / 4 | `AIR_GUARD` with both palms drifting out and in (z ±4 at 0.2 rad/tick), `occupy` none (you can fight inside it). |
| `AIR_RIDING` (ambient, on the scooter) | 2 / 4 | 10.1. |
| `AIR_GLIDING` | 2 / 4 | 10.2. |
| `AIR_MEDITATING` | 4 / 8 | 10.5. |

**Ambient particles of a master (Air ES ≥ 60):** a thin `GUST`-like breeze: 1 `CLOUD` (small, α 0.4, `EnergyParticle`
white `#F4F8FF` scale 0.6) every 8 ticks drifting in a slow circle at knee height, 2 at ES 100; a soft `AIR_LOOP`
breeze sound at volume 0.15 while standing still (reduced-motion: no particles).

**Air first-person idle (key held):** both hands up and open at shoulder height (`dx ±0.16, dy +0.06, dz +0.08, rx −10,
ry ∓20`), slowly circling (r 0.02 at 0.12 rad/tick).

### 4.1 Air Blast — הדף אוויר

**`AIR_BLAST`** (dur 14 / 1 / 5, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `AIR_REST` | `AIR_REST` | | | | |
| 3 | E | **−110,30,−30** | **−110,−30,30** | −8 / 8 | −4 | lean −6, twist −12 | both arms **circle up and across** (a gathering sweep from the outside in: the hands pass above the head) |
| 6 | E | −60,50,−40 | −60,−50,40 | −10 / 10 | 0 | lean −2, twist 12 | the circle comes down the other side: hands close together at the chest, palms forward |
| 8 | **O** | **`AIR_PALMS`** | **`AIR_PALMS`** | −20 / 14 | 2 | **lean 10**, twist 0 | **the release**: both palms thrust forward together, a step into it (the front leg forward); the blast leaves at tick 8 |
| 14 | S | `AIR_REST` | `AIR_REST` | | 0 | lean 0 | the hands float back down |

**Air Jump, `AIR_JUMP`** (dur 12 / 1 / 4, force): t 3 both palms **press down** beside the thighs (10,∓10,±24 with `E`),
knees bend (drop 0.18, legs ±8); t 5 (`O`) palms **snap down** to 30,∓10,±30 as the body **shoots up** (drop 0 → the jump
happens on the server at tick 5), lean −6; in the air the `LAUNCHED` stance (11.3) takes over until landing (`LAND_SOFT`,
with a puff). Mid-air second jump: the same from the flight pose.

**Lock:** none. **Blend:** the step at tick 8 drives the legs only for 2 ticks; walking continues otherwise; while
**gliding** (3.3), the blast aimed at the ground plays `AIR_BLAST_DOWN` (dur 10): palms sweep **down** to 20,0,±30 at tick
5 (`O`) from the glider grip, the body pitches up 10° (lean −10) as you rise; the glider pose resumes. Under water: half
range, the same gesture slowed ×1.5 (`Slot.rate`). **Started by:** the slot's cooldown edge (2 s; the jump's 2 s is told
apart by `Live.AIR_JUMPING` rising). **Prediction:** always (the blast needs no target).

**First person:** t 3 both hands sweep up and in: `M dx −0.12, dy +0.30, dz +0.06, ry −40, rz −20` (O mirrored); t 6
hands together low at the chest `dx ∓0.04, dy +0.04, dz +0.10, ry ∓10`; t 8 **thrust**: `dx ±0.06, dy +0.06, dz +0.34,
rx −12` (both palms fill the lower centre); recovery. The FOV kicks ×1.03 for 4 ticks at the release (not with reduced
motion). Jump: hands press down `dy −0.22, rx +35` then snap `dy −0.30`, the camera's rise is the jump.

**Effects:**

| Where | What | Count |
|---|---|---|
| Gather (t 3–8) | 2 white `EnergyParticle` (`#F4F8FF`, scale 0.5, α 0.5) per tick trailing the hands (`HandTracker.hand`), drawn as a wisp: a 6-point `EnergyGeometry.polyline` ribbon α 0.2 following each hand's last 6 positions | 4/tick + 2 ribbons |
| Release | a **blast cone**: a translucent `ENERGY_VEIL` cone (`FxShapes.cylinder` r 0.3 → 2.5, length 0 → range over 8 ticks, white α 0.18 → 0) with 3 rotating **swirl rings** (`EnergyGeometry.ring` r growing with the cone, α 0.3, white) and 10 `CLOUD` + 6 `GUST`-style white motes spawned along the cone's axis; grass and leaves in the cone **sway** (client: `GUST` vanilla particles at ground level, 4) | 20 |
| On a target | 6 white motes at the chest + the target's tumble (12.2) | 6 |
| Air Jump | a `MATTER_FLAT` white dust ring at the feet r 0.3 → 1.6 over 6 ticks α 0.4 + 8 `CLOUD` + a 1-block `FxShapes.cylinder` puff under the feet α 0.3 | 9 |
| Blast while gliding | the cone fires downward; the glider's cloth **snaps taut** (the glider layer's wings scale 1.0 → 1.08 for 3 ticks) | |
| Cap | 2 cones alive, 30 motes | |

**Sounds:** `air_gather` (t 3–8: a soft inhale-like whoosh rising), `air_blast` (t 8: a hard, breathy "whumpf" with a
sharp onset and a 0.4 s decaying rush), `air_hit` (a hollow thump when a creature is thrown), `air_jump` (a short
down-whoosh and a puff).

### 4.2 Air Shield — מגן אוויר

**Raise, `AIR_SHIELD_UP`** (dur 12 / 2 / 5, CAST, not force): t 3 both arms **circle outward and down** from `AIR_REST`
to −30,∓20,±50 (`E`); t 7 they **circle up** to −120,∓20,±30 (the hands trace a sphere around the body); t 12
`AIR_GUARD` hand-over to `AIR_GUARDING` (palms out at the sides, as if holding the sphere). **Lower, `AIR_SHIELD_DOWN`**
(dur 8): palms **press inward** to the chest (−60,∓40,∓10 at tick 4) and drop to rest; the sphere thins. **Break /
Burst, `AIR_SHIELD_BURST`** (dur 12 / 1 / 4, force): from the guard, both arms **snap wide** to −60,0,±80 at tick 3
(`B`) with lean −6 and drop 0 → −0.08 (a tiny hop); recovery.

**Lock:** none; the shield moves with you (speed ×0.8 at M1–6: an attribute, not a lock). **Blend:** walking, sprinting
and jumping continue; the guard stance keys only the arms; the stance's weight falls to 0.4 while sprinting (so the arms
can swing). Under water: refused. **Started by:** `Toggle.AIR_SHIELD` (on → UP, off → DOWN) and the payload kind
`SHIELD_BURST` (→ BURST). **Prediction:** on the press when ready and not in water.

**First person:** UP: hands sweep out `dx ±0.30, dy −0.08` then up `dy +0.28, dx ±0.14` then settle at the guard: `dx
±0.22, dy +0.06, dz +0.12, rx −10, ry ∓25` (both palms at the screen's edges, slightly turned out), with a slow drift
(±0.015 at 0.2 rad/tick) and, when a projectile is stopped, a 2-tick **flinch** of the nearer hand (`dz −0.04`). DOWN:
hands come to the chest `dx ∓0.02, dz +0.06` and fall. BURST: both hands fly to the edges `dx ±0.40, dy +0.04` in 3
ticks; the camera shakes 0.5.

**Effects:** the shield is Enhanced's shield pipeline (`ShieldStyle` + the sphere passes): a `ShieldStyle(veil 0xDDEBFF,
lines 0xFFFFFF, texture "textures/entity/air_lattice.png", radius 1.6→2.4, strength, maxTicks)`; the veil is drawn
**faint** (base α 0.05, rim 0.25: air is nearly invisible) in `ENERGY_VEIL`, the lattice is a **swirl**, not a hex web:
`EnergyGeometry.sphere` with the texture scrolling **around** (scroll on U at 0.05/tick, tilesU 4, tilesV 2) in
`energyField(air_lattice)` at α 0.15 / 0.35; plus **3 orbiting wind rings** (`EnergyGeometry.ring` r = radius at tilted
normals rotating 6°/tick, α 0.25 white, width 0.05) and 2 `CLOUD`-like white motes/tick drifting on the surface. A
stopped projectile: a **ripple** (`ShieldFx.hit` pattern: `circleOnSphere` arc growing over 10 ticks) and 6 white motes;
the projectile is flung with 2 motes. Strength: as it falls below 40% the rings **wobble** (their normals jitter ±8°,
smooth) and the veil flickers at 2 Hz (the shield's `look.flicker`); **burst**: the sphere swells ×1.4 over 4 ticks and
vanishes, an expanding `ENERGY_VEIL` **shockwave sphere** (r → burst radius over 8 ticks, α 0.3 → 0) + a `MATTER_FLAT` ground
ring + 24 `CLOUD` outward + `GUST` ×6 + shake 0.5. **Cap:** 1 shield, 20 motes. (Lowering by hand: the sphere fades over 6
ticks as Enhanced's `SHIELDS_GOING` does.)

**Sounds:** `air_shield_up` (a rising swirl into a sustained hush), `air_shield_loop` (a soft, airy rotating hum,
volume 0.35, pitch rises slightly as strength falls), `air_shield_hit` (a muffled "whup" + a flutter), `air_shield_down`
(the hush falling away), `air_shield_burst` (a deep "boom-whoosh" with a wide stereo rush, 0.9 s).

### 4.3 Air Glider — גלשן אוויר (item)

Covered in 10.2 (the glider pose). Opening: `AIR_GLIDER_OPEN` (dur 8 / 1 / 3, group RIDE): the M hand raises the staff
overhead (−160,0,10) at tick 3 (`O`); the wings **snap open** (the glider layer's wing parts rotate from folded (z ±5)
to open (z ±85) over 4 ticks with `B`); hand-over to `AIR_GLIDING`. Closing: the reverse in 6 ticks.

### 4.4 Air Blades — להבי אוויר

**`AIR_BLADES`** (dur 12 / 1 / 4, CAST, force): a **cutting** arm, Ba Gua's palm change:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `AIR_REST` | `AIR_REST` | | | | |
| 3 | E | **−100,50,−40** | −40,−20,20 | −8 / 8 | 0,−12,0 | lean −2, **twist −20** | the M arm winds **across the body**, palm edge leading (the hand passes the opposite shoulder), the body coils |
| 5 | **O** | **−86,−60,40** | −30,10,−20 | −14 / 10 | 0,10,0 | lean 6, **twist 24** | the **slice**: the arm sweeps horizontally across the view in 2 ticks, palm edge out; the blade leaves at tick 5 |
| 8 | S | −80,−50,36 | −30,10,−20 | | 0,6,0 | lean 4, twist 16 | hold |
| 12 | S | `AIR_REST` | `AIR_REST` | | 0 | lean 0, twist 0 | |

M4 (two blades): the O arm slices the other way at tick 7 (−86,60,−40 with twist −20): a V. M8 (three): the M arm
slices again at tick 9 (dur 14). **Air Slice, `AIR_SLICE`** (dur 14 / 1 / 5, force): a **full spin**: t 2 both arms out
level (−80,0,±60), t 10 **`yaw(360)`** with the arms held out and drop 0.06, legs alternating (−16/16 at 4, 16/−16 at 8),
lean 4; t 14 `AIR_REST`, yaw 360 (one turn). The sweep hits at tick 6 (the server's hit tick).

**Lock:** none. **Blend:** walking continues (the slice's step is 2 ticks); the spin rides over a walk (yaw is visual,
the walk cycle keeps running); flying (Creative/True Flight): fine. Under water: refused. **Started by:** cooldown edges
(1.5 s vs 3 s). **Prediction:** always.

**First person:** t 3 `M dx −0.34, dy +0.10, dz +0.06, ry −50, rz +15` (the hand crosses to the far left edge); t 5 `M dx
+0.36, dy +0.06, dz +0.20, ry +40, rz −20` (the sweep across: the hand crosses the whole lower screen in 2 ticks); hold;
rest. Second blade: the O hand sweeps the other way at 7. Slice: both hands at the edges `dx ±0.36, dy +0.02, dz +0.10`,
the **camera yaws 360°** over 8 ticks (the only gesture that turns the first-person camera: `ViewportEvent` yaw += the
gesture's yaw track; **reduced motion: no camera yaw, the hands sweep alone**).

**Effects:**

| Where | What | Count |
|---|---|---|
| Wind-up | a 4-point ribbon trailing the M hand (white α 0.25) | 1 ribbon |
| Blade | an `AirBladeEntity` drawn as a **crescent**: a translucent `ENERGY_VEIL` arc (an `annulus` segment of 100°, outer r 0.9, inner 0.7, tilted to the throw plane) with a bright thin edge ribbon (`ENERGY_BEAM` white α 0.7, width 0.03) and a faint motion blur (the last 3 positions as fading arcs α 0.3/0.15/0.05); 1 white mote/tick | 1/tick |
| Cut | on a creature: 6 white motes + a 0.4 s **slash line** (an `ENERGY_BEAM` ribbon across the hit point α 0.6 → 0); on plants: vanilla block-break particles of the plant (`levelEvent 2001`) | 6 |
| Slice | a horizontal **disc of wind** (`FxShapes.annulus` r 0 → 3 over 4 ticks, white α 0.3 → 0, `ENERGY_VEIL`) + 12 white motes on the ring + grass sways (`GUST` ×4) | 16 |
| Cap | 3 blades, 20 motes | |

**Sounds:** `air_blade_cast` (a short slicing whoosh with a high "shing" of air), `air_blade_fly_loop` (a thin whistle
following the blade, volume 0.3), `air_blade_hit` (a crisp cut with a thump), `air_slice` (a spinning whoosh rising
then falling, 0.7 s).

### 4.5 Spiritual Projection — השלכה רוחנית

**Sit, `AIR_MEDITATE_DOWN`** (dur 16 / 2 / 6, CAST, not force; **locks movement** via `Live.MEDITATING` for the whole
meditation): t 4 arms gather to the chest (−70,∓40,∓10, `E`), drop 0.10, head 6; t 8 the body **sinks** (drop 0.40,
legs x −40/−40 z ±30: the knees fold out), arms lower to the knees (−30,∓30,±40); t 16 **`AIR_SIT`** (drop 0.72, legs x
−90 z ±55, hands resting palms up: −20,±30,∓40), head 4, lean 0: hand-over to `AIR_MEDITATING` (10.5: a breathing sway
±1.5° lean at 0.05 rad/tick, the hands rising 2° on the inhale). **The spirit leaves, `AIR_SPIRIT_OUT`** (dur 10, on the
**spirit entity**, 10.6): the spirit copy stands up out of the body (drop 0.72 → 0 over 10 ticks with `O`, arms spreading
to −40,0,±40) while the body stays seated. **Return, `AIR_SPIRIT_IN`**: the body's head lifts (head −6 → 0 in 4 ticks) and
the hands open (y ±30 → ±50) for 6 ticks; then `AIR_MEDITATE_UP` (dur 10): the reverse of sitting. **Deep Meditation**
(Sneak): the sit alone, with a soft blue glow. **Snap-back on damage:** `AIR_MEDITATE_UP` plays at ×1.5 rate (`Slot.rate`)
with a flinch (`HIT_STAGGER` layered, group HIT).

**Lock:** input locked while meditating (the body cannot move); the spirit is controlled instead (`TECH.md` 3: the
client's camera entity is the `SpiritEntity`; W A S D steer it). **Blend:** sitting replaces every locomotion layer
(`wAir`, `wRun`, `wSprint` forced to 0 while `MEDITATING`: **NEW** 14.3 "stance owns the body" flag). **Started by:**
`Live.MEDITATING` rising/falling; `Live.PROJECTING` for the spirit. **Prediction:** on the press when on the ground and
ready.

**First person:** on the way down the camera lowers with the body (the eye height follows `drop`: vanilla handles a
sitting pose's eye height only for vehicles; **NEW** 14.4: an eye-height offset bound to `MEDITATING` = −0.9 blocks, eased
over 16 ticks); the hands rest low and open at the bottom corners (`dx ±0.30, dy −0.30, dz +0.10, rx +40, ry ∓30`). When
the spirit leaves, the view **lifts out of the body**: the camera entity becomes the spirit (the transition: a 10-tick
camera move from the body's eyes to the spirit's, with the overlay 2.2 fading in). In the spirit the hands are the
spirit's: translucent blue, drawn with `entityTranslucent` and the spirit tint (`#9FDCFF` α 0.45), floating a little
higher than normal (`dy +0.06`, bob ±0.02 at 0.1 rad/tick).

**Effects:**

| Where | What | Count |
|---|---|---|
| Sitting down | 6 `CLOUD`-like white motes at the ground; a dust ring | 6 |
| Meditating body | a faint **blue aura**: `PowerAuraLayer`-style `energySwirl` over the inflated model, `#7FD0FF` α 0.18 slowly scrolling; 1 `END_ROD` (small, white-blue) rising from the head every 10 ticks; the **tether**: an `ENERGY_BEAM` thread from the body's forehead to the spirit (`HandTracker.forehead`), `#9FDCFF` α 0.25, drawn as a sagging `polyline` (a catenary of 12 points), visible within 32 blocks | 0.1/tick + 1 thread |
| The spirit | the player's own model drawn **translucent blue** (Enhanced's `AstralBodyRenderer` / `AstralLayer` pattern: `entityTranslucent` with the spirit tint, no armour, no held items, name tag faint), legs folded (the spirit floats in the sit pose with arms free: a stance `SPIRIT_FLOAT` = `AIR_SIT` legs, arms at −30,∓20,±30 drifting), a soft bob; 2 blue motes (`#9FDCFF`, scale 0.5)/tick trailing | 2/tick |
| Through walls (spirit's client) | `EntityGlow`: hostile mobs `#8FD3FF`, players white, villagers/animals green; **caves**: a client raycast grid every 10 ticks marks air pockets within 24 blocks as faint `ENERGY_VEIL` boxes (α 0.08, blue) — "search for caves" | ≤ 64 boxes |
| Snap-back | a white flash (overlay) + 12 blue motes at the body + 6 at the spirit's last spot, the thread **snaps** (drawn straight for 2 ticks then gone) | 18 |
| Deep Meditation | the aura only, plus 1 green-white mote (`#CFF5E0`) per 2 ticks rising when healing ticks | 0.5/tick |
| Cap | 1 spirit, 1 thread, 20 motes | |

**Sounds:** `spirit_sit` (a soft breath out and cloth settling), `spirit_leave` (a rising shimmer with a bell-like
inhale, 1.2 s), `spirit_loop` (a quiet, hollow wind with distant chimes, volume 0.3, for the spirit's client only),
`spirit_return` (the shimmer reversed, a soft "whum"), `spirit_snap` (a sharp intake + a crack), `spirit_meditate_loop`
(a slow breathing hush, volume 0.2).

### 4.6 Air Bubble — בועת אוויר

**`AIR_BUBBLE_ON`** (dur 10 / 1 / 4, CAST, not force): under water, both hands **circle around the head** (t 3: −150,∓30,±30
with `E` (hands up by the ears), t 6: −170,∓10,±10 (meeting over the head), t 10 rest) as the bubble forms. **Off:** the
hands brush down past the face (−120 → −40 in 6 ticks) as it pops. **Air Cushion:** no gesture (a passive); the landing
puff plays `LAND_SOFT` with a lighter dip (0.06).

**Lock:** none. **Blend:** swimming is vanilla's (Enhanced hands the body to vanilla when `getSwimAmount > 0`), so under
water the gesture plays only the **arms** when not swimming (treading water); while swimming the bubble simply appears
(no gesture). **Started by:** `Toggle.AIR_BUBBLE`. **Prediction:** in water and ready.

**First person:** hands rise beside the view `dx ±0.26, dy +0.24, dz +0.08, rx −30` then meet above `dx ±0.06, dy +0.34`;
the overlay 2.2 (the fog lifts) fades in over 6 ticks.

**Effects:** a **bubble** around the head: an `ENERGY_VEIL` sphere r 0.55 centred on the head (follows `head`), base α
0.10 rim 0.35, white-blue `#E8F6FF`, with 2 slow highlights (`EnergyGeometry.ring` arcs α 0.3) and 1 vanilla `BUBBLE`
particle per 4 ticks rising from its top; forming: 8 `BUBBLE` + a 0.1-radius pop ring; popping: 12 `BUBBLE` + `BUBBLE_POP`.
Friend's bubble (M6): the same on them. Cushion landing: 6 `CLOUD` + ring. **Cap:** 2 bubbles, 10 bubbles/s.

**Sounds:** `air_bubble_on` (a glassy "bloop" reversed into a hush), `air_bubble_loop` (a soft underwater hum with
breathing, volume 0.3), `air_bubble_off` (a pop), `air_cushion` (a soft "fwoomp").

### 4.7 Air Scooter — קטנוע אוויר

**Mount, `AIR_SCOOTER_MOUNT`** (dur 10 / 1 / 4, group RIDE): t 3 both arms **spin down and around** (−40,∓50,∓30 → the
hands sweep low in a circle, `E`), drop 0.10; t 6 (`O`) a **hop**: drop −0.2 (up), legs fold (x −50/−50, z ±30: the ride
crouch), arms out for balance (−30,0,±40); t 10 hand-over to `AIR_RIDING` (10.1). **Dismount:** `AIR_SCOOTER_DISMOUNT`
(dur 8): a hop off (drop −0.1 then `LAND_SOFT`), arms down. **Air Spout, `AIR_SPOUT`** (hold; `AIR_SPOUT_UP` dur 10 /
1 / 4, then the stance `AIR_SPOUTING` while `Live.AIR_SPOUTING`): both palms **press down** beside the hips (20,∓10,±30 at
tick 4, `O`) as the column lifts you (the server moves you up 1 block per 2 ticks); the stance: arms held down and out
(10,0,±40) with a drift ±4 at 0.3 rad/tick, legs hanging (x 10/10), head level, lean 0; release: `LAUNCHED` → `LAND_SOFT`.

**Lock:** riding uses a vehicle entity (`AirScooterEntity`), so vanilla's riding pose is **replaced** by the stance
(Enhanced hands the body to vanilla when `isPassenger`; **NEW** 14.3: a `rideOverride` set of vehicle types the animator
keeps control of). **Started by:** the mount/dismount edges of the vehicle (`isPassenger` of a scooter); `Live.AIR_SPOUTING`.
**Prediction:** the mount on the press when ready and on the ground.

**First person:** mount: hands sweep low `dy −0.20, dx ±0.20`, then rise to the riding hands: slightly out and down,
palms down (`dx ±0.22, dy −0.10, dz +0.10, rx +25`), **bobbing** with the ball (±0.03 at 0.5 rad/tick, scaled by speed) and
leaning into turns (`dx` shifts ∓0.06 with `turn`). FOV ×1.12, camera roll 4° into turns, a slight forward tilt (pitch
+3°) at full speed. Spout: hands down `dy −0.26, rx +30` trembling 0.008.

**Effects (10.1 for the ride):** the **ball**: an `ENERGY_VEIL` sphere r 0.5 under the player (white `#F4F8FF`, base α 0.12,
rim 0.4) with **4 swirl bands** (`EnergyGeometry.ring` at tilted normals spinning 25°/tick, α 0.35, width 0.06) and the
air lattice texture scrolling fast (`energyField(air_lattice)`, α 0.2); 3 white `CLOUD`-like motes/tick kicked up behind
(more on dirt: `BLOCK` dust ×2 of the ground), a `MATTER_FLAT` dust streak on the ground (a lane 0.8 wide fading over 10
ticks); over water: `SPLASH` ×4/tick and a ripple ring; wall-ride: `BLOCK` dust from the wall. Mount: 10 `CLOUD` + ring.
Dismount burst (M10): the shield-burst sphere r 2 + 16 `CLOUD`. Spout: a **column** (`SandShapes.funnel`-like frustum r
0.8 → 0.5, height to the player, swirl texture `air_swirl.png` scrolling up, α 0.35) + 4 motes/tick along it + a ground
dust ring. **Cap:** 1 ball, 1 column, 30 motes.

**Sounds:** `air_scooter_mount` (a spin-up whoosh), `air_scooter_loop` (a continuous airy whir, pitch and volume
follow speed (0.8–1.3), volume 0.5), `air_scooter_dismount` (a whoosh down), `air_scooter_burst` (the shield burst,
shorter), `air_spout_loop` (a rushing updraft hum, volume 0.5), `air_spout_end` (the rush fading).

### 4.8 Tornado — טורנדו

**`AIR_TORNADO`** (dur 18 / 2 / 6, CAST, force): the Ba Gua **spin**:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `AIR_REST` | `AIR_REST` | | | | |
| 4 | E | −90,40,−20 | −90,−40,20 | −10 / 10 | −4 | lean −4, drop 0.06 | both arms level, hands close, palms facing: the funnel seeds between them (effects) |
| 10 | E | −100,0,30 | −100,0,−30 | 10 / −10 | 0 | lean 0, drop 0.08, **yaw 360** | a **full turn** with the arms opening (the hands trace the funnel's spiral) |
| 13 | **O** | **−96,−10,10** | −60,20,−30 | −18 / 12 | 2 | **lean 10**, twist 10 | the **send**: the M palm pushes forward, the funnel rolls away (server: the tornado entity spawns at 13) |
| 18 | S | `AIR_REST` | `AIR_REST` | | 0 | lean 0, twist 0, yaw 360 | |

Steering (M5, key held): the stance `AIR_STEERING` (aimed): the M arm points at the tornado (`aim(PROJECTILE, −8, 0,
0)`: the `PROJECTILE` target is the player's tornado entity) with a slow circling of the hand (r 6° at 0.25 rad/tick), O
braced. **Air Wheel, `AIR_WHEEL`** (dur 14 / 1 / 5, then the stance `AIR_WHEELING` for 4 s): t 3 arms down and back
(20,0,±40), a crouch (drop 0.16); t 6 (`O`) arms **sweep forward and up** to −70,∓20,±30 as the wheel forms at the legs;
the stance: arms held forward-low (−50,∓10,±24) with a lean of 14 (a runner's lean) and legs **blurred**: the run cycle at
×1.6 speed (the `RunCharacter` `AIR_WHEEL`: legs 0.4 amplitude (hidden in the wheel), arms 0.6, lean 14, bob 0.01).

**Lock:** none. **Blend:** the spin is visual yaw over a walk; flying: allowed. Under water: the whirlpool version
plays the same gesture at ×1.3 rate. **Started by:** cooldown edges (8 s vs 10 s), `Live.AIR_STEERING`, `Live.AIR_WHEELING`.
**Prediction:** always.

**First person:** t 4 both hands level, close, palms facing `dx ±0.08, dy +0.10, dz +0.26, ry ∓40` (a funnel of motes
spins between them); t 10 hands open `dx ±0.30` while the **camera yaws 360°** over 6 ticks (reduced motion: no yaw); t 13
`M dz +0.34, dx +0.04` push; rest. Steering: the M hand points `dz +0.28` circling 0.02. Wheel: hands forward-low `dy
−0.10, dz +0.20`, FOV ×1.12, the camera **bobs** ±0.04 at 0.8 rad/tick.

**Effects:**

| Where | What | Count |
|---|---|---|
| Seed (t 4–13) | a mini funnel between the hands: `SandShapes.funnel` r 0.1 → 0.3, h 0.4, white swirl texture α 0.4, spinning 40°/tick + 2 motes/tick | 2/tick |
| Tornado | the `TornadoEntity`: **two counter-rotating funnels** (Enhanced's tornado recipe, white): outer r 0.9 → radius·1.2 at the top, h = height, 28 seg × 8 rings, 3 turns, texture `air_swirl.png` α 0.45 → 0.2, scroll 0.06, twist 0.05; inner 0.6 → 0.8r, α 0.3, counter-rotating; **6 ring bands** (`EnergyGeometry.ring` α 0.2 at 6 heights, radii following the funnel, spinning 20°/tick); 8 `CLOUD` motes/tick on a helix + 4 `BLOCK` dust of the ground per tick at the base (grass, leaves, snow it crosses) + items/entities inside spin (server) | 12/tick |
| Lifted creatures | 2 white motes/tick each; their pose: `LIFTED`-tumble (12.2) | 2/tick each |
| End | the funnel **unravels** top-down over 8 ticks (rings fly off outward as 6 fading arcs) + 16 `CLOUD` | 16 |
| Firenado (suggestion) | the funnels tinted `#FF8A3A` with `FLAME` ×6/tick and `SMOKE` at the top | |
| Air Wheel | a **horizontal wheel** around the legs: 2 `EnergyGeometry.ring`s r 0.9 (normals ±Y, tilted 10° with speed) spinning 50°/tick, α 0.4 white + a translucent disc α 0.1; 4 motes/tick + ground dust streak; hits: 6 motes | 4/tick |
| Cap | 2 tornadoes, 1 wheel, 60 motes | |

**Sounds:** `tornado_cast` (a spinning whoosh that builds over 0.6 s into a roar onset), `tornado_loop` (a deep, wide
rotating roar with a whistling top layer, volume 0.7, follows the entity), `tornado_end` (the roar unwinding 0.8 s),
`air_wheel_loop` (a fast whirring rush, volume 0.5), `air_wheel_hit` (a thump).

### 4.9 Zaheer's Path — דרכו של זהיר

**The unlock moment** (`AIR_TETHERS_CUT`, dur 40 / 4 / 10, played once when the path opens, not force; locks input for
40 ticks): the bender **kneels** (t 10: drop 0.48, legs x −40/−40 z ±20, head 12, arms hanging 10,0,±16: grief), holds (t
10–24: a breath), then **rises into the air** (t 32: drop −0.6 (lifted), legs trailing x 20/20, arms spreading to
−40,0,±60, head −6, lean −4: the first flight, `B`), and settles (t 40) into `ZAHEER_HOVER` (10.6). Screen: a slow white
flash and the HUD thread icon appears.

**True Flight** (10.6 for the profile): take-off `AIR_TAKEOFF_ZAHEER` (dur 12): a small crouch (drop 0.12 at 3) then the
body **lifts straight** with arms sweeping down (−20,0,±50 → 20,0,±30 at 8, `O`) and legs trailing. Flight: the profile's
anchors (10.6). Hover (Sneak): `ZAHEER_HOVER`: cross-legged (`AIR_SIT` legs: x −90 z ±55), hands on the knees, a slow
spin (yaw 0.4°/tick) and bob (0.06 at period 60). Landing: `LAND_SOFT` with a feather dip.

**Suffocation, `AIR_SUFFOCATE_START`** (dur 10 / 1 / 4, then the stance `AIR_SUFFOCATING` while `Live.SUFFOCATING`): t 3
the M hand **reaches** at the target (aim(LOOK, 0, 0, 0), fingers spread y 20, `O`), the O hand low and back (10,0,−30);
t 6 the hand **closes slowly** (y 20 → −10, `E`) and **draws back** 6° in x: pulling the air; the stance: the M arm aimed
at the target with **tremble 1.4 at 1.8**, pulsing back 4° every 20 ticks (each pulse = the victim's gasp), the O arm
braced, head 4, lean 8 (leaning in), legs planted (±10). Release: `AIR_SUFFOCATE_END` (dur 8): the hand **opens** (y
−10 → 30, `B`) and drops. **Air Vacuum, `AIR_VACUUM`** (dur 14 / 1 / 5, force): both hands reach (−80,∓20,±10 at 3),
then **clench and pull in** to −60,∓50,∓20 (`I` at 6) with lean −8 (leaning back: pulling the air out of the sphere); hold
to 10; rest.

**Lock:** the unlock moment locks 40 ticks; suffocating locks nothing (you may walk slowly: speed ×0.5 attribute).
**Blend:** flying with the profile. **Started by:** the payload kind `TETHERS_CUT`; `Live.SUFFOCATING`; cooldown edges
(Vacuum 15 s). **Prediction:** suffocation on the press with a breathing creature in sight within range.

**First person:** tethers cut: the camera lowers (eye offset −0.5 over 10 ticks) then **rises** 1.2 over 8 ticks with
a white bloom; hands hang then spread wide `dx ±0.36`. Suffocation: `M dz +0.30, ry +25` (reach, fingers open) then `ry
−15, dz +0.22` (closing), trembling 0.012, pulsing `dz −0.04` every 20 ticks; the O hand low `dy −0.16`. Vacuum: both
`dz +0.26` then `dx ∓0.10, dz +0.14, ry ∓30` (clench in).

**Effects:**

| Where | What | Count |
|---|---|---|
| Tethers cut | a column of white light (`FxShapes.cylinder` r 0.6, h 6, α 0.4 → 0 over 20 ticks, `ENERGY_BEAM`) + 30 white motes rising + a ground ring; a **red thread** (an `ENERGY_BEAM` polyline `#B33A3A` from the chest to the ground, α 0.6) that **snaps** at tick 24 (two halves fall and fade) | 30 |
| True Flight | the flight profile's effects: a faint white **wake** (two `FlightEffects` ribbons from the feet, α 0.15, white, 1.2 blocks) + 1 `CLOUD`/tick at speed; take-off: a dust ring + 8 motes; hover: 1 mote per 10 ticks drifting around | 1/tick |
| Suffocation | a **thread of pulled air**: an `ENERGY_VEIL` ribbon from the victim's mouth to the caster's hand, white α 0.25, wavering (`flowingPath` jitter 0.1); 3 `CLOUD`-like motes/tick drawn **from the victim toward the hand** (spawned at the victim's head with velocity toward the hand); the victim's air meter bubbles (vanilla, server-set `airSupply`); the victim's pose 12.2 (clutching); every 20 ticks a `BUBBLE_POP`-like gasp mote at the victim | 3/tick |
| Vacuum | a 3-block **sphere of absence**: an `ENERGY_VEIL` sphere r 3 with inverted rim (α 0.0 centre → 0.3 rim, dark-blue tint `#1C2A40`) and 8 motes/tick flying **inward** to the centre; fires inside go out with `SMOKE` ×2 each; creatures inside show the gasp mote | 8/tick |
| Cap | 1 thread, 1 sphere, 30 motes | |

**Sounds:** `tethers_cut` (a slow rising chord with a snapping string, 2 s), `flight_takeoff` (a soft lift whoosh),
`flight_loop` (a gentle wind rush, volume follows speed 0.2–0.5), `suffocate_start` (a sharp inhale reversed), `suffocate_loop`
(a thin sucking wind with a heartbeat under it, volume 0.5; the victim hears their own choking: `suffocate_victim_loop`),
`suffocate_end` (a gasp of released air), `air_vacuum` (a deep implosive "thoomp" then silence for 0.3 s then a soft
rush back), `air_vacuum_loop` (a hollow pressure hum).

### 4.10 Air hit reactions

Pushed targets **tumble** (12.2); blade cuts flinch; suffocated targets clutch; tornado-lifted spin. An airbender who is
hit while in a spin finishes the spin (yaw gestures are never cut).

---

## 5. Water

### 5.0 Water idle: the ready stance

| Stance | tauIn / tauOut | Rows |
|---|---|---|
| `WATER_IDLE` (ambient) | 6 / 6 | `WATER_REST` plus Tai Chi **"wave hands like clouds"**: the M hand draws a vertical figure-eight (x −20 + 20·sin(0.07·age), y 10·sin(0.14·age), z 10 + 8·cos(0.07·age)), the O hand the mirror a half-phase behind; the weight shifts (lean 3·sin(0.035·age), legs x ∓8 following); head level; drop 0.06. |
| `WATER_SOURCE` (aimed, while `Live.WATER_DRAWING`: a source is being drawn, 3 ticks before any ability) | 1 / 3 | the M hand **reaches toward the source** (`aim(ANCHOR, 0, 0, 0)`: the anchor is the water block / the skin at the hip / the plant) for 3 ticks: a quick pull. |
| `WATER_OCTOPUS` (ambient while the octopus is up) | 3 / 5 | arms slightly out (−20,0,±30), elbows soft, a slow sway ±3; `occupy` none. |
| `WATER_ARMS` (ambient while arms are on) | 3 / 5 | 5.5. |
| `WATER_RIDING` (spout) | 2 / 4 | 10.3. |
| `WATER_HEALING` (aimed, hold) | 2 / 4 | 5.2. |
| `BLOOD_HOLD` (aimed) | 1 / 4 | 5.7. |

**Ambient particles of a master (Water ES ≥ 60):** 1 `bending:droplet` every 10 ticks lifting from a nearby water
source (within 4) and orbiting the hands once before falling; at ES 100 a thin water ring (`MATTER_FLAT` `#3C8DD6` α 0.15,
r 0.8) rotates slowly at the feet while standing still. Full moon: 2 extra `END_ROD`-like silver motes per 10 ticks.

**Water first-person idle:** both hands at the belly, palms up, slowly circling (`dx ±0.10, dy −0.10, dz +0.14, rx −20,
ry ∓20`, r 0.03 at 0.07 rad/tick); a droplet orbits them.

**The source draw (every Water ability starts with it):** 3 ticks before the ability's own keyframes, the M hand reaches
toward the source and a **water ribbon** flies from it to the hand (an `ENERGY_VEIL` ribbon `#3C8DD6` α 0.5, width 0.08,
`flowingPath` jitter 0.15 over 3 ticks, `MATTER` alpha (not additive), with 3 `bending:droplet`s along it). From the
**Skin**: the ribbon rises from the hip (the Skin item is drawn on the belt: `LOOKS.md` 6). From a **plant**: the plant
greys over 3 ticks (a `BLOCK` tint change via the temp state) and 4 droplets rise from it. Every table below starts at
the ability's tick 0 = after the draw; the draw is `WATER_DRAW` (dur 3 / 1 / 1, group CAST, not force) and the ability's
gesture starts as it ends (the gesture is started with `param = 3` delay, or the animator chains them: **NEW** 14.6 gesture
chaining).

### 5.1 Water Shot — יריית מים

**`WATER_SHOT`** (dur 14 / 2 / 5, CAST, force): Tai Chi "push hands":

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | E | `WATER_GATHER` | `WATER_GATHER` | −12 / 12 | 2 | lean −3, drop 0.10 | the water gathers at the belly between the palms (effects) |
| 5 | E | **−100,−20,20** | −60,30,−10 | −14 / 10 | 0 | lean 0, drop 0.08 | the M hand **rises in an arc** (wrist leading: y leads x), the O hand stays low: the stream rises with the hand |
| 8 | **O** | **−86,0,4** | −40,10,−20 | **−20 / 16** | 2 | **lean 8**, drop 0.06 | the **push**: the palm shoots forward, the weight shifts to the front leg; the shot leaves at tick 8 |
| 11 | S | −80,0,6 | −40,10,−20 | | 2 | lean 6 | |
| 14 | E | `WATER_REST` | `WATER_REST` | −6 / 6 | 0 | lean 0, drop 0.06 | |

M4 (two streams): the O hand pushes at tick 10 (−86,0,−4), lean 10. M8: a third from the M hand at 12 (dur 16). M10 lob:
the push aims up (`mAim(LOOK, −30, 0, 0)` at 8).

**Water Whip (Sneak, hold): `WATER_WHIP_START`** (dur 8 / 1 / 3) then the stance `WATER_WHIPPING` (aimed, tauIn 2 / out
4) while `Live.WATER_WHIPPING`: start: from the gather, the M arm **sweeps up and back** to `WATER_WHIP` (−70,40,−10: the
whip hangs behind the shoulder); the stance: the arm **lashes** every 10 ticks: a 4-tick swing from `WATER_WHIP` to
aim(LOOK, −10, 20, 0) (`O`) and a 6-tick return (`E`), the wrist leading (y swings 40 → −20 → 40), lean 4 → 8 → 4 with
each lash, the O hand balancing low; `occupy(M)`. End: the arm lowers and the whip falls (`WATER_WHIP_END` dur 8).

**Lock:** none. **Blend:** walking continues; the push's step drives the legs for 3 ticks; under water: the same at
×1.2 rate (water is the element). **Started by:** the slot's cooldown edge (1 s), `Live.WATER_WHIPPING`. **Prediction:**
a source in reach (the client checks: water within 8, a skin with charge, or M7+ with a plant) and ready.

**First person:** draw: the M hand dips toward the source `dz +0.30, dy −0.14` (3 ticks). Shot: t 0 both hands at the
belly `dx ±0.08, dy −0.12, dz +0.14, rx −20` (water pooling between them); t 5 `M dx −0.10, dy +0.18, dz +0.12, ry −30`
(the arc up and across); t 8 `M dz +0.34, dy +0.06, dx +0.04, rx −10` (push); recovery. Whip: the M hand cocks back and
high `dx +0.24, dy +0.26, dz −0.04, ry +30`, then lashes `dz +0.34, dx −0.08, ry −25` over 4 ticks, every 10 ticks; the
whip ribbon hangs from the hand (effects).

**Effects:**

| Where | What | Count |
|---|---|---|
| Gather (t 0–5) | a **water blob** between the hands: an `ENERGY_VEIL`... no, a `MATTER_TEXTURED(water_still)` sphere r 0.15 → 0.3 (vanilla's `block/water_still` animated texture, tinted `#3C8DD6`, α 0.7) + 2 droplets/tick orbiting | 2/tick + 1 sphere |
| Stream flight | the `WaterShotEntity` drawn as a **ribbon** of water: an `ENERGY_VEIL`-style ribbon chain through the last 8 positions (width 0.25 head → 0.08 tail, `MATTER_TEXTURED(water_flow)` α 0.75 with a white highlight ribbon α 0.4 on top, width 0.05) + 2 droplets/tick falling off + 1 `SPLASH`/tick | 3/tick |
| Hit | a **splash**: 14 `SPLASH` + 6 droplets + a `MATTER_FLAT` wet disc on the ground under the target (α 0.3, 40 ticks); the target **drips** (12.3) | 20 |
| Miss (block) | 10 `SPLASH` + a wet decal (`webDecal`-style quad with `wet.png`, α 0.4 → 0 over 60 ticks) | 10 |
| Vs fire | a **steam burst**: 10 `CLOUD` white + the fire's particles stop; a fireball hit: 16 `CLOUD` + a hiss | 10–16 |
| Whip | a **hanging rope of water**: a chain of 10 segments (verlet-ish: each follows the previous with gravity, client-simulated) from the hand, `MATTER_TEXTURED(water_flow)` ribbon width 0.14 → 0.08, α 0.75, highlight ribbon; droplets 1/tick from the tip; lash: the tip whips to the target (the chain's end is driven to the hit point over 4 ticks) with 8 `SPLASH` at the hit | 1/tick + 8 |
| Cap | 3 shots, 1 whip, 40 droplets | |

**Sounds:** `water_draw` (a liquid "shloop" rising, 0.3 s), `water_shot` (a liquid whoosh with a wet onset), `water_hit`
(a splash and a slap), `water_splash_block` (a splash), `water_vs_fire` (a hiss and a steam puff), `water_whip_swing`
(a wet whoosh), `water_whip_crack` (a liquid crack), `water_whip_loop` (a soft sloshing while it hangs, volume 0.25).

### 5.2 Healing — ריפוי

**`WATER_HEAL_START`** (dur 10 / 2 / 4, CAST, not force) then the stance `WATER_HEALING` while `Live.WATER_HEALING`: start:
from the gather the hands **rise together** to `WATER_HEAL` (palms down at chest height, −60,∓20,∓10) over 8 ticks (`E`),
head 6 (looking at the hands or the friend), lean 2, drop 0.08; the stance: the hands **hover** with a slow circular
stir (x ±4, y ±6 at 0.15 rad/tick, both hands in the same direction: Katara's circles), aimed at the friend (`aim(LOOK,
−10, 0, 0)` for the M hand when a friend is targeted, else both at the belly: self-healing hands press to the chest
(−70,∓30,∓20)), a gentle sway; `occupy` both. End: the hands lower (`WATER_HEAL_END` dur 8). **Cleanse (Sneak):**
`WATER_CLEANSE` (dur 10 / 1 / 4, force): a **splash**: both hands scoop from the belly (t 3: `WATER_GATHER`) and **throw
upward** at the target (t 6: `aim(LOOK, −20, ∓10, 0)`, `O`), lean 6; rest.

**Lock:** none, but healing breaks if either moves fast (server). **Blend:** the stance rides over standing; walking
slowly keeps it. **Started by:** `Live.WATER_HEALING`; cooldown edge (Cleanse 10 s). **Prediction:** a source in reach,
and a target below full health in sight (or yourself below full).

**First person:** hands rise to the centre, palms down, `dx ±0.10, dy +0.04, dz +0.20, rx +20, ry ∓15`, stirring (a
circle r 0.02 at 0.15 rad/tick); the glowing water pools between them (effects); self-healing: the hands turn inward
toward the camera `rx +50, dz +0.08` (palms toward you). Cleanse: scoop `dy −0.12` then throw up `dy +0.30, dz +0.24`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Hands | a **glowing water veil** on the hands: `MATTER_TEXTURED(water_still)` sphere r 0.2 per hand α 0.6 tinted `#8FE3FF` + an additive glow (`FxShapes.glow` r 0.3 `#CFF5FF` α 0.3, `ENERGY_BEAM`); 2 `END_ROD`-like glow motes (`#CFF5FF`, scale 0.4)/tick drifting up | 2/tick |
| The healed | a **water veil** over the target: `PowerAuraLayer`-style `energySwirl` with `water_flow` texture over the inflated model, `#8FE3FF` α 0.3 scrolling up; 3 glow motes/tick rising around them; a ring of 6 droplets orbiting at the waist; each heart healed: a brief white pulse of the veil (α +0.3 for 4 ticks) | 3/tick |
| Stream | an `ENERGY_VEIL` ribbon from the healer's hands to the target, `#8FE3FF` α 0.35, gentle (`flowingPath` jitter 0.05), with 2 droplets/tick flowing along it | 2/tick |
| Cleanse | a splash arc from the hands to the target (a ribbon over 4 ticks) and 12 `SPLASH` + 8 glow motes bursting at them; poison particles stop | 20 |
| Cap | 2 targets (M4), 30 motes | |

**Sounds:** `heal_start` (a soft liquid shimmer rising), `heal_loop` (a gentle bubbling with a warm pad underneath,
volume 0.4; a soft chime every heart healed: `heal_tick`), `heal_end` (the shimmer fading), `cleanse` (a splash with a
bright "pling" and a sigh).

### 5.3 Water Octopus — תמנון מים

**Form, `WATER_OCTOPUS_FORM`** (dur 20 / 2 / 6, CAST, not force): t 4 the gather (`WATER_GATHER`, drop 0.12); t 8 the arms
**circle outward and up** (−110,∓40,±30, `E`), the water rising around the body; t 14 both arms **sweep down and out** to
−30,0,±50 (`E`) as the arms of water spread; t 20 `WATER_OCTOPUS` stance hand-over. **Drop:** `WATER_OCTOPUS_DROP` (dur 10):
the hands **clap together** at the chest (−70,∓50,∓20 at tick 4, `O`) and the arms collapse into a puddle; rest. **Lash
(Sneak), `WATER_OCTOPUS_LASH`** (dur 12 / 1 / 4, force): both arms **whip forward** from the sides (t 3: −40,0,±60; t 6:
`WATER_PUSH` with `O` and lean 10; rest), every tentacle lashes with them.

**The attack strikes (auto, when you hit someone):** `WATER_TENTACLE_STRIKE` (dur 8, group HIT... no: group `CAST` would
cancel casts; a **new group `AUTO`** (**NEW**, 14.2) for reactive extra gestures): a small shoulder twist (twist ±10) as two
tentacles strike; the arms are not keyed (your punch plays).

**Lock:** none; speed ×0.85 (attribute). **Blend:** the octopus stance adds only a sway; walking, sprinting, jumping as
usual; under water: the arms are 8 at any mastery and the sway is larger (±6). **Started by:** `Toggle.WATER_OCTOPUS`;
payload kinds `OCTOPUS_LASH`, `TENTACLE_STRIKE`, `TENTACLE_SLAP` (a projectile slapped). **Prediction:** source in reach and
ready.

**First person:** form: gather `dy −0.12`, rise `dy +0.30, dx ±0.24, ry ∓30`, sweep down and out `dx ±0.34, dy −0.06, rx
+20`; the **tentacles are visible in first person**: the two nearest arms (the ones attached at the shoulders) are drawn
curling into the lower corners of the view (the octopus renderer draws all arms from the body's anchor points, which the
first-person camera sees from inside: the two front-lower arms sit at the screen's bottom edges, waving). Lash: both
hands sweep forward `dz +0.32`. Drop: hands clap at the centre `dx ∓0.02, dz +0.14`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Form | the water **climbs the body**: the octopus layer (`LOOKS.md` 1.3: a `MATTER_TEXTURED(water_flow)` body veil over the inflated model, α 0.55 `#3C8DD6`, plus **8 tentacles**: each a chain of 7 segments (`ENERGY_VEIL`-style ribbon quads with the water texture, width 0.22 root → 0.06 tip, α 0.7), rooted at the body (2 at each shoulder, 2 at each hip), **swaying** with a per-arm phase: `angle = base + 25°·sin(0.12·age + i·0.8)` on two axes, tips curling (the last 3 segments bend 30° more)) rises over 14 ticks (the veil's clip plane bottom-up; tentacles grow from root to tip 1 segment per 2 ticks); 4 droplets/tick | 4/tick + layer |
| Standing | 1 droplet per 2 ticks drips from a tip; the sway; under water the tentacles move slower and wider | 0.5/tick |
| Slap | the nearest tentacle **snaps** to the projectile (its tip driven to the hit point over 2 ticks, then back over 6) with 6 `SPLASH` | 6 |
| Strike (on your hit) | two tentacles snap to the target with 8 `SPLASH` and a wet disc | 8 |
| Lash | all 8 tentacles whip forward in a fan (tips to the cone's edge, 4 ticks) + 16 `SPLASH` | 16 |
| Grab (M10) | two tentacles wrap the creature (their last 4 segments bend around its box) for 20 ticks | |
| Drop | the layer collapses in 6 ticks (clip top-down) into a **puddle**: a `MATTER_FLAT` disc r 1.2 α 0.5 fading over 40 ticks + 20 `SPLASH` | 20 |
| Fire on it | a tentacle **evaporates**: 10 `CLOUD` and it shrinks tip-first over 10 ticks | 10 |
| Freeze on it | the tentacles **stiffen** (sway → 0 over 2 ticks) and tint `#BFE9FF` with frost lines (`ENERGY_BEAM` white cracks α 0.5) for 2 s | |
| Cap | 1 octopus (8 chains), 40 splashes | |

**Sounds:** `octopus_form` (a surging swell of water rising around, 1 s), `octopus_loop` (a soft lapping and sloshing,
volume 0.35, pitch 0.9), `octopus_slap` (a wet slap), `octopus_strike` (two quick wet thwacks), `octopus_lash` (a wide
liquid whip-crack), `octopus_drop` (a collapse into a splash), `octopus_steam` (a hiss), `octopus_freeze` (a crackling
freeze).

### 5.4 Water Spout — עמוד מים

**Rise, `WATER_SPOUT_UP`** (dur 16 / 2 / 5, group RIDE) then the stance `WATER_RIDING` (10.3) while `Live.WATER_SPOUTING`:
t 4 the gather with a deep knee bend (drop 0.18, legs ±10); t 8 both arms **spiral upward** (−140,∓30,±20 with y
sweeping 30 → −30, `E`): the column rises under the feet (server: the spout entity lifts you 0.5 block/tick); t 12 arms
settle out and down (−40,0,±40) for balance, legs **slightly bent and apart** (x −10/−10, z ±12: standing on water); t
16 hand-over. The ride (10.3): arms out and low, swaying with the column's spin (twist ±6 at 0.2 rad/tick), the body
leaning into steering (`lean 8·fwd`, `roll 6·side`), a bob (0.04 at period 16). **Ram:** `WATER_SPOUT_RAM` (group AUTO, dur
8): a forward lean kick (lean +14 for 3 ticks, `O`). **Collapse:** `LAUNCHED` → the fall → `LAND_SOFT` with a big splash.
**Wave Ride (Sneak), `WATER_WAVE_RIDE`** (dur 12 / 1 / 4, then the stance `WATER_SURFING` for the ride): both arms **sweep
back** (20,0,±40 at 3) then **forward and up** (−90,∓10,±20 at 6, `O`) as the wave rises behind; the surf stance: a
surfer's crouch (drop 0.16, legs x −20/−20 z ±14: sideways: the body yaws 30° to the travel direction via `twist 30`),
arms out (−40,∓20,±50), lean 6. **Freeze (M8, Sneak while riding):** the column freezes: the ride stance stops swaying (the
`tauOut` 4 brings it still) and a stomp gesture (`EARTH_WALL`'s stomp keys) plays.

**Lock:** the spout is a vehicle (`WaterSpoutEntity`): vanilla's riding pose is overridden (14.3). **Blend:** flying
no; swimming no (you ride). **Started by:** mount/dismount edges, `Live.WATER_SPOUTING`, `Live.WATER_SURFING`, payload
kind `SPOUT_RAM`. **Prediction:** standing in/by water (or a skin at M5+) and ready.

**First person:** rise: gather `dy −0.14` then both hands spiral up `dy +0.34, dx ±0.14, ry ∓40` (the camera rises with
the body: smooth interpolation of the vehicle), then settle out `dx ±0.28, dy −0.08, rx +15`; riding: the hands sway with
the spin (`dx ±0.03 sin`), the camera **rolls** 3° with steering and bobs 0.04; FOV ×1.05; water spray at the bottom of the
screen (droplet particles spawned in front of the camera at the column's top ring: 2/tick, visible as flying drops).
Ram: a 3-tick forward lurch (`dz +0.06` of both hands, camera pitch +2). Surf: hands out `dx ±0.30, dy −0.04`, the camera
yawed 30° from travel (the body's twist: the first-person camera keeps the look; only the hands shift `dx` by +0.08).

**Effects:**

| Where | What | Count |
|---|---|---|
| The column | the `WaterSpoutEntity` renderer: a **twisted frustum** (`SandShapes.funnel` with `water_flow` texture, r 1.2 at the base → 0.7 at the top, 3 turns, scroll up 0.08/tick, twist 0.06, α 0.75 `#3C8DD6`, `MATTER_TEXTURED`) with a white **foam ring** at the top (`EnergyGeometry.ring` width 0.12 α 0.6) and at the base (r 1.6, α 0.5); **spiral highlight ribbons** (2 `ENERGY_VEIL` helices α 0.4, width 0.08, white); 6 droplets/tick flung outward from the top ring + 4 `SPLASH`/tick at the base + 2 `BUBBLE` rising inside | 12/tick |
| Over land | the base loses the splash; a wet trail (`MATTER_FLAT` disc chain α 0.3, 60 ticks) | 1 disc/4 ticks |
| Ram | 20 `SPLASH` at the target + a foam burst ring | 20 |
| Collapse | the frustum **falls** (its height → 0 over 6 ticks, α 0.75 → 0) into a 3-block **splash disc** (`FxShapes.disc` r 0 → 3 α 0.6 → 0 over 8 ticks) + 30 `SPLASH` + 10 droplets | 40 |
| Frozen (M8) | the frustum turns to **ice**: texture swap to `ice` (translucent `#BFE9FF` α 0.8), the spin stops, frost lines, 6 `SNOWFLAKE`; it is a temp block pillar for 6 s then shatters (16 `BLOCK` ice dust) | 6 / 16 |
| Wave Ride | a **wave**: a `SandShapes.ridge` lane in water (crest 1.1, width 3, length 6, the `water_flow` texture, α 0.8) running ahead of the rider with a foam crest ribbon (white α 0.6) + 8 `SPLASH`/tick + 4 droplets | 12/tick |
| Cap | 1 spout, 1 wave, 60 splashes | |

**Sounds:** `spout_rise` (a rushing surge rising in pitch, 1 s), `spout_loop` (a roaring, churning water column with a
whirl, volume 0.7, pitch with height 0.9–1.1), `spout_ram` (a heavy splash-thud), `spout_collapse` (the column
dropping into a broad splash), `spout_freeze` (a crackling freeze with a deep creak), `wave_ride` (a long rolling wave
with foam hiss, loop `wave_loop` volume 0.6), `wave_end` (the wave breaking).

### 5.5 Water Arms — זרועות מים

**On, `WATER_ARMS_ON`** (dur 12 / 2 / 4, CAST, not force): the gather at 3; at 6 both arms **thrust forward** (−80,∓10,±6,
`O`) and the water **runs down them** from the shoulders to the hands; at 9 the arms **flex wide** (−60,∓30,±40, `B`: the
new arms test their reach); at 12 the `WATER_ARMS` stance: arms slightly forward and out (−30,∓10,±24), **always moving**:
a slow serpentine (x ±6, y ±8 at 0.2 rad/tick, each arm in counter-phase: Ming-Hua's arms never rest), `occupy` none (the
vanilla punch plays as a lunge: the arm's punch swing is kept, and the water arm **extends** to the reach: effects). **Off:**
the arms **shake off** the water (t 3: −40,∓20,±50 with tremble 3 at 3.0; t 8 rest) and it splashes down. **Grab (Sneak),
`WATER_ARM_GRAB`** (dur 12 / 1 / 4, force): the M arm **shoots out** (aim(LOOK, 0, 0, 0), `O` at 3), **closes** (y −20 at 5),
and **hauls back** (−40,30,20 at 8, `I`, lean −10: the body leans back with the pull); rest. **Climb (M4, Space at a wall):**
`WATER_ARM_CLIMB` (dur 10): both arms reach up (−160,∓10,±10) and the body is pulled up 4 blocks (server velocity), arms
sweeping down to −60 as you rise (the pull). **Swing (M7):** the same to a ceiling, then the `LAUNCHED` stance.

**Lock:** none. **Blend:** the stance's serpentine adds to walking arms (ADD rows, so the swing shows through); sprinting
is fine; swimming: vanilla (the arms are drawn still extended on the vanilla pose). **Started by:** `Toggle.WATER_ARMS`;
cooldown edges (grab 4 s); payload kinds `ARM_CLIMB`, `ARM_SWING`. **Prediction:** source in reach and ready.

**First person:** on: gather `dy −0.12`, thrust `dz +0.30`, flex `dx ±0.30`; the arms are **drawn as water** from the
elbows down: the first-person arm render is replaced (`RenderArmEvent` override: the arm's texture becomes the water
texture with the highlight ribbon, slightly longer (scale z 1.3), translucent) and the hands are water claws (a `ModelPart`
of three water "fingers"); the serpentine is a slow `dx ±0.03, dy ±0.02` wander. Items in hand are hidden. Punching: the
hand shoots forward (`dz +0.40` for 2 ticks, beyond the normal swing) and the water arm's extension stretches toward the
target (the water arm mesh extends to the hit point). Grab: `dz +0.40, ry −20` then haul back `dz −0.10, dx +0.16`.

**Effects:**

| Where | What | Count |
|---|---|---|
| On | water runs down the arms: 6 droplets/tick for 6 ticks sliding from the shoulders; the **water arm layer** (`LOOKS.md` 1.3): each arm wrapped in a `MATTER_TEXTURED(water_flow)` sleeve (an inflated arm part, `CubeDeformation(0.25)`, α 0.65 `#3C8DD6`) plus an **extension**: a tapered ribbon chain from the hand to `reach` blocks along the arm's direction (width 0.18 → 0.05, 6 segments with a lazy S-curve (sin along the length ±0.15), α 0.7) ending in a **hand of water** (3 short ribbons as fingers); highlights; 1 droplet per 2 ticks from the tips | 6/tick then 0.5/tick |
| Punch | the extension **snaps straight** to the target over 2 ticks (its end driven to the hit point) and relaxes over 6; 8 `SPLASH` at the hit | 8 |
| Grab | the extension wraps the target (the last 2 segments bend around its box) for 4 ticks and **drags**; 6 `SPLASH` | 6 |
| Climb / swing | both extensions shoot up to the wall/ceiling (anchored: `aim(ANCHOR)`), the body is pulled; 6 droplets at the anchor | 6 |
| Off | 16 `SPLASH` + a puddle disc | 16 |
| Freeze on them | the sleeves tint ice `#BFE9FF` and stop moving 2 s; frost lines | |
| Fire on them | the extensions shrink tip-first with 8 `CLOUD` | 8 |
| Cap | 1 pair, 30 splashes | |

**Sounds:** `water_arms_on` (a surge running down with a wet flex), `water_arms_loop` (a quiet continuous slosh,
volume 0.25), `water_arm_strike` (a wet snap-thwack), `water_arm_grab` (a wet grip and a drag), `water_arm_climb` (a
liquid pull with a creak), `water_arms_off` (a shake-off splash).

### 5.6 Ice — קרח

**Ice Knives, `WATER_ICE_KNIVES`** (dur 12 / 1 / 4, CAST, force): the gather (t 0, `WATER_GATHER`); t 3 the M hand **rises
and freezes the water** (−70,30,−10 with a 2-tick **hold**: the knives crystallise, a `tremble 1.0 at 4.0` for 2 ticks: the
frost shiver); t 6 (`O`) a **flick** forward and out (−88,−30,24, lean 8, twist 12), knives fly at 6, 7, 8; t 12 rest. M6+:
the O hand flicks at 8 too. **Ice Spikes (Sneak), `WATER_ICE_SPIKES`** (dur 14 / 1 / 5, force): the gather; t 4 both palms
**press down** toward the aimed ground (`aim(LOOK, 30, ∓10, 0)`, `E`); t 7 (`I`) both hands **jerk up** 20° (−70) as the
spikes **erupt** (server: spikes rise 1 block per tick from 7); lean −6, drop 0 → −0.04 (a hop); t 14 rest.

**Freeze, `WATER_FREEZE`** (dur 14 / 1 / 5, force): the gather; t 4 the M arm **sweeps out level** over the water
(`WATER_WAVE`: −110,−30,30 / −50,30,−20, `E`); t 7 (`O`) the arm **drops and stiffens**: −80,0,20 with the hand flat (y 0),
**tremble 1.2 at 4.5** for 3 ticks (the freezing shiver through the arm), the body **sinks** (drop 0.14) and stiffens (lean
2): the ice spreads from the aim point at tick 7 outward 1 block per tick; t 14 rest. On a creature: the same, aimed at
its feet. **Snow Throw (Sneak), `WATER_SNOW_THROW`** (dur 10 / 1 / 4, force): the gather at the belly (snow forms: effects);
t 4 both hands **scoop up** (−100,∓20,∓10, `E`); t 6 (`O`) **fling** forward (`WATER_PUSH`, lean 8): the snowballs fly in a
spread at 6; t 10 rest.

**Ice Wall, `WATER_ICE_WALL`** (dur 14 / 1 / 5, force): the gather; t 4 both arms **sweep across** low (−40,∓40,∓20 with
y crossing: the hands pass each other, `E`), drop 0.14; t 7 (`O`) both arms **rise fast** to −130,∓10,±20 with the frost
shiver (tremble 1.4 at 4.5, 3 ticks), lean −4: the wall **shoots up** at the midpoint; t 10 hold; t 14 `WATER_REST`. **Ice
Dome (Sneak), `WATER_ICE_DOME`** (dur 14 / 1 / 5): t 4 arms gather; t 7 both arms **rise and arch overhead** (−170,∓20,±20
with the shiver) and **sweep down the sides** to −60,0,±60 at 10 (`E`: tracing the dome); rest.

**Lock:** none. **Blend:** walking continues (the freeze's sink drives no legs); under water: Freeze aimed up freezes the
surface above you. **Started by:** cooldown edges (2 s / 6 s; 4 s / 2 s; 8 s / 15 s), each pair told apart by length as
Thanos's are; `Live` none. **Prediction:** source in reach (ice from the Skin or water) and ready; Freeze: water or a wet
creature in sight.

**First person:** knives: gather `dy −0.12`; the M hand rises `dy +0.14, dz +0.12, ry −20` and **shivers** (tremble 0.015 for
2 ticks) as three ice knives appear fanned on the fingers (effects); flick `dz +0.32, dx +0.12, ry +20`. Spikes: hands press
down `dy −0.16, dz +0.22, rx +35` then jerk up `dy +0.06`. Freeze: the M hand sweeps across `dx −0.24 → +0.20, dy +0.08`
then drops flat `dy −0.06, dz +0.24, rx +30` and shivers; the frost spreads from the aim point. Snow: scoop `dy +0.20, dx
±0.10, ry ∓30` then fling `dz +0.30`. Wall: cross low `dx ∓0.16, dy −0.12` then rise `dy +0.34, dx ±0.12` with the shiver.
Dome: rise overhead `dy +0.40, dx ±0.06` then sweep down and out `dx ±0.36, dy −0.10`; the dome's inside is seen (ice
walls around the view).

**Effects:**

| Where | What | Count |
|---|---|---|
| Knives form | 3–7 **ice knives**: thin boxes 0.5 long (`MATTER_FLAT` ice `#BFE9FF` α 0.85 with a white edge) appearing on the fingers with 3 `SNOWFLAKE` each and a frost **crackle** (an `ENERGY_BEAM` white bolt 0.3 long, 2 ticks) | 3 each |
| Knife flight | the `IceKnifeEntity`: the box spinning end over end 30°/tick, a faint sparkle trail (1 `SNOWFLAKE`/2 ticks) | 0.5/tick |
| Knife hit | the knife **shatters**: 10 `BLOCK` dust of `ice` + 4 `SNOWFLAKE`; the chilled target gets frost at the hit (12.3) | 14 |
| Spikes | each spike is a temp block `bending:ice_spike` (a pointed model, 3 states of rise: 0.33 / 0.66 / 1.0 height, cycled at ticks 0, 1, 2 of each spike) with 6 `BLOCK` ice dust and 2 `SNOWFLAKE` on eruption; a frost **crack line** runs along the ground ahead (an `ENERGY_BEAM` white polyline α 0.5, 2 ticks ahead of the spikes); melting: 4 `DRIPPING_WATER` each | 8 per spike |
| Freeze (water) | ice spreads: temp `frosted_ice`-look blocks placed outward in rings 1 block/tick, each with 3 `SNOWFLAKE` and a frost crackle bolt; a white **frost ring** (`MATTER_FLAT`, r growing with the ice, α 0.4 → 0) | 3 per block, capped 30/tick |
| Freeze (feet) | the victim's feet are **encased**: a `MATTER_FLAT` ice block mesh (0.9 × 0.6 × 0.9) at their feet with frost lines, 8 `BLOCK` ice dust; it shatters on release (12 dust) | 8 |
| Freeze (constructs) | frost lines over the construct (`ENERGY_BEAM` white cracks α 0.5 over its surface) and a tint shift to `#BFE9FF` | |
| Snow | 4+M `ITEM_SNOWBALL`-trailing projectiles (vanilla snowball render), impact 6 `SNOWFLAKE` + `ITEM_SNOWBALL` ×4; the settled snow (M6): temp `snow` layers with 10 `SNOWFLAKE` | 10 |
| Ice Wall | rows of temp **ice** blocks (vanilla ice texture, translucent) rise bottom-up over 3 ticks, each block with 3 `SNOWFLAKE` + a frost crackle; a white frost **bloom** on the wall's face (an `ENERGY_BEAM` branching bolt tree across the face α 0.6 → 0 over 6 ticks); standing: 1 `SNOWFLAKE`/20 ticks; melting (fire): `DRIPPING_WATER` ×4 per block + `CLOUD` ×6 (steam); end: blocks fade to `DRIPPING_WATER` ×2 | 4 per block |
| Ice Dome | a hemisphere of temp ice blocks (r 2–3, hollow) closing **top-down**... bottom-up over 4 ticks; inside: the light is blue-tinted (the ice blocks do that) | 4 per block |
| Cap | 8 knives, 1 spike line (≤ 15), 1 ice disc (≤ 169 blocks at M10: placed over 13 ticks), 3 walls, 1 dome, 60 snowflakes | |

**Sounds:** `ice_knives_form` (three crystalline "tinks" with a frost hiss), `ice_knife_throw` (a thin icy whistle),
`ice_knife_shatter` (a glassy shatter), `ice_spikes` (a cracking eruption: a rapid series of ice cracks rising in
pitch), `freeze_cast` (a sharp frost hiss sweeping outward with crackles, 0.8 s), `freeze_feet` (a crunching encasement),
`ice_shatter` (glassy break), `snow_throw` (a soft fluffy whoosh ×4), `snow_hit` (a soft pat), `ice_wall_rise` (a fast
crystalline growth: crackles rising into a resonant ring), `ice_wall_break` (shatter), `ice_melt_loop` (a hissing drip,
volume 0.3), `ice_dome` (the wall sound, longer, with a hollow boom as it closes).

### 5.7 Blood Bending — כשפות דם

**Grab, `BLOOD_GRAB`** (dur 16 / 2 / 5, CAST, force; **locks movement** via `Live.BLOODBENDING` while held) then the
stance `BLOOD_HOLD` (aimed, tauIn 1 / out 4) while `Live.BLOODBENDING`:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | E | `WATER_REST` | `WATER_REST` | | 0 | | |
| 4 | E | −50,20,10 | −50,−20,−10 | 0/0 (±8) | 4 | **lean 8**, drop 0.08 | the hands rise slowly, fingers curling (y in), the body **hunches** |
| 8 | E | **`BLOOD_CLAW`** | **`BLOOD_CLAW`** | (±8) | 6 | **lean 14**, drop 0.12 | both claws forward, **tremble 3.0 at 2.2** begins (Hama's hands) |
| 12 | **I** | aim(LOOK, −10, 15, 0) with tremble 3 | aim(LOOK, −10, −15, 0) with tremble 3 | (±8) | 8 | lean 16, drop 0.14 | the claws **clench** toward the target (y pulls in 12°): the victim is seized at tick 12 and lifted |
| 16 | S | `BLOOD_HIGH`-M aimed | `BLOOD_CLAW`-O aimed | (±8) | 6 | lean 14, drop 0.12 | hand over: one claw high (the puppet hangs from it), the other low |

The stance **`BLOOD_HOLD`**: the M claw aims at the victim (`aim(HELD, −20, 10, 0)`: the held entity), raised when the
victim is high; the O claw at the victim's feet (`aim(HELD, 30, −10, 0)`); both tremble 3.0 at 2.2; **the hands move the
puppet**: the bender's look drives the victim's position (server), so the claws follow it; a slow alternating twist (±6 at
0.1 rad/tick: the puppeteer's shoulders); head 6, lean 14, drop 0.12, legs planted ±8; `occupy` both. **Release:**
`BLOOD_RELEASE` (dur 10): the claws **open** (y → ±40, `B`) and drop, the body straightens (lean 14 → 0 with `E`). **Fling,
`BLOOD_FLING`** (dur 12 / 1 / 4, force): from the hold, both arms **whip** from the claws to aim(LOOK, −20, ∓30, 0) with
twist 20 and lean 4 → 18 (`I` at 4), a half-step (legs −14/14); rest at 12. **Severance, `BLOOD_SEVER`** (dur 30 / 2 / 8,
force; locks 30 ticks): the victim is drawn to within 1 block (the hold); t 6 the M claw **rises** to the victim's forehead
(`aim(HELD, −45, 0, 0)` → the forehead), the O claw to its chest (`aim(HELD, 0, 0, 0)`), both trembling; t 14 (`I`) the M
**thumb presses** (x −6, a 2-tick push forward) and **holds** (t 14–24: the hands go **still**: tremble 0 — the one
moment a blood bender is still), head 8 looking into the victim's eyes, lean 18; t 26 the M hand **draws back** (`O`) as
the victim's limbs **sag**; t 30 release into `WATER_REST`. The victim's pose: 12.4.

**Lock:** the bender cannot move while holding (`BLOODBENDING` input lock); the victim cannot move (`BLOODBENT` lock).
**Blend:** none needed (standing). Under water: the same. **Started by:** `Live.BLOODBENDING` (grab), its fall (release),
payload kinds `BLOOD_FLING`, `BLOOD_SEVER`. **Prediction:** a target in sight within range on a full moon (or M10): the
client knows the moon phase and its own mastery.

**First person:** the hands rise into the view as **claws**: `dx ±0.16, dy +0.08, dz +0.22, rx −20, ry ∓25, rz ∓15`, then
clench (`ry` pulls ∓10 more, `dz +0.02`), trembling 0.016 (strong), the whole view **tilts** 2° forward (camera pitch
+2: the hunch); holding: the hands follow the puppet's height (the M hand `dy` from +0.00 to +0.30 as the victim rises;
the O hand `dy −0.10`), the overlay 2.2 (red vignette, heartbeat). Fling: both hands whip `dx ±0.20, dz +0.30` in 3 ticks.
Sever: the M hand rises to the centre-top of the view `dx −0.02, dy +0.26, dz +0.30, rx −30` (the thumb on the forehead,
the victim's face fills the view since they are 1 block away), the O hand at the centre `dy +0.02, dz +0.30`; the
tremble stops for 10 ticks; a red pulse; then the hands fall. Release: hands open and fall `dy −0.20`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Wind-up | 2 **dark-red motes** (`#7A0E1E`, scale 0.5, α 0.6)/tick rising from the claws; the moon's light: a faint silver glow on the hands (`FxShapes.glow` r 0.2 `#E8F0FF` α 0.2) | 2/tick |
| Seize | the victim is outlined (`EntityGlow` dark red `#7A0E1E`); **4 red threads** (`ENERGY_BEAM` `#9A1E2E` α 0.45, width 0.03, `flowingPath` jitter 0.08) run from the claws to the victim's wrists and ankles (`HandTracker`-like points on mobs: the model's limb pivots if humanoid, else box corners); 3 red motes/tick at the victim | 3/tick + 4 threads |
| Hold | the threads **pulse** with the heartbeat (α 0.45 ↔ 0.7 at 0.9 Hz); the victim's pose 12.4 (the puppet, with a 2-tick lag behind the claws); 1 mote/tick | 1/tick |
| Fling | the threads **snap** (fade in 3 ticks), 12 red motes burst at the victim, a motion streak (`ENERGY_BEAM` red α 0.3) | 12 |
| Severance | the threads **all gather at the forehead** (converging over 8 ticks); at the press, a **dark pulse**: an `ENERGY_VEIL` sphere r 0 → 2 over 6 ticks, `#2A0A10` α 0.5 → 0, and the victim's **element colour** (their aura) **drains**: 16 motes of their element's colour flow from them into the blood bender's hand over 10 ticks, then **grey out** (fade to `#777777`); a silver flash from the moon; the victim's HUD greys (`UI.md`) | 20 |
| Release | the threads fade in 6 ticks, 6 red motes | 6 |
| Refused (stronger waterbender) | the threads form and **break at once** (2 ticks), the defender's eyes flash blue (`GlowingEyesLayer` `#8FE3FF` for 10 ticks) and 8 blue motes burst at them; the caster staggers (`HIT_STAGGER`) | 8 |
| Cap | 2 victims (M10), 8 threads, 30 motes | |

**Sounds:** `blood_grab` (a low, wet, straining groan with a heartbeat entering, 0.8 s), `blood_loop` (a slow heartbeat
with a tense string drone and a faint liquid creak, volume 0.6; the victim hears a louder, faster heartbeat:
`blood_victim_loop`), `blood_move` (a creaking strain with each large puppet move), `blood_fling` (a wet snap and a whoosh),
`blood_release` (a breath out, the heartbeat stopping), `blood_sever` (a long inhale, silence, then a deep soft thud and
a sigh of released air, 1.5 s; the victim hears a high ringing fade `severed_victim`), `blood_refused` (a glassy chime
and a snapping string: the defender's strength).

### 5.8 Water hit reactions

Soaked targets drip (12.3); frozen feet give the stuck pose; a creature hit by a spout tumbles; the bloodbent puppet is
12.4; a severed bender's **loss** is 12.5.

---

## 6. Fire

### 6.0 Fire idle: the ready stance

| Stance | tauIn / tauOut | Rows |
|---|---|---|
| `FIRE_IDLE` (ambient) | 3 / 5 | `FIRE_BOW`-lite: fists chambered at the hips (−30,0,±34), the front leg forward (x −14/+12, z ±8), lean 6, drop 0.08; the **breath**: head −2 ↔ +1 and lean 6 ↔ 4 at 0.1 rad/tick (a visible inhale every 3 s); small fist pulses (x ±2 at 0.3 rad/tick: tension). |
| `FIRE_PUNCHING` (ambient while Fire Punch is on) | 3 / 5 | fists raised to a guard (−70,∓20,∓10 → the hands before the chin), elbows in; the breath; the punch swing is **kept** (the vanilla swing plays through the ADD rows). |
| `FIRE_BREATHING` (aimed, while Fire Breath) | 2 / 4 | 6.3. |
| `FIRE_WHIPPING` (aimed) | 2 / 4 | 6.4. |
| `FIRE_JETTING` | 1 / 3 | 6.5. |
| `LIGHTNING_CHARGING` (aimed) | 1 / 4 | 6.7. |
| `LIGHTNING_REDIRECTING` (aimed) | 1 / 3 | 6.8. |

**Ambient particles of a master (Fire ES ≥ 60):** 1 `SMALL_FLAME` (blue: `SOUL_FIRE_FLAME` for Azula) every 8 ticks from
a fist, rising 0.3 and dying; at ES 100 a heat shimmer sheet over the shoulders (`MATTER_TEXTURED(heat.png)` α 0.15, 1 ×
0.6 blocks) while standing still; a faint crackle sound every 4 s at volume 0.1.

**Fire first-person idle:** fists chambered low at the sides (`dx ±0.22, dy −0.18, dz +0.04, rx +10, ry ∓10`), rising
0.01 on the inhale; a flame flickers on the knuckles (1 `SMALL_FLAME` per 10 ticks at each hand).

### 6.1 Fireball — כדור אש

**`FIRE_FIREBALL`** (dur 12 / 1 / 4, CAST, force): Northern Shaolin straight punch:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `FIRE_REST` | `FIRE_REST` | | | | |
| 2 | O | **−20,0,30** | −70,−20,−10 | −30 / 25 (±8) | −3 | lean 4, drop 0.10 | the **chamber**: the M fist pulls back to the hip, the O fist guards; the bow stance drops in; the **inhale** (head up) |
| 4 | **O** | **`FIRE_PUNCH`-M (−90,0,0)** | −10,0,−30 | −32 / 26 (±8) | 2 | **lean 12**, twist 14 | the **punch**: the fist snaps level forward in 2 ticks, the rear fist pulls back, the hips turn; the fireball leaves at tick 4 |
| 7 | S | −88,0,2 | −10,0,−30 | | 2 | lean 12, twist 12 | held out (the fist smokes) |
| 12 | S | `FIRE_REST` | `FIRE_REST` | −8 / 8 | 0 | lean 2, twist 0 | |

M5 (two): the O fist punches at tick 6 (−90,0,0 mirror, twist −14). M8 (three): the M fist again at 8 (dur 14). **Great
fireball (M10, double-tap), `FIRE_GREAT_FIREBALL`** (dur 20 / 2 / 6): t 4 both fists chamber at the hips (−20,0,±30), a deep
bow stance (drop 0.16), a 4-tick **inhale** (head −8, lean −4); t 10 both arms **circle out and back in** (−100,∓50,±40 →
−80,∓30,∓10 at 10: the hands gather the fire before the chest); t 13 (`O`) **both palms thrust** (`FIRE_DOUBLE`, lean 14);
recovery. **Small Fireball (Sneak), `FIRE_FLICK`** (dur 8 / 1 / 3): a wrist flick: the M hand rises to −60,10,10 at 2 and
flicks to −80,−10,14 at 4 (`O`); rest.

**Lock:** none. **Blend:** the bow stance's legs are keyed 2–4 only, so walking shows otherwise; while flying (Creative,
Fire Jet): the legs are masked (14.3); under water: refused. **Started by:** the slot's cooldown edge (1.2 s; the great
one 4 s; the flick 0.5 s). **Prediction:** always (not in water).

**First person:** t 2 `M dx +0.10, dy −0.16, dz −0.10, rx +10` (the fist pulls back past the edge), O up as a guard `dx
−0.12, dy +0.10, dz +0.14, ry +20`; t 4 **punch**: `M dx +0.02, dy +0.06, dz +0.40, rx −5` (the fist shoots to the centre;
the fireball bursts from it); t 7 hold; recovery. Camera: a 2-tick FOV kick ×1.02 at the punch (not with reduced motion).
Great fireball: both fists chamber `dy −0.16`, circle out `dx ±0.30, dy +0.14`, gather `dx ±0.06, dy +0.04, dz +0.14`,
thrust `dz +0.40` both. Flick: `M dy +0.06, dz +0.22` quick.

**Effects:**

| Where | What | Count |
|---|---|---|
| Chamber (t 2–4) | the fist **ignites**: 4 `FLAME` (Azula: `SOUL_FIRE_FLAME`; every "FLAME" below follows the look's palette) + a `FxShapes.glow` orb r 0.12 `#FFD23F` (blue: `#CFF2FF`) at the fist | 4 |
| Flight | the `FireballEntity`: a **core sphere** (`FxShapes.glow` r = size, core `#FFF4C2`, rim `#FF6A1F`; blue: core `#E8FBFF` rim `#3FA4FF`) + a **flame tail**: 5 trailing glow discs shrinking (α 0.6 → 0) + 3 `FLAME` + 1 `SMOKE`/tick; the ball **tumbles** (its texture-less: no need) | 4/tick |
| Burst | an **expanding flash sphere** (`ENERGY_VEIL`... `ENERGY_BEAM` sphere r 0 → burst radius over 4 ticks, α 0.6 → 0) + 12 `FLAME` + 6 `LAVA`-like embers (`EnergyParticle` `#FF8A1F` scale 0.4 with gravity: use `bending:grain` tinted orange with the ember flag) + 4 `SMOKE` + a `MATTER_FLAT` **scorch disc** on the ground (α 0.4, 100 ticks) | 22 |
| Great fireball | r 1.2, tail 8 discs, 6 `FLAME`/tick, burst r 4: 30 `FLAME` + 12 embers + 8 `SMOKE`, shake 0.5 | 8/tick |
| Small fireball | r 0.1, 1 `SMALL_FLAME`/tick, no burst (a 0.3 flash) | 1/tick |
| Vs water | the ball **snuffs**: 16 `CLOUD` steam + a hiss, no burst | 16 |
| Cap | 6 fireballs, 60 flames | |

**Sounds:** `fire_ignite` (t 2: a sharp fire catch with a breath, 0.2 s), `fireball_throw` (a punchy whoosh with a
crackling body), `fireball_fly_loop` (a soft roaring flutter, volume 0.3, follows the ball), `fireball_burst` (a fiery
"boom-crackle" 0.6 s; the great one adds a sub-bass drop), `fire_flick` (a tiny puff), `fire_vs_water` (a hiss and a
steam sigh).

### 6.2 Fire Punch — מכת אש

**On, `FIRE_PUNCH_ON`** (dur 10 / 1 / 4, CAST, not force): the fists **clench and ignite**: t 3 both fists rise to the
guard (−70,∓20,∓10, `O`) with a sharp **inhale** (head −6 for 2 ticks); t 5 the fists **flare** (a 2-tick tremble 2 at 5.0: the
ignition); t 10 `FIRE_PUNCHING` hand-over. **Off:** the fists open and shake out (tremble 3 for 3 ticks) and drop; the flames
die. **Each hit:** the vanilla swing with an ADD row: the punching arm gets +10° x snap (via `FIRE_PUNCH_HIT`, group AUTO,
dur 6: at tick 1 the arm +8 x (ADD), lean +4, twist +8 toward the punch; the third hit of a combo (M4) adds a step (legs
−20/16) and lean +10). **Fire Kick (Sneak), `FIRE_KICK`** (dur 14 / 1 / 5, force; **NEW** leg z keys): t 3 the body
**coils**: twist −20, lean 4, the M arm back (20,0,30), O forward (−60,−10,−20), weight on the left leg (legs 10/−6); t 6
(`O`) **the kick**: right leg to **`FIRE_KICK`** (x −105, z +22), the body leans back (lean −10), twist +25, arms swing
(M −60,0,40 / O 40,0,−20) — the crescent of flame leaves the foot at tick 6; t 9 the leg sweeps across (z +22 → −10 with
x −90, `E`); t 14 `FIRE_REST`, twist 0. Mid-air: the same with a flip: `yaw` 0 (no), but `lean` runs −10 → −60 → 0 (a
back-flip feel via lean only, since full flips need pitch > 90: **lean is clamped to ±112 in the profile; a flip is a
later idea**).

**Lock:** none. **Blend:** the kick's legs are keyed 3–9 (walking pauses on the legs for 6 ticks); flying: the kick plays
without the stance legs. Under water: ×0.5 damage, the same gesture at 0.8 rate. **Started by:** `Toggle.FIRE_PUNCH`;
payload kinds `FIRE_PUNCH_HIT` (with the combo count), cooldown edge (kick 3 s). **Prediction:** toggle always; kick
always.

**First person:** on: fists rise to the guard `dx ±0.14, dy +0.08, dz +0.16, ry ∓15` and ignite (the knuckles burn: 2
`FLAME` per hand per tick at the fist, drawn in front of the camera); the fists glow (a `glow` orb at each). Hit: the
vanilla swing plus `dz +0.08` snap. Kick: the camera **leans back** 6° (pitch −6, roll +4 with the twist) for 4 ticks
(reduced motion: none), a flame crescent sweeps across the lower screen (the kick's leg is not drawn in first person: a
**flame arc effect** at the bottom of the view stands for it: an `ENERGY_BEAM` arc ribbon sweeping left to right over 3
ticks); hands swing `dx +0.20 / −0.20`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Fists on | 2 `FLAME`/tick per fist (blue for Azula) + a glow orb r 0.12 + 1 `SMOKE`/4 ticks; the flames **flare** (4 extra) on each punch | 4/tick |
| Hit | a **flash** at the target (`ENERGY_BEAM` sphere r 0.6 α 0.6 → 0 over 3 ticks) + 8 `FLAME` + 4 embers; the target burns (vanilla fire render) | 12 |
| Combo (M4) | the third hit adds a `FxShapes.cylinder` jet 1 block behind the fist (α 0.4, 3 ticks) and 6 extra `FLAME` + knockback dust | 6 |
| Jab (M7) | a small fireball (6.1 small) from the fist | |
| Kick | a **crescent of flame** (`ENERGY_BEAM` annulus segment of 120°, r 1.2 → 2.5 over 4 ticks, `#FF6A1F` → `#FFD23F` edge, α 0.7 → 0) sweeping with the foot + 12 `FLAME` along it + 4 embers; targets flash | 16 |
| Cap | 1 pair of fists, 40 flames | |

**Sounds:** `fire_fists_on` (an inhale and two fire catches), `fire_fists_loop` (a low crackling flicker, volume 0.3),
`fire_punch_hit` (a punch thud with a fiery "fwoosh"), `fire_combo` (a bigger whoomp), `fire_fists_off` (a flame
dying out), `fire_kick` (a sweeping fire whoosh with a sharp crack), `fire_kick_hit` (a flaming thud).

### 6.3 Flamethrower — להביור

**`FIRE_FLAMETHROWER_START`** (dur 8 / 1 / 3, CAST, force) then the stance `FIRE_FLAMETHROWING` (aimed, tauIn 2 / out 4)
while `Live.FIRE_STREAMING`: start: t 2 both fists chamber at the hips (`FIRE_BOW`), a 2-tick inhale (head −6); t 5
(`O`) both arms **shoot forward** to `FIRE_DOUBLE` (−90,∓6,±4), palms open (y 0), lean 12, the bow stance (−30/25); the
stance: both arms aimed along the look (`aim(LOOK, 0, ∓6, 0)`) with **tremble 1.0 at 3.5** (the recoil hum), the body
braced (lean 12, drop 0.12, legs −30/25 z ±8), head level; `occupy` both; every 20 ticks a small **recoil** (both arms +4
x for 2 ticks, lean +2). End: the arms **drop** (`FIRE_FLAMETHROWER_END` dur 8: `E` to `FIRE_REST`) and the jets die.
**Fire Breath (Sneak, M7+), `FIRE_BREATH_START`** (dur 10 / 2 / 4) then the stance `FIRE_BREATHING` (aimed) while
`Live.FIRE_BREATHING`: start: both fists chamber and pull back and **down** (10,0,±40 at 4), the chest **opens**: head
**−14** (back), lean **−8**, a 6-tick inhale (the ribs: the body scale is not animatable; the head tilt and lean back
read as the breath); at 8 the head **snaps forward** (head +6) and the breath begins; the stance: `FIRE_BREATH` (arms
back and wide −40,0,±50, head aimed: the head drives the jet: `aim` on the head via the look (the shared head rule
already looks along the camera) with head +6 additive, lean −2 → +6 pulsing at 0.3 rad/tick (the exhale push), drop 0.04,
legs −20/20 z ±10; **locks movement** (`Live.FIRE_BREATHING` input lock: you cannot walk while breathing). End: the head
drops (head +12 for 3 ticks: a cough) and the arms fall. Before M7, Sneak plays `FIRE_FLAMETHROWER_START` one-handed
(the M arm only).

**Lock:** flamethrower none (speed ×0.6 attribute); breath locks. **Blend:** the flamethrower stance rides over slow
walking (the legs keep walking at 0.6 speed; the bow stance legs are **masked** by the walk amount: when `limbAmount >
0.2` the stance's leg rows weight → 0: 14.3); flying (Fire Jet): the arms still aim. Under water: refused. **Started by:**
`Live.FIRE_STREAMING`, `Live.FIRE_BREATHING`. **Prediction:** on the press when not in water (the client sets a local
"predicted stream" that ends when the server's bit arrives or 12 ticks pass).

**First person:** start: fists chamber `dx ±0.20, dy −0.16` then both **shoot forward** `dx ±0.10, dy +0.04, dz +0.38, rx
−6, ry ∓6` (both palms at the lower centre, the jets leaving them: the jets' origins are `FirstPersonArms.hand`);
trembling 0.010 while firing; the recoil `dz −0.03` every 20 ticks; the heat ripple overlay (2.2) on the lower half; FOV
×0.98 (a slight tunnel). Breath: the hands pull back and down to the bottom corners `dx ±0.30, dy −0.26, dz −0.04`, the
camera **tilts up** 8° over 6 ticks (the inhale: pitch −8) then **snaps down** 10° (pitch +2 relative) as the breath
starts; the jet comes from the **camera** (origin = eye − 0.2 y, forward): the lower-middle of the view is fire; the heat
ripple covers the whole screen at α 0.1; the camera shakes 0.15 continuously (reduced motion: none).

**Effects:**

| Where | What | Count |
|---|---|---|
| The jets | Enhanced's `ContinuousBeam`/`BeamRenderer` with a **flame style** (`BeamStyle FLAME`: core `#FFF8D0`, inner `#FFE070`, main `#FFA030`, body `#E85A10`, fringe `#8C2A08`; blue: `#F4FDFF`, `#CFF2FF`, `#6FC4FF`, `#3FA4FF`, `#1B4FCC`), **two beams** (one per hand: `BeamPayload` per hand, `offHand` true for the second), drawn **wider and softer** than an energy beam: the 5 layers at ×2.2 width (fringe 0.66, body 0.37, main 0.2, inner 0.11, core 0.05), taper **inverted** (narrow at the hand 0.25 → full width at 2 blocks → 1.4× at the end: a cone), `flowingPath` jitter 0.08 (fire wavers), the strands replaced by **flame tongues**: 4 `crossfade`d short ribbons branching off the last third, upward-biased (gravity −: fire rises); + 6 `FLAME` + 2 `SMOKE` per tick spawned along each jet + 1 `LAVA`-like ember/tick; at the end point: a `glow` splash r 0.6 and a scorch disc | 18/tick for both jets |
| Hit | the target burns; 6 `FLAME` + 3 embers/hit | 9 per hit |
| Melting ice | `DRIPPING_WATER` ×4 + `CLOUD` ×4 per block | 8 |
| Fire Breath | one beam from the mouth (`HandTracker.forehead` − 0.25 y), style FLAME at ×3 width, cone taper to 2.5 wide, 10 `FLAME` + 4 `SMOKE`/tick + heat shimmer sheet along it; the bender's head glows (`GlowingEyesLayer`-style mouth mask `fire_mouth.png`, `#FFD23F`) | 14/tick |
| Cap | 1 stream (2 beams) per player, 60 flames | |

**Sounds:** `flamethrower_start` (an inhale and a double ignition "fwoosh"), `flamethrower_loop` (a roaring jet with
a crackling layer, volume 0.7, pitch 1.0; blue fire: pitch 1.15 with a hissing top), `flamethrower_end` (the roar
cutting out with a puff), `fire_breath_inhale` (a long dragon's inhale 0.6 s), `fire_breath_loop` (a deeper, wider roar
with a vocal rasp, volume 0.8), `fire_breath_end` (a cough-puff and embers), `fire_melt` (hiss + drips).

### 6.4 Fire Whip — שוט אש

**`FIRE_WHIP_START`** (dur 8 / 1 / 3, CAST, force) then the stance `FIRE_WHIPPING` (aimed, tauIn 2 / out 4) while
`Live.FIRE_WHIPPING`: start: t 2 the M fist chambers low and back (20,0,30, `O`); t 5 the arm **sweeps up and out** in an arc
to −60,−30,50 (`E`) as the whip **ignites along the arm** and uncoils from the fist (effects); the stance: the M arm held
up and back (`FIRE_WHIP`-like: −70,−20,40), **strikes every 12 ticks**: a 3-tick lash (`O`) to aim(LOOK, −10, 10, 0) with
lean 4 → 12, twist 0 → 16, the wrist leading (y −20 → 20), the O fist guarding at the chest (−70,−20,−10); then a 7-tick
return (`E`); legs in the bow stance (masked while walking); `occupy(M)`. M4 (two whips, double-tap): the O arm mirrors
with lashes offset by 6 ticks. End: the arm **snaps the whip out** (a flick to −90,−30,30 for 2 ticks, `B`) and it dies
(`FIRE_WHIP_END` dur 8). **Whip Lash (Sneak), `FIRE_WHIP_LASH`** (dur 14 / 1 / 5, force): the M arm **winds back far**
(40,20,40 at 3, twist −20, lean −4); t 6 (`I`) the **crack**: aim(LOOK, −15, 20, 0) with lean 14, twist 24, a long step
(legs −34/28); hold 2; t 14 rest.

**Lock:** none (speed ×0.8). **Blend:** walking at 0.8; the bow legs masked while walking; flying: fine; under water:
refused. **Started by:** `Live.FIRE_WHIPPING`; cooldown edge (lash 3 s). **Prediction:** not in water.

**First person:** start: the M fist chambers `dx +0.18, dy −0.18, dz −0.06` then sweeps up and out to the right edge `dx
+0.34, dy +0.22, dz +0.04, ry +35, rz −20` (the whip uncoils from the hand down out of view and back in as it hangs);
lashes: the hand snaps to the centre `dx +0.06, dy +0.10, dz +0.34, ry −20` over 3 ticks and returns over 7; the whip's
coil is drawn from the hand (effects). Lash: wind back `dx +0.38, dy +0.10, dz −0.10` then crack `dx 0, dz +0.40`. The
O fist guards `dx −0.14, dy +0.08, dz +0.14`.

**Effects:**

| Where | What | Count |
|---|---|---|
| The whip | a **rope of fire**: a chain of 12 segments from the hand (verlet-like: gravity **upward** 0.3 (fire rises) and drag, so it hangs up and back like a flame, not down like water), drawn as a `BeamRenderer`-style 3-layer ribbon along the chain (widths 0.22 / 0.12 / 0.04, FLAME palette, additive) with 2 `FLAME`/tick along it and 1 `SMOKE`/2 ticks from the tip; the tip has a small glow orb | 2/tick |
| Lash | the tip is **driven to the target** over 3 ticks (the chain's end point set, the rest follows) with a **crack flash** (`ENERGY_BEAM` sphere r 0.5 α 0.7 → 0 in 3 ticks) + 8 `FLAME` + 3 embers at the hit; the ground/block hit gets a **fire block** (if allowed) or a scorch disc | 11 |
| Cutting a rock (M7) | 6 grains + 8 `FLAME` at the cut, the rock halves | 14 |
| Wrap (M10) | the chain's last 4 segments **wrap** the target's box for 6 ticks while it is pulled | |
| Lash (Sneak) | the chain stretches to length +3 straight (a 2-tick snap), a bigger flash r 0.9, 12 `FLAME`, knockback dust | 16 |
| End | the chain **burns out** tip-first over 4 ticks (α → 0) with 6 `SMOKE` | 6 |
| Cap | 2 whips, 40 flames | |

**Sounds:** `fire_whip_ignite` (a rising fiery hiss as the whip uncoils, 0.4 s), `fire_whip_loop` (a crackling,
fluttering flame with a soft wind, volume 0.4), `fire_whip_strike` (a whip-crack with a fiery whoosh), `fire_whip_hit`
(a crack-thud and a flare), `fire_whip_lash` (a big crack with a long whoosh), `fire_whip_end` (a flame snapping out).

### 6.5 Fire Circle — מעגל אש

**`FIRE_CIRCLE`** (dur 16 / 2 / 5, CAST, force):

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | S | `FIRE_REST` | `FIRE_REST` | | | | |
| 3 | O | −60,40,−30 | −60,−40,30 | 0/0 (±10) | −4 | lean 2, drop 0.10, **twist −30** | both arms **wind across the body**, fists lit, the shoulders coil hard |
| 4 | L | −80,0,60 | −80,0,−60 | | 0 | **yaw 90** | the spin begins: arms fly out level |
| 12 | L | −80,0,60 | −80,0,−60 | | 0 | **yaw 360**, lean 0, drop 0.12, twist 0 | a **full spin** over 8 ticks with the arms out: the ring leaves the fists at 4 and runs out (server: the ring expands from tick 4 over 10 ticks) |
| 16 | S | `FIRE_BOW` | `FIRE_BOW` | −30/25 (±8) | 0 | yaw 360, lean 10 | lands in the bow stance |

M6+ (the leg sweep): `FIRE_CIRCLE_LEGS` (dur 20 / 2 / 6): the same to tick 12, then at 13 the body **drops** (drop 0.42,
the left leg folds x −60 z −20: a crouch) and the **right leg sweeps** (`FIRE_KICK` legs: x −30 z +60 → the leg level,
sweeping: **yaw 360 again** from 13 to 19 with the leg out, arms braced on the ground side (M −40,0,60 / O 20,0,−30),
lean 10) — the second, lower ring leaves at 14; t 20 up into `FIRE_BOW`. The whole gesture turns 720° (two yaw turns:
allowed, "a whole number of turns"). **Fire Jet (Sneak, hold), `FIRE_JET_START`** (dur 6 / 1 / 2) then the stance
`FIRE_JETTING` (tauIn 1 / out 3) while `Live.FIRE_JETTING`: start: fists **point down** (30,0,±24 at 3, `O`), knees bend
(drop 0.14); the stance: fists and feet jetting: arms **down and back** (30,∓10,±30) with tremble 1.5 at 4.0, legs **straight
and slightly back** (x 12/12 z ±8), head level, **lean follows the steering**: `lean = 20·fwd` (leaning into the direction of
travel), roll `10·side`; a bob 0.02 at period 6 (the thrust). End: `LAUNCHED` → `LAND_SOFT` with a fiery dip (6 `FLAME`).

**Lock:** none. **Blend:** the spin is visual yaw; the walk keeps running under it (the legs are not keyed 4–12), so you
can run and spin; flying: the circle plays (the ring at your height); under water: refused. Jet: the body is in the air
(`wAir` 1 with the `FIRE_JET` flight profile: 10.7). **Started by:** cooldown edges (8 s; M10 double at 10 ticks apart),
`Live.FIRE_JETTING`. **Prediction:** not in water; jet always.

**First person:** t 3 both hands cross `dx ∓0.16, dy +0.06, dz +0.12, ry ∓40` (fists lit); t 4–12 both hands fly to the
edges `dx ±0.38, dy +0.08, dz +0.06` and the **camera yaws 360°** over 8 ticks (reduced motion: no yaw; the ring is still
seen expanding); fire sweeps across the view (the ring passes through the camera's position: the ring's ribbon is drawn
around you, so you see it leave); t 16 the bow stance hands. Leg sweep: the camera **drops** 0.4 (eye offset) for 6 ticks
and yaws again; the lower screen is flame. Jet: hands point down at the bottom corners `dx ±0.28, dy −0.30, dz +0.06, rx
+40`, trembling 0.012, jets of flame from them (the first-person jet effects: two `FLAME` streams downward from the hands,
6/tick each, drawn in view space); FOV ×1.1; the camera pitches +4 into travel; a soft shake 0.1.

**Effects:**

| Where | What | Count |
|---|---|---|
| Wind-up | the fists ignite (4 `FLAME` each) + a crossing of two flame ribbons before the chest (the hands' trails) | 8 |
| The ring | an **expanding ring of fire**: `EnergyGeometry.ring` r 0 → radius over 10 ticks at waist height (normal Y), width 0.45, FLAME palette (3 rings: fringe/main/core widths 0.45/0.25/0.08), additive; **flame tongues**: 24 short upward ribbons (`crossfade`d, 0.3–0.6 high) riding the ring; 16 `FLAME`/tick along the ring (capped) + 6 `SMOKE`/tick; the ground under the ring gets a **scorch ring** (`MATTER_FLAT` annulus α 0.35, 100 ticks); fire blocks where allowed; targets crossed flash | 22/tick for 10 ticks |
| Leg ring (M6) | a second ring 0.3 above the ground, 4 ticks later, FLAME, width 0.3, with 12 `FLAME`/tick | 12/tick |
| Snuffed by water | a gap opens in the ring (the ring's arc is drawn with a missing segment where water cut it) + 10 `CLOUD` | 10 |
| Fire Jet | **jets from the hands and feet**: 4 `BeamRenderer` short beams (0.8 long, FLAME, width ×1.5, pointing down and back from `HandTracker.hand` and `.foot`) + 6 `FLAME` + 2 `SMOKE`/tick per jet (capped 20/tick total) + a **heat trail**: `FlightEffects` ribbons from the feet (FLAME body colour α 0.3, 1.5 blocks); landing: a 1.2-block `MATTER_FLAT` scorch + 8 `FLAME` | 20/tick |
| Cap | 2 rings, 1 jet set, 80 flames | |

**Sounds:** `fire_circle_wind` (two fire catches and a sharp inhale), `fire_circle` (a spinning whoosh that erupts into a
wide ring roar, 1 s, with a crackle tail), `fire_circle_legs` (the same with a lower second roar), `fire_circle_snuff`
(a steam hiss), `fire_jet_start` (two jet ignitions), `fire_jet_loop` (a roaring thrust with a flutter, volume 0.6,
pitch with speed), `fire_jet_end` (the jets cutting, embers falling), `fire_land` (a fiery thud).

### 6.6 Fire Healing — ריפוי באש

**`FIRE_HEAL_START`** (dur 10 / 2 / 4, CAST, not force) then the stance `FIRE_HEALING` (aimed, tauIn 2 / out 4) while
`Live.FIRE_HEALING`: start: both palms **rise to the chest** (−70,∓30,∓14, `E`), a 4-tick **inhale** (head −6, lean −4);
at 8 the palms **turn out** toward the target (aim(LOOK, −10, ∓12, 0)), the exhale (head +2, lean +2); the stance: the
palms held out and slightly apart, a slow **breathing** cycle (x ±4, lean ±3 at 0.1 rad/tick: the healer breathes with
the patient), fingers spread (y ±12 pulsing); self-healing: the palms press to the own chest (−80,∓40,∓20). `occupy` both.
End: the palms lower (`FIRE_HEAL_END` dur 8). **Burn Out (Sneak), `FIRE_BURN_OUT`** (dur 12 / 1 / 4, force): both palms
rise (−70,∓30,∓14) and at 5 **clap toward the target** (aim(LOOK, 0, ∓4, 0), `O`, lean 6): a sharp flash; t 12 rest.

**Lock:** none (both must stand still: server). **Started by:** `Live.FIRE_HEALING`; cooldown edge (Burn Out 15 s).
**Prediction:** a target below full health in sight (or self).

**First person:** the palms rise to the centre `dx ±0.10, dy +0.06, dz +0.20, rx −10, ry ∓20`, warm light between them,
breathing ±0.01; self: palms toward the camera `rx +50, dz +0.08`. Burn Out: a clap `dx ∓0.02, dz +0.30`.

**Effects:**

| Where | What | Count |
|---|---|---|
| Hands | a **warm glow** between the palms: `FxShapes.glow` r 0.25 `#FFD98A` (core `#FFF4C2`) α 0.5, pulsing with the breath; 2 **ember-glow motes** (`EnergyParticle` `#FFC46A` scale 0.4, slow, rising)/tick | 2/tick |
| The healed | a **golden fire aura**: `energySwirl` over the model `#FFD98A` α 0.25 scrolling slowly up (the dragons' fire: warm, not burning); 3 glow motes/tick rising; each heart: a pulse | 3/tick |
| Stream | an `ENERGY_BEAM` soft ribbon (fringe only, width 0.3, `#FFC46A` α 0.25) from the palms to the target, gently wavering | 1 |
| Burn Out | a **flash** at the target (`ENERGY_BEAM` sphere r 1 α 0.7 → 0 in 4 ticks, gold) + 10 glow motes + the poison particles **burn away** (8 `SMOKE` tinted green-grey) | 18 |
| Cap | 1 target, 30 motes | |

**Sounds:** `fire_heal_start` (a soft inhale and a warm crackle rising into a hum), `fire_heal_loop` (a low warm pad
with a soft crackle and a breathing cycle, volume 0.4; a soft chime per heart `heal_tick` at pitch 0.9), `fire_heal_end`
(the hum fading), `fire_burn_out` (a bright flash-sizzle and a sigh).

### 6.7 Lightning Generation — יצירת ברק

**Charge, `LIGHTNING_CHARGE`** (dur 24 / 3 / 1, CAST, force; **the gesture is held** at its last key while the key is down
(`HeroAnimator.hold`), **locks movement** via `Live.LIGHTNING_CHARGING`): Iroh's form:

| t | ease | M | O | legs (z) | head | body | notes |
|---|---|---|---|---|---|---|---|
| 0 | E | `FIRE_BOW` | `FIRE_BOW` | −24/22 (±8) | 0 | lean 4, drop 0.10 | the bow stance, a **breath in** (head −4 over 4 ticks) |
| 6 | E | **−120,−30,50** | **−40,−40,−30** | | −2 | lean 2 | the M arm **circles up and out**, the O arm circles **down and in**: the two-finger hands trace opposite circles (y and z sweep) |
| 12 | E | −60,50,−40 | −120,30,30 | | 0 | lean 4, twist −10 | the circles continue (the arms have swapped high/low): sparks follow the fingertips |
| 18 | E | **`LIGHTNING_MEET`** | **`LIGHTNING_MEET`** | | 4 | lean 4, drop 0.12, twist 0 | the fingers **meet at the stomach** (both hands at the belly, fingertips touching): the charge gathers |
| 24 | **O** | **`LIGHTNING_POINT`-M (−92,0,2)** | −20,10,−16 | −30/25 (±8) | 0 | **lean 6**, twist 12 | the M arm **extends**, two fingers out, the O hand braced at the hip: **held here** until release |

On **release** (`LIGHTNING_RELEASE`, dur 10 / 1 / 4, force): a sharp **recoil**: the pointing arm kicks back 10° (x −82)
and the body leans back 4 (lean 2) for 2 ticks (`O`), the head snaps −3; then the arm lowers to `FIRE_BOW` over 8 ticks. The
bolt fires at release tick 0. **Dual (M10), `LIGHTNING_CHARGE_DUAL`**: the same to 18, then **both** arms extend
(`FIRE_DOUBLE` with two fingers: −92,∓4,±2) at 24, both braced; release recoils both. **Lightning Arc (Sneak),
`LIGHTNING_ARC`** (dur 10 / 1 / 4): a quick jab: the M hand to `LIGHTNING_MEET` at 3 (`E`) and **jabs** out (−90,−10,10,
`O`) at 5 with twist 10; rest.

**Lock:** charging locks movement; released, free. **Blend:** standing only (locked); if the player is airborne when the
key is pressed the charge still plays (a falling bender charges: legs trail). Under water: works (you take 1 heart).
**Started by:** `Live.LIGHTNING_CHARGING` (rise → CHARGE held; fall → RELEASE if the cooldown started in the same tick,
else a cancel: `LIGHTNING_CANCEL` dur 8: the arms drop and the sparks die); the Arc by its cooldown edge (5 s).
**Prediction:** on the press when ready: the charge is local at once; the release is predicted on key-up after 10 ticks.

**First person:** t 0 fists chamber `dx ±0.20, dy −0.16`; t 6–18 the hands **circle**: the M hand traces a circle of r
0.22 clockwise (centre `dx +0.12, dy +0.04, dz +0.24`), the O hand counter-clockwise (centre `dx −0.12`), two fingers out
(the hand model is the vanilla hand: the "two fingers" are read from the **spark effect** at the fingertips and the
hand's `ry` turn toward the camera); t 18 the hands **meet at the bottom centre** `dx ±0.03, dy −0.16, dz +0.22, rx +30, ry
∓30` with blue-white sparks crawling between them; t 24 the M hand **extends to the centre** `dx +0.04, dy +0.06, dz +0.44,
rx −8` with an **arc of lightning** crackling between the two fingers and the O hand (at the hip `dx −0.22, dy −0.20`); the
**FOV pulls in** to ×0.96 over the 24 ticks; the hum rises; **release**: a white flash (overlay α 0.6 → 0 in 4 ticks), the
hand kicks back `dz +0.36`, the camera jolts (shake 0.3, high frequency) and FOV snaps to 1.0; reduced motion: no
flash beyond α 0.2, no jolt.

**Effects:**

| Where | What | Count |
|---|---|---|
| Circles (t 6–18) | **sparks crawl on the arms**: 2 `ELECTRIC_SPARK` + 1 white-blue `EnergyParticle` (`#DDF4FF`, scale 0.4)/tick per hand at `HandTracker.hand`, plus a thin **arc** between the hands when they are within 1 block: `EnergyGeometry.bolt(a, b, 0.02, jitter 0.3, #DDF4FF, seed, phase)` crossfading every 1.5 ticks | 6/tick + 1 arc |
| Meet (t 18–24) | a **crackling ball** at the stomach: `FxShapes.glow` r 0.15 → 0.3 `#FFFFFF` core / `#9FD8FF` rim + 4 short random bolts off it per tick (`crossfade`), 4 `ELECTRIC_SPARK`/tick; the body's **eyes glow** white-blue (`GlowingEyesLayer` mask `#DDF4FF`) while charging | 8/tick |
| Held (24+) | the arc from the fingers to the O hand persists (0.4 long); 3 sparks/tick at the fingertips; a faint blue glow on the extended arm (a `ENERGY_BEAM` sleeve ribbon α 0.15) | 3/tick |
| The bolt | an **instantaneous bolt**: `EnergyGeometry.lightningPath(hand, hit, jitter 0.35)` drawn in **3 layers** (`ThorEffects.beam` recipe: halo `#4DB8FF` 0.30 wide, blue `#9FD8FF` 0.15, core `#FFFFFF` 0.055) for **8 ticks** fading (α 1 → 0 with 2 **reshapes** at ticks 2 and 4: a flicker), **2–4 branches** (`lightningPath` from random path points, reach 1–3 blocks, widths halved), a **flash sphere** at the origin r 0.6 and at the hit r 1.0 (`ENERGY_BEAM`, α 0.8 → 0 in 3 ticks), 20 `ELECTRIC_SPARK` at the hit + 8 along the path, a scorch disc; the sky **flashes** faintly for clients within 24 blocks (a white overlay α 0.15 for 2 ticks); dual (M10): two bolts from the two hands **joining** 1 block ahead into one 1.5× bolt | 28 + 1 bolt |
| The hit | the target is **lit white** for 4 ticks (`EntityGlow` white, or an overlay tint via `LivingEntityRenderer` colour: 12.5 paralysed), 12 sparks, the paralysis pose; in water: 4 radiating bolts along the surface (`lightningPath` 4 blocks) | 12 |
| Arc (Sneak) | a 6-block bolt at half widths, 4 ticks, 8 sparks | 8 |
| Cancel | the ball **pops**: 10 sparks fly off, the eyes dim | 10 |
| Cap | 1 charge, 2 bolts alive, 60 sparks | |

**Sounds:** `lightning_charge` (a rising, crackling electric whine from 0 to 24 ticks, 1.2 s: a sine sweep 180 → 2400 Hz
with crackle bursts every 3 ticks; the two-finger circles have soft "zip" sparks), `lightning_hold_loop` (a tense
crackling hum, volume 0.5, pitch slowly rising), `lightning_strike` (a deafening crack: a 2 ms click, a broadband burst,
a sub thump and a 0.8 s rumble tail; the dual bolt adds a second crack 40 ms later), `lightning_hit` (a sizzling impact
with a "zap" and a short ring), `lightning_arc` (a short crackling zap), `lightning_cancel` (sparks fizzling out),
`thunder_distant` (a rumble heard 24–64 blocks away, 0.6 s later).

### 6.8 Lightning Redirection — הטיית ברק

**Stance, `LIGHTNING_REDIRECT_START`** (dur 8 / 1 / 3, CAST, not force) then the stance `LIGHTNING_REDIRECTING` (aimed,
tauIn 1 / out 3) while `Live.REDIRECTING`: start: the bow stance; the M arm **rises level** toward the threat (aim(LOOK,
0, 0, 0): the receiving arm points at where the bolt will come from), two fingers out; the O arm **low and back** at the
hip (−30,0,−30): `REDIRECT_IN`; the body **sideways** (twist 30: the stomach turned away from the line, Iroh's lesson
that the bolt must not cross the heart), head looking at the threat, lean 2, drop 0.12; the stance holds with a calm
**breath** (lean ±1.5, no tremble: the redirector is calm). **The bolt arrives (success), `LIGHTNING_REDIRECT_PASS`** (dur
12 / 1 / 4, force; plays at the hit tick, group CAST replaces the stance's gesture): t 0 the bolt enters the receiving
(M) arm; t 3 (`E`) the M arm **sweeps down** to the stomach (−40,30,−20) and the body **bends** forward (lean 14, drop
0.18: the energy passes through the belly: the overlay shows it); t 5 the **O arm rises** from the hip (`E`) to level
(−92,0,−2) as the M arm settles at the hip: `REDIRECT_OUT`; t 6 the bolt **leaves** the O arm (the server fires the
redirected bolt at tick 6 of the pass, i.e. 6 ticks after the hit: the window); recoil (O −82 for 2 ticks); t 12
`FIRE_BOW`. If the player **looks elsewhere** before tick 6, the O arm aims there (aim(LOOK)). **Failure (hit),
`LIGHTNING_REDIRECT_FAIL`** (dur 14): the M arm receives (t 0), the body **convulses**: `PARALYSED` with a violent tremble
(4 at 6.0) and lean −12 at t 3 (`O`), a stagger back (drop 0.2, legs 10/−20), then `HIT_KNOCKDOWN` (12.1). **Ground It
(Sneak), `LIGHTNING_GROUND`** (dur 12): from the pass's tick 5, instead of rising, the O arm **drives down** to the ground
(60,0,−20 with lean 18 and drop 0.26, `I` at 7: a kneel); the bolt goes into the earth; t 12 up to `FIRE_BOW`, and 1 heart
heals (a soft gold pulse).

**Lock:** the stance slows (speed ×0.5 attribute), no lock; the pass locks for 12 ticks. **Blend:** the stance's legs
masked while walking; flying: allowed (airborne redirection, Zuko on the airship). **Started by:** `Live.REDIRECTING`;
payload kinds `REDIRECT_PASS`, `REDIRECT_FAIL`, `REDIRECT_GROUND` (sent at the hit). **Prediction:** the stance on the
press when ready.

**First person:** the stance: the M hand **level at the centre-right** `dx +0.14, dy +0.08, dz +0.40, rx −6` (two fingers
toward the threat), the O hand low left `dx −0.26, dy −0.22, dz +0.02`, the camera **yawed 20°** toward the threat side
relative to the body (the body's twist is third-person only; in first person the hands shift `dx` +0.08 to show the
sideways stance), breathing; **the pass**: a **white-blue bolt enters the right hand** (the bolt effect ends at the
hand), the hand **sweeps down to the bottom centre** `dx +0.02, dy −0.26, dz +0.18, rx +40` over 3 ticks while a **bright
line runs down the right edge of the screen, across the bottom (the stomach) and up the left edge** (the overlay: an
`ENERGY_BEAM`-style bolt drawn in screen space as a 3-layer polyline along that U path, white core, 6 ticks, with the red-hot
"path" afterglow `#FF6A4A` α 0.4 fading over 20 ticks), the camera **dips** 0.15 and pitches +6 (the bend), then the O hand
**rises to the centre** `dx −0.04, dy +0.06, dz +0.42` and the bolt **leaves it** (the bolt effect from that hand along the
look); a flash; recovery. Failure: a white flash α 0.8, the paralysis overlay (2.2), the camera jolts and falls back
(pitch −20 over 6 ticks, the knockdown). Ground It: the O hand drives to the bottom `dy −0.34, dz +0.14` and the bolt goes
down out of view, a gold pulse (`#FFD98A` α 0.3 fade) as the heart heals.

**Effects:**

| Where | What | Count |
|---|---|---|
| Stance | 1 spark/2 ticks at the receiving fingertips (anticipation); the eyes faintly lit (`#DDF4FF` α 0.5) | 0.5/tick |
| The pass | the incoming bolt's path **continues onto the body**: a 3-layer bolt ribbon (`ThorEffects.beam` recipe at half widths) drawn **along the arm to the stomach and up the other arm** using `HandTracker.hand(M)` → `HandTracker` belly (a new tracked point: the body's centre, **NEW** 14.8) → `HandTracker.hand(O)`, alive 6 ticks (the bolt is *inside* the bender for 6 ticks: visible on the body as the glowing path, additive, with 6 sparks/tick along it); then the **outgoing bolt** (6.7's bolt recipe) from the O hand; the body's **eyes blaze** white for 8 ticks; a `MATTER_FLAT` scorch under the feet (the earth took a little) | 6/tick + 2 bolts |
| Failure | the bolt hits (6.7 hit effects ×1.25), the body convulses (12.5), 20 sparks over 8 ticks, smoke 4 | 24 |
| Ground It | the body path ends at the O hand pointing **down**: a bolt into the ground (`lightningPath` 1.5 blocks down, branching along the ground surface 4 ways 3 blocks each, 4 ticks), a 2-block `MATTER_FLAT` scorch star, 16 sparks; a **gold pulse** on the bender (`energySwirl` `#FFD98A` α 0.4 for 10 ticks) | 16 |
| Cap | 1 body path, the bolts of 6.7 | |

**Sounds:** `redirect_stance` (a slow calm inhale with a faint electric tension, 0.6 s), `redirect_loop` (a very quiet
high hum, volume 0.15), `redirect_pass` (the crack of the hit at once, then a **rising electric whoosh through the body**
0.3 s: a filtered noise sweep up, a short pause, then the strike's crack again as it leaves: three events, 0.6 s total),
`redirect_fail` (the strike crack + a sizzling convulsion + a body fall), `redirect_ground` (the pass's whoosh then a deep
earth thud with a crackle spreading and a soft gold chime).

### 6.9 Fire hit reactions

Burning targets flail (12.3 burn); a fireball or whip knock staggers; Fire Kick knocks down; lightning gives the
**paralysis** pose (12.5); a Fire Circle crossing pushes.

---

## 7. Movement, rides and forms

### 7.1 Walking and running per element (`RunCharacter`)

| Element | walkArms | sprintArms | sprintLegs | lean | twist | bob | Character |
|---|---|---|---|---|---|---|---|
| Earth | 0.85 | 1.05 | 1.0 | 8 | 4 | 0.030 | heavy, planted, a little slower-looking |
| Earth, armoured | 0.9 | 0.8 | 0.8 | 6 | 2 | 0.035 | a stomping walk; the armour's footsteps are stone |
| Air | 1.1 | 1.25 | 1.1 | 4 | 7 | 0.012 | light and bouncy; the arms swing wide; airbenders **land softly** (the shared `LAND_SOFT` with half the dip) |
| Air Wheel | 0.6 | 0.6 | 0.4 | 14 | 3 | 0.010 | legs hidden in the wheel |
| Water | 1.0 | 1.1 | 1.0 | 5 | 6 | 0.018 | smooth, the arms flowing (an extra 4° y sway on the arm swing) |
| Fire | 0.95 | 1.2 | 1.05 | 7 | 8 | 0.022 | sharp, forward, the fists closed (y −10) |
| Blood (while holding) | — | — | — | — | — | — | cannot move |

Sprinting leans by `lean`, as Enhanced does. **Idle at rest** (not holding a key): the element's `IDLE` with weight
0.4 (a hint of the stance: a bender never stands quite like a villager).

### 7.2 Jumping and landing

Every bender uses Enhanced's `LAND_SOFT` (a dip) on landings over 3 blocks; airbenders at half dip with a puff;
earthbenders in armour with `LAND_HERO`'s deeper dip, dust and a 0.3 shake. Earth Launch, Air Jump, Fire Jet and the
Spout collapse all end in the shared `LAUNCHED` stance (11.3) while airborne.

### 7.3 Swimming

Vanilla owns the body while swimming (Enhanced's rule). Waterbenders get **no** special swim pose in 1.x; the passives
are speed. Air Bubble's sphere follows the head in the swim pose. **(suggestion)** a waterbender swim stroke later.

---

## 8. (Reserved: see 10 for rides and 11 for shared stances.)

---

## 9. Idle stances summary

| Element | Ready stance (key held) | Resting hint (weight 0.4) | Ambient master effect |
|---|---|---|---|
| Earth | `EARTH_IDLE` horse stance, drop 0.24, breathing fists | shoulders heavy, feet apart | grains rising at the feet |
| Air | `AIR_IDLE` circling palms, swaying weight | palms open, light sway | a circling breeze |
| Water | `WATER_IDLE` wave-hands figure-eights, soft knees | one foot forward | a droplet orbiting the hands, full-moon motes |
| Fire | `FIRE_IDLE` bow stance, fists chambered, visible breath | fists loose, forward lean | a knuckle flame, heat shimmer |

---

## 10. Rides, flight and forms

### 10.1 The Air Scooter ride (`AIR_RIDING` stance; the vehicle `AirScooterEntity`)

The body **crouches on the ball**: legs folded (x −55/−55, z ±28: knees out), drop 0.45 (the ball is 0.9 high and the
rider's seat is at its top, so the entity's passenger offset places the hips; the stance's drop is relative to the
seat), lean `6 + 10·fwd`, roll `12·side` (banking into turns), arms **out for balance** (−30,∓10,±45) with a wobble (±3 at
0.5 rad/tick scaled by speed), head level; a bob 0.03 at period 7 (the ball's spin). Hop: a `drop` kick −0.15. Wall-ride:
roll 60° toward the wall (the body leans into it; the ball is on the wall). Passenger (M8): the same pose behind, arms
holding the rider's waist (aim(ANCHOR) at the rider). Dismount: `AIR_SCOOTER_DISMOUNT`.

### 10.2 The glider pose (`AIR_GLIDING` stance; `FALL_FLYING` with the glider open)

Vanilla's elytra owns the body when `isFallFlying` (Enhanced hands over). Bending **overrides** it for the glider (14.3
`rideOverride` includes "fall-flying with the glider"): the body **horizontal** (lean 85–100 by speed via a `FlightProfile
GLIDER`: p1 88, p2 100, pBack 70, kClimb 0.3, diveAdd 10, maxRoll 35, kTurn 0.9, kStrafe 0.4, tauRoll 5, no spring, bob
0.02 period 40, armDrift 0, legSway 2, legFlutter 3, lazyRoll 2, noBrake false), **both arms up gripping the staff**
overhead (−150,∓12,±8: the hands at the glider's crossbar, `mWorld` not used: the arms follow the body), legs **trailing
straight and together** (x 0/0 z ±2, a slow flutter ±3), head looking along the flight (the shared head rule keeps the
look). The **glider layer** (`LOOKS.md` 5): a staff with two wings rendered from the hands' midpoint, wings angled 12°
up, flexing with speed (the wing parts' z rotation ±5°·speed at 0.4 rad/tick). Launch: `AIR_GLIDER_OPEN` (4.3). Air
Blast downward: `AIR_BLAST_DOWN` (4.1): the body pitches up 10° (lean 80) and the hands stay on the staff (the blast comes
from the feet in this case: effects from `HandTracker.foot`). Landing: the wings fold (`B` over 4 ticks) and `LAND_SOFT`.
First person: the hands are **up out of view** (both hands `dy +0.46, dz +0.10`: only the staff's lower end and the
wings' shadow edge are drawn: a glider underside quad above the camera, translucent cloth); the wind streaks (Enhanced's
first-person wind) at speed; FOV ×1.06.

### 10.3 The Water Spout ride (`WATER_RIDING`; `WaterSpoutEntity`)

Standing on the column: legs **apart and bent** (x −12/−12, z ±14), drop 0.10, arms **out and low** (−30,∓10,±48) swaying
with the spin (twist 6·sin(0.2·age), arms z ±4 counter), lean `8·fwd`, roll `6·side`, a bob 0.04 at period 16; the
head level. Ram: `WATER_SPOUT_RAM` (a lean kick). Freeze (M8): still. Surf (`WATER_SURFING`): the surfer's crouch (5.4).
First person: 5.4.

### 10.4 The Earth Pillar rise (`EARTH_RIDE_UP`)

While the pillar lifts you: arms held high (−60,0,±30) with the palms up, legs together (±4), drop 0.04, head −4: a
rising hero; the server moves the player (3.3). At the top the stance fades (tauOut 4) into `EARTH_IDLE`.

### 10.5 Meditation (`AIR_MEDITATING`)

`AIR_SIT`: cross-legged (legs x −90 z ±55: **NEW** leg z in rows is Enhanced's; in keys it is 14.1), drop 0.72, hands on
the knees palms up (−20,±30,∓40), head 4 (slightly bowed), a **breathing sway**: lean 1.5·sin(0.05·age) and the hands
rising 2° with the inhale, the head ±1. Eyes **closed**: a `GlowingEyesLayer`-style mask that draws **closed lids** (a
skin-coloured strip over the eyes: `eyes_closed.png`, cutout, tinted by the skin's face colour sampled at runtime from the
skin texture: **NEW** 14.9) while meditating. Deep Meditation: the same with a soft glow. The sitting is also what the
**body** shows while the spirit is out.

### 10.6 The spirit form (`SpiritEntity`, `SPIRIT_FLOAT` stance) and Zaheer's flight

**Spirit:** the player's model drawn translucent blue (`AstralBodyRenderer` pattern: `entityTranslucent`, tint `#9FDCFF` α
0.45, no armour or items, a faint `energySwirl` aura α 0.15, the name tag at α 0.4), floating in `AIR_SIT` legs with the arms
**free** (−30,∓20,±30 drifting ±6 at 0.1 rad/tick), drop 0 (it floats at its own position), a bob 0.08 at period 50, a
slow lean into movement (`lean 15·fwd`), roll `8·side`; **the flight profile `SPIRIT`**: p1 20, p2 35, pBack −10, kClimb
0.2, diveAdd 15, maxRoll 20, kTurn 0.6, kStrafe 0.3, tauRoll 6, no spring, bob 0.08/50, armDrift 6, legSway 0 (the legs
stay folded), legFlutter 0, lazyRoll 3, noBrake true (a spirit stops dead), defaultSpeed 0.08. Passing through blocks:
the renderer draws it anyway (no culling by blocks; it is translucent). Effects: 2 blue motes/tick trailing, the tether
thread (4.5).

**Zaheer's True Flight (`FlightProfile ZAHEER`):** p1 35, p2 70, pBack 0, kClimb 0.5, diveAdd 20, maxRoll 30, kTurn 0.8,
kStrafe 0.5, tauRoll 5, no spring, bob 0.06/60, armDrift 4, legSway 3, legFlutter 2, lazyRoll 2, noBrake false,
defaultSpeed 0.06; anchors: **HOVER** = `ZAHEER_HOVER` (cross-legged `AIR_SIT` legs, hands on knees), **CRUISE** = arms
**swept back** (30,∓10,±30), legs together trailing (x 10/10), the body leaning 35; **SPRINT** = arms back and tight
(40,0,±14), legs straight (x 20/20), lean 70: Zaheer's arrow. Take-off `AIR_TAKEOFF_ZAHEER` (4.9) ends on the hover
anchor (the shared test rule). Brake: `BRAKE` (Enhanced's) with the arms flaring forward. Landing: `LAND_SOFT` with a
feather dip.

### 10.7 Fire Jet flight (`FlightProfile FIRE_JET`)

Short bursts only (3–6 s): p1 15, p2 30, pBack −5, kClimb 0.3, diveAdd 10, maxRoll 25, kTurn 1.0, kStrafe 0.6, tauRoll 3,
**spring** (omega 0.35, zeta 0.5: Ozai's jerky thrust), bob 0.02/6, armDrift 0, legSway 0, legFlutter 0, lazyRoll 1,
noBrake true, defaultSpeed 0.07; anchors: HOVER = `FIRE_JETTING` (fists down and back, legs straight back), CRUISE =
the same with lean 15, SPRINT = lean 30, fists further back (40,∓10,±30). No take-off gesture beyond `FIRE_JET_START`;
landing `LAND_SOFT` + a fiery dip.

---

## 11. Shared stances and transitions

### 11.1 Armour forming and crumbling

**Earth Armour** (3.2): bottom-up in 6 slices over 14 ticks (feet, shins, thighs/hips, chest/arms, shoulders, head), each
slice revealed with 6 grains and a stone clunk; crumbling top-down in 6 ticks with the slices' grains. **Steel Armour**
(3.8): top-down in 5 slices over 5 ticks (head, chest, arms, legs, boots) with white motes and claps; peeling off
top-down. **Water Octopus** (5.3): the body veil bottom-up over 14 ticks and the tentacles growing root to tip; collapsing
top-down into a puddle in 6 ticks. The slice reveal is one shared helper, **`LayerReveal`** (**NEW** 14.10): a model-space
clip that shows parts of a layer when a plane (y) passes them, with a per-part "pop" (scale 1.1 → 1.0 over 2 ticks) and a
particle burst callback.

### 11.2 The glider opening and closing, the scooter mount, the spout rise

4.3, 4.7, 5.4: all in group `RIDE` (so they never cancel a `CAST`), each ending on its ride stance.

### 11.3 `LAUNCHED` (airborne after a launch)

While `Live.LAUNCHED` (set by Earth Launch, Air Jump, Fire Jet's end, Spout collapse, Air Blast's self-knockback, Blood
Fling on you, Tornado): arms **wide and back** (−70,0,±70 → drifting to −40,0,±50 as the fall continues), legs
**trailing** (x −20/20 alternating slowly ±10 at 0.2 rad/tick: a kick), head −6 (looking ahead/down), lean `−10 + 30·(vy
< 0)` (leaning forward once falling), a small roll wobble ±4. It ends on landing (`LAND_SOFT`) or water (`SPLASH` ×12 and the
swim pose).

### 11.4 `LIFTED` and `BLOODBENT` (held in the air by someone else)

Enhanced's `Live.LIFTED` pose (arms wide, legs trailing, drop −0.3) for Metal Grip and Tornado; `BLOODBENT` is 12.4.

### 11.5 Transitions between elements' looks

Choosing an element (the awakening) plays `AWAKEN_<ELEMENT>` (dur 40 / 4 / 10, locks 40 ticks): Earth: a stomp and both
arms rise (the ground ring); Air: a spin (yaw 360) with arms out and a lift (drop −0.3 then down); Water: the gather and a
slow outward circle with a water ring; Fire: a deep breath, fists ignite, both arms thrust up (−176) with a column of
flame. Changing a look (Sneak + O): Enhanced's `LOOK_CHANGE` (a quick turn) with the element's burst (20 motes).

---

## 12. Hits and defeat: what a target shows

Reactions are gestures in group `HIT` on players (never cancelling a `CAST`; they layer by weight) and, for mobs, a
shared **`VictimPoses`** hook (**NEW** 14.11): a `LivingEntityRenderer.setupRotations` mixin tail that tilts/shakes any
living entity carrying one of the mod's effects or `Live`-like synced flags (the mod attaches a small synced `VictimState`
to mobs it hits: `bending:victim` attachment, 1.5 s), and a `HumanoidModel.setupAnim` tail for humanoid mobs (zombies,
skeletons, villagers, piglins, illagers) that keys their arms and head for the puppet and paralysis poses.

### 12.1 Earth: stagger, knockdown, stuck

| Reaction | Players (`HIT` gesture) | Mobs (`VictimPoses`) | Effects |
|---|---|---|---|
| `HIT_STAGGER` (rock, wall push, knife, whip, fireball) | dur 10: lean −14 at 2 (`O`), head +6, arms flail +20 x ADD, legs 10/−14; back to 0 at 10 (`E`) | body pitch −12 → 0 over 8 ticks (renderer tilt), head yaw jitter ±6 for 4 ticks | 6 grains / sparks / splashes by element at the hit |
| `HIT_KNOCKDOWN` (boulder, Fire Kick, blood fling landing, redirect fail) | dur 20: lean −40 at 3 (`O`), drop 0.5 at 6 (`I`: on the ground), arms up (−120,0,±40), legs x −40/−40; rise: lean 0 and drop 0 at 20 (`E`) | pitch −60 for 10 ticks then up (the mob lies back), dust | a dust puff |
| `STUCK` (quicksand, frozen feet) | stance while the effect lasts: legs **still** (the walk cycle weight → 0: 14.3), arms pulling up (−100,∓30,∓20) with a tug every 20 ticks (ADD +15 x for 4 ticks), lean −6 | legs frozen (a `walkAnimation` speed 0 via the state), the body shivers (frozen) or sinks 0.4 (quicksand: the renderer translates −0.4 y over 20 ticks) | ice block at the feet / the sand collar |
| `LIFTED` (Metal Grip, Tornado) | Enhanced's | the renderer lifts and tumbles: pitch 20·sin, roll 15·cos at 0.3 rad/tick | motes |

### 12.2 Air: tumble, clutch, spin

| Reaction | Players | Mobs | Effects |
|---|---|---|---|
| `HIT_TUMBLE` (Air Blast, Air Slice, Tornado fling) | `LAUNCHED` plus a **roll**: roll 0 → 360 over 12 ticks (a `roll` track in the gesture: **NEW** 14.1 roll keys are Enhanced's `roll(v)`: exists), arms wide; ends on landing | the renderer rolls the body 360° over 12 ticks and pitches −30 | 6 white motes trailing |
| `CLUTCHING` (Suffocation) | stance: both hands at the throat (−130,∓50,∓30), head +14 (bowed), lean 12, drop 0.14, a shudder (tremble 2 at 3.0), knees buckling (drop 0.14 → 0.26 over the hold); every 20 ticks a gasp (head −10 for 3 ticks) | head down, body pitch 15, shivering; humanoids: arms to the neck | gasp motes |
| `SPINNING` (inside a tornado) | `LIFTED` with a yaw spin 30°/tick | the renderer yaws 30°/tick and lifts | |

### 12.3 Water: soaked, burning, chilled

| Reaction | Players | Mobs | Effects |
|---|---|---|---|
| `SOAKED` (Water Shot, whip, spout) | no pose; a flinch (`HIT_STAGGER` at half weight) | a flinch | **drips**: 2 `DRIPPING_WATER`/tick from the model's lowest points for 3 s + a darker tint (`LivingEntityRenderer` colour ×0.9 for 3 s: the wet look: **NEW** 14.11 colour hook) |
| `BURNING` (any fire) | vanilla's fire render plus `HIT_FLAIL` (dur 12: arms shake tremble 4 at 4.0, head −8, lean −6 → 6) once per ignition | the vanilla fire; a shake | |
| `CHILLED` (ice knives) | a shiver: tremble 2 at 5.0 on both arms for 20 ticks (ADD rows via `CHILLED` state), head +4 | the renderer shivers (x/z jitter 0.01) | frost crackles on the body (`ENERGY_BEAM` white lines α 0.5), 1 `SNOWFLAKE`/2 ticks, a blue tint ×0.9 |

### 12.4 Blood bending: the puppet

**Players (`BLOODBENT` stance, tauIn 1 / out 6):** `PUPPET`: arms **wrenched up and back** (−100,∓40,∓60: the elbows
out, the hands high behind), the legs **apart and bent** (x −20/30, z ±25), the head **back** (−10) and turned (y 20), lean
−10, **lifted** (drop −0.4: the server lifts the entity; the stance adds the body's hang), a **tremble 3 at 2.6** on
everything (the strain), and **the limbs follow the claws**: the caster's M claw's elevation drives the victim's arm x
(ADD −20·clawUp), the O claw drives the legs (the victim's `aim(ANCHOR)` on the caster's hands, 2 ticks behind): when the
caster raises a hand, the puppet's arms rise; when they lower it, the puppet sags. The victim's own input is locked
(`MovementInputUpdateEvent`), the camera free (so they see the caster), the view tilting with the body (2.2). On release:
the puppet **drops** (`LAUNCHED` → `LAND_SOFT` / `HIT_KNOCKDOWN` if from height). **Mobs (`VictimPoses`):** the renderer
lifts the entity (the server does), pitches the body −10 → +10 following the caster's O claw, rolls ±8 with the tremble,
and humanoid mobs get the arms up and back (the `HumanoidModel` tail: arms x −100, y ∓40, z ∓60) and the head back; animals
(quadrupeds) get legs splayed (a `QuadrupedModel` tail: legs x ±40 outward, **NEW** 14.11) and the body rolled 15.
Effects: the red threads (5.7), 1 red mote/tick.

### 12.5 A bender losing their bending (Bending Severance) and paralysis

**Severed (`SEVERED`, a one-shot `BLOOD_SEVERED` dur 60 / 4 / 16, locks 60 ticks):** at the touch, the body **goes
rigid** (t 0–8: `PARALYSED`-like, arms out, tremble 1 at 3.0), then **sags**: t 16 the arms fall (10,0,±14), the head
drops (+18), the knees give (drop 0.36, legs x −30/−30 z ±12: a kneel), lean 10; t 16–48 kneeling, swaying slightly, the
head slowly lifting (+18 → +8); t 60 rising slowly (`E`) to `NEUTRAL` with the head still down (+6: the `SEVERED` stance
keeps head +6 and arms 2° heavier for the whole severed time, and the element's idle hint is **gone**: weight 0). Effects:
the element's colour drains from the body (an `energySwirl` in the element's colour α 0.4 → 0 over 20 ticks flowing
**upward and out** of the body: 16 motes of the colour rising from the chest), then a grey dust ring; the HUD greys. The
victim's first person: the hands fall limp to the bottom corners for 60 ticks (`dy −0.34, rx +50`), the camera lowers 0.3
(the kneel) and a desaturation post effect (saturation 0.6 for 60 ticks, then 0.85 for the severed duration via
`spirit.json`'s sibling `severed.json`: Saturation 0.85, ColorScale 1/1/1). Sound: `severed_victim` (the high ring fading
into muffled quiet: a low-pass on the music for 3 s is **not** possible on vanilla's sound engine; the ring alone).

**Paralysed (`PARALYSED`, while `bending:paralysed`):** players: the stance `PARALYSED` (arms out rigid, legs locked
apart, head −8, a **fast tremble 2 at 6.0** on arms and head, lean −6), input locked; the walk cycle weight → 0; mobs: the
renderer shakes (x/z jitter 0.02 at 6 rad/tick) and the humanoid tail puts the arms out (−20,0,±46), the head back;
effects: 2 `ELECTRIC_SPARK`/tick crawling on the body, a white-blue flicker tint (`#DDF4FF`, colour hook, 3 Hz), the eyes
white; sound `paralysed_loop` (a buzzing crackle, volume 0.4) at the victim; the first-person overlay 2.2. Release: the
body **slumps** (`HIT_STAGGER` at full weight: lean −14 then up) and 6 sparks fall off.

---

## 13. Budgets: the four-bender fight

The worst case the design is tuned for: an earthbender in armour behind a wall throwing rocks, an airbender on a scooter
with a shield up shooting blades, a waterbender with the octopus whipping, a firebender flamethrowing, all within 32
blocks of one camera, at *Effect density* 1.0 and vanilla Particles "All":

| Source | Particles/tick | Meshes/frame | Entities |
|---|---|---|---|
| Earth: wall (standing) + armour + 3 rocks | 2 + 0.1 + 3 + impacts (22 per hit, ≈ 1 hit/s) ≈ **6/tick avg** | wall dust 0, armour layer 1, rocks 3, rings ≤ 2 | 3 rocks |
| Air: scooter + shield + 3 blades | 3 + 2 + 3 + hits 6 ≈ **9/tick** | ball 1 (+4 rings), shield 1 (+3 rings), blades 3, cones ≤ 2 | 1 ball, 3 blades |
| Water: octopus + whip | 0.5 + 1 + slaps 6/2 s ≈ **5/tick** | octopus veil 1 + 8 chains, whip 1 chain | 0 |
| Fire: flamethrower (2 jets) | 18/tick + hits 9 ≈ **22/tick** | 2 beams (5 layers each) + tongues ≈ 10 draws | 0 |
| **Total** | **≈ 42/tick → with 10–21-tick lifetimes ≈ 600 live particles**, under vanilla's 16,384 cap and comparable to a vanilla rainstorm | **≈ 35 batched draws** in 4 render types (one `endBatch` each) | 7 |

Rules that keep it there: counts above are **per tick caps**, not "per effect spawn" guesses; `density()` scales
everything; nothing beyond 48 blocks spawns; meshes beyond 96 are skipped; `MAX_EFFECTS 160` as `SandFx`; each
`*Fx` class keeps a bounded list and drops the oldest; one `RenderLevelStageEvent` subscriber (`BendingWorldFx`) draws
every element's meshes in **four batches** (`MATTER_FLAT`, `MATTER_TEXTURED`, `ENERGY_VEIL`, `ENERGY_BEAM`), never one batch per
effect. Server side, a 4-bender fight runs ≤ 7 projectile entities, ≤ 2 ride entities, ≤ 2 walls (≤ 112 temp blocks),
1 pillar, and the hazards (quicksand patches, lava paths, tornadoes) are **ticked as server-side objects**, not entities,
except the tornado (an entity so it can carry riders). Payloads: one FX payload per event (not per tick) except keep-alives
(beams every 8 ticks, tornado position every 4 ticks), as Enhanced does.

---

## 14. Reuse: what comes from Enhanced, what is new

### 14.0 Reused as is (copied into `bending.client.render`, package name changed, nothing else)

`GestureId` (shape; Bending's own constants), `Gestures` (+ `Def`, `Builder`, `ArmKey`, `Track`, `Mode`, `Target`),
`GestureTable`, `GestureRoutes` (Bending's own arms), `GestureEffects`, `HeroAnimator` (renamed `BenderAnimator`),
`HeroAnimState`, `HeroPoses`, `PoseContext`, `Stance`, `Stances`, `StancePoses`, `FlightProfile`, `FlightPoses`,
`FlightGestures` (takeoff/land/brake keys), `RunCharacter`, `RunPoses` (minus the speedster styles), `FirstPersonArms` (+
`Offset`), `FirstPersonMoves` (the 14 rows), `AnimMath`, `BodyLean`, `HandTracker` + `HandTrackerLayer`, `AimMath`,
`HeroClient` interface (renamed `ElementClient`), `HeroClients`, `AnimDebugCommands` (`/bending_anim`), `AnimStateOverlay`,
the mixins `PlayerModelMixin`, `PlayerRendererMixin`, `AbstractClientPlayerMixin` (skins), `ItemInHandRendererAccessor`,
`CapeLayerMixin`, `HumanoidArmorLayerMixin` (hide armour under suits), the FX toolkit (`EnergyGeometry`, `FxShapes`,
`SandShapes` → `MatterShapes`, `EnhancedRenderTypes` → `BendingRenderTypes` with `MATTER_FLAT`/`MATTER_TEXTURED` =
`SandRenderTypes`), `BeamStyle`/`BeamStyles`/`BeamSpec`/`ContinuousBeam`/`BeamPayload`/`BeamRenderer` (+ a `Drawer` for
the flame cone), `ShieldStyle`/`ShieldStyles`/`ShieldFx` + the dome passes of `WorldEffectsRenderer`, `FlightEffects`
(trails, rings, wind, `lay`, `polyline`), `EnergyParticle`/`EnergyParticleOptions`, `PowerAuraLayer` (the swirl aura),
`GlowingEyesLayer`, `AstralBodyRenderer`/`AstralLayer` (the spirit), `Afterimages` (not used in 1.x), `EntityGlow` + its two
mixins, the overlay recipes (`PhoenixOverlay` vignette, `ApocalypseOverlay` gradient), `SlowMotionEffect` (post-shader
loading, for `spirit.json`/`severed.json`), `ArenaFx.shake` (camera shake), `PowerLoopSounds`, `PinLock`/`EntombedLock`
(input locks, generalised to a `Live`-bit set), `TelekineticBlockEntity` + renderer (the Rock entity's base: `renderSingleBlock`
of a state) and `Telekinesis.hold` (Hold Rock, Metal Grip), `LegIk` (not needed in 1.x), `PreviewPuppet` (screens),
`HudDraw.visual` (RTL).

### 14.1 Building blocks that are NEW (and which lane builds them, `PLAN.md`)

| # | Block | What | Lane |
|---|---|---|---|
| 14.1 | **Leg z in gesture keys** | `Gestures.Builder.legs(rx, rz, lx, rz)` and a `Track legZ` (width 2), sampled like `legX`; `PoseContext.leg` already takes z. Needed by the horse stance's stomps in keys, Fire Kick, the sweep, meditation, the scooter crouch in gestures | 1.0 ANIM (core) |
| 14.2 | **Groups `RIDE`, `HIT`, `AUTO`** | three more `GestureId.Group`s and `HeroAnimState.SLOTS` 3 → **4** (so a cast, a ride, a hit reaction and an auto strike can overlap); `AUTO` and `HIT` gestures never take a slot from `CAST` (the slot picker prefers freeing the oldest non-CAST) | 1.0 ANIM |
| 14.3 | **Channel masks and body ownership** | per-gesture masks: `legsMaskedByAir` (legs keys × (1 − wAir)), `legsMaskedByWalk` (stance leg rows × (1 − limbAmount/0.2)); a `Live`-driven **`bodyOwner`** flag that forces `wAir/wRun/wSprint` to 0 and ignores `vanillaOwnsBody` for `isPassenger` of the mod's rides and `isFallFlying` with the glider (`rideOverride`); needed by meditation, the scooter, the spout, the glider | 1.0 ANIM (meditation and glider in 1.1 use it) |
| 14.4 | **Camera drop and eye-height offsets** | a `ViewportEvent.ComputeCameraAngles`... position hook (`CalculateDetachedCameraDistance` is for third person; for first person use `ViewportEvent.ComputeCameraAngles` plus a `Camera` position nudge via a small `CameraMixin.setup` tail (**one new mixin**)): the gesture's `drop` applied to the first-person camera (×0.5, reduced-motion gated) and a per-`Live` eye offset (meditation −0.9, the sever kneel −0.3) | 1.0 ANIM |
| 14.5 | **Particles `bending:grain` and `bending:droplet`** | two `TextureSheetParticle`s with gravity (grain: 0.06, bounces once; droplet: 0.08, a `SPLASH` on landing), tint from options (grain: a `BlockState` tint or RGB; droplet: RGB), 20–40 ticks, `PARTICLE_SHEET_TRANSLUCENT`, shadow-lit (not fullbright: matter) | 1.0 FX |
| 14.6 | **Gesture chaining** | `HeroAnimator.startAfter(s, id, afterId)`: a gesture queued to start when another ends (the water source draw, the great fireball's inhale) | 1.2 ANIM (needed from Water; a `param` delay stands in before) |
| 14.7 | (suggestion) fake light discs | a warm `MATTER_FLAT` disc under flames | later |
| 14.8 | **Tracked body centre** | `HandTracker.belly(player)`: the body part's centre in world space (for the redirection path, the blood threads' anchor) | 1.3 ANIM (FIRE) |
| 14.9 | **Closed eyes** | an eye-lid mask layer tinted by the skin's sampled face colour (`NativeImage` read once per skin, cached) | 1.1 LOOKS |
| 14.10 | **`LayerReveal`** | model-space bottom-up/top-down reveal of a `ModelPart` layer in slices with pop and burst callbacks | 1.0 LOOKS/FX |
| 14.11 | **`VictimPoses`** (mobs) | `LivingEntityRendererMixin` tail (tilt, lift, shake, colour tint) and `HumanoidModelMixin` + `QuadrupedModelMixin` `setupAnim` tails reading a synced `bending:victim` attachment (puppet, paralysed, stuck, chilled, soaked), 1.5 s TTL | 1.0 ANIM (stagger/knockdown/stuck), 1.2 (puppet), 1.3 (paralysed) |
| 14.12 | **Verlet chains** | a tiny client rope sim (`ChainSim`: N points, gravity vector, damping, pinned root, a driven end) for the water whip, octopus tentacles, water arms, fire whip; drawn as ribbon chains | 1.2 FX (Water), reused 1.3 |
| 14.13 | **Mesh entities** | renderers for `RockEntity` (block display, tumbling), `AirBladeEntity` (crescent), `AirScooterEntity` (sphere + rings), `TornadoEntity` (funnels), `WaterShotEntity` (ribbon), `WaterSpoutEntity` (funnel), `IceKnifeEntity`, `SteelKnifeEntity`, `FireballEntity`, `LavaShurikenEntity`, `SpiritEntity`, `SandStreamEntity`; one `BendingWorldFx` subscriber batching hazards' meshes (walls' dust, quicksand collars, lava shimmer, rings, bolts, threads) | per element lane |
| 14.14 | **Screen overlays** | `ArmourSlitOverlay`, `SpiritOverlay`, `BloodOverlay`, `ParalysisOverlay`, `QuicksandOverlay`, `BubbleOverlay`, `HeatOverlay`, `RedirectPathOverlay` (screen-space bolt) | per element lane |
| 14.15 | **Flame beam drawer** | a `BeamRenderer.Drawer` for the FLAME style: inverted taper, tongues instead of strands, two beams per player (`offHand`) | 1.3 FX |
| 14.16 | **Lightning bolt FX** | `LightningFx`: the 3-layer branching bolt with reshapes and flash spheres, body-path variant | 1.3 FX |
| 14.17 | **Water surfaces** | `MATTER_TEXTURED(water_still/water_flow)` ribbons and spheres using vanilla's animated water sprites (`TextureAtlas.LOCATION_BLOCKS` sprite lookup) with the biome water tint | 1.2 FX |

### 14.2 Tests that pin all this (the animation lane's `*AnimationTests`, server-side, pure)

As Enhanced's `AnimationMathTests`: every `GestureId` has well-formed keys (`problems()` empty, attack ≥ 1, release ≥ 1, no
ADD/ABS mixing), every yaw gesture ends on a whole turn, every gesture that hands over to a stance ends on the stance's
`StancePoses` constant (no pop), every ability/toggle/Live/flag is acted or explained (`GestureRoutes.acting`), no
gesture is orphaned, every flight profile's take-off ends on its hover anchor, the new leg-z track obeys the same
rules, the `VictimPoses` tables are within joint limits, the first-person offsets stay under 0.8, and each element's
`*AnimationTests` checks the timing of each gesture (the tables above are the expected numbers: the test reads the
`Def` and asserts the key at tick t equals the table's pose within 0.5°), which arm each drives, the prediction rules
and the cooldown-length routing. What the poses look like is checked by eye in a real client with `/bending_anim play
<gesture> <tick>` and `/bending_anim target <player>` from a second client, in both languages, three render rounds per
lane (`TECH.md` 8).
