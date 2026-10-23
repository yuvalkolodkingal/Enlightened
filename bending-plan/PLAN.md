# Bending — Build Plan

The method is Enhanced's (3.0 to 3.8): a **skeleton step S0** does every shared edit once (registries, payloads,
attachments, config options, lang keys in both languages, sound placeholders, tags, the plug-in stubs, the seams with
neutral stubs, a protocol bump pinned by a test), then **lanes** work in parallel, each in its own **git worktree and
branch** from the skeleton, each **owning its packages** and writing into shared files **only in its own region**
(`// ===== <lane> (1.x) =====`), never editing a lang file, a config class, `ModNetwork`, a payload record,
`ModAttachments`, `ModTags`, `ClientSetup`, `TestKit` or `DataTests` outside its region (a needed change is a
`request(I)`: a ten-line commit for the seam keeper, merged within the day). A **seam keeper** (one agent, the skeleton's
owner) answers requests and keeps the skeleton branch green. An **integrator** merges the lanes into the release branch
in a fixed order, does the cross-lane joins, runs the lanes' batches and then the **full suite**, and ships. The **gate**
for a lane to merge: the build is green (`portLint` both loaders), its tests pass twice in a row on both loaders, its lang
block is complete in both languages, its real-client checklist is done by someone other than the author (the integrator
or a sibling lane), it edited no file outside its ownership, no existing test changed. The gate for a release: the full
suite green on both loaders with matching parity reports, the real-client check of every lane, the universal jar built
and run on both loaders, and the per-release checklist below.

Conventions in every brief: the design is `DESIGN.md` (D), the animation bible `ANIMATION.md` (A), looks `LOOKS.md` (L),
screens `UI.md` (U), sounds `AUDIO.md` (S), architecture `TECH.md` (T). Numbers in a brief come from those files; a lane
never invents a number: it reads the config option the skeleton defined.

**Estimates** are in **agent-days** (one autonomous build agent, one day ≈ a long session with tests) and calendar
days with the lanes in parallel. Enhanced's releases (8 lanes, ~450 lane tests, ~2,500 total) took about two to three
weeks each in calendar time with the lanes parallel; the estimates below assume that pace.

---

## Release 1.0 — Core + Earth (כשפות אדמה)

**Contents:** the repository, the platform (both loaders, the universal jar), the bender data model, progression,
the loadout and keys, the Element Choice, the Bending screen, the HUD, the animation and FX toolkits, temp blocks
and rights, the audio pipeline, **every Earth ability** with its gestures, effects, sounds and looks (Toph default),
GameTests and the real-client check. Network protocol **"1"**.

### S0-1.0 — the skeleton (seam keeper; 3–4 agent-days; branch `v10-skel`)

1. **Repository**: `bending` with `common/`, `neoforge/`, `fabric/`, `processor/`, `tools/`, `docs/` (this plan's eight
   documents under `docs/plan/`, `docs/DESIGN.md` = D rewritten as the living design, `docs/wiki/` Home EN/HE), the CI
   workflows, `.gitattributes`, the license line as Enhanced's.
2. **Platform copy**: every class of T 2.1 copied from Enhanced with the package and mod id changed; the processor; the
   config backend; the GameTest adapter; `portLint` with the `fabric` rule; `META-INF/services`; the NeoForge entry points
   and a **stub** `platform/fabric/` (`FabricPlatform` etc. as compiling no-op shells throwing `UnsupportedOperationException`
   with "L0-FABRIC fills this") so the Fabric subproject compiles from day one.
3. **Data model**: `Element`, `Ability` (every card of all four elements, all releases: ids, elements, acts, costs, parents,
   gates: so later releases never reorder), `BenderData` + `BenderSave` + `WorldSave` + `bending:victim` (T 5), `Benders`,
   `BenderTicker`, `AbilityHandler`, `Loadout`, `Mastery`, `ElementStrength`, `SkillTree`, `Cooldowns`, `Live` (every bit of
   all releases, in element regions), `Toggle`, `Severance` and `ZaheerPath` as data holders with no behaviour.
4. **Registries**: every item, block, block entity, entity type (**all releases**, with placeholder entity classes that
   do nothing: so ordinals never move), damage type JSON, effect, sound event (**232**, with `tools/sound/placeholder.ogg`),
   particle type, tag file, data component, advancement JSON, creative tab; `ModAttachments`; `ModNetwork` with **every
   payload of 1.0** (handlers calling lane stubs) and protocol "1".
5. **Config**: `BendingConfig` with every section and **every option of all four elements** (D's numbers as defaults, keys
   named in T 5.4), `BendingClientConfig` (U 2.3); every option's comment names its unit.
6. **Lang**: every key of 1.0 in `en_us.json` and `he_il.json` (elements, abilities with Sneak variants, HUD, screens,
   messages, subtitles for all 232 sounds, items, blocks, effects, death messages, config translations, key names), with a
   region per lane marked by sentinel keys `"_lane.earth": ""` removed at release (Enhanced's 3.0 rule) and
   `tools/add_lang.py` to merge blocks by union.
7. **Plug-in stubs**: `EarthServer`/`EarthClient`, `AirServer`/`AirClient`, `WaterServer`/`WaterClient`, `FireServer`/`FireClient`
   registered in `ElementServers`/`ElementClients`, every method a no-op; `ElementFx` senders; `TempBlocks`, `BendingRights`,
   `Hazards`, `Grip`, `AreaTargets`, `Raycasts`, `SourceFinder`, `MetalCharge`, `Victims` as **interfaces with neutral
   implementations** where another lane fills them (the seams), each pinned by `SkeletonTests.theSeamsAreNeutralUntilTheLanes`.
8. **Client copy**: the animation system, the FX toolkit, the UI kit, `PowerLoopSounds`, overlays' base, `ClientSetup`
   with a region per lane, `KeyBindings`, `ClientEvents`, `BendingHud` skeleton (header + 4 empty slots), `BendingScreen`
   with four empty tabs, `ElementChoiceScreen` shell; `GestureId`/`Stance`/`RunCharacter` enums with **every constant of
   all releases** in regions (Enhanced appends only; here they are all present from S0 so the lanes never collide);
   `GestureRoutes` arms per lane.
9. **Tests**: `TestKit` with the new helpers (T 14.1), `SkeletonTests`, `DataTests`, `SoundAssetTests` (with the 1.0
   placeholder list), `NetworkTests`, `PlatformSeamTests`, `ConfigFileTests`, `AnimationMathTests`, `AnimationSignalTests`
   (empty regions per lane), `tools/parity.py`. **The full skeleton run passes on NeoForge** (`All N required tests passed`,
   N ≈ 120) and compiles on Fabric.
10. **Docs**: `docs/HEROES-AND-BOSSES.md`'s equivalent `docs/LANES.md`: the regions, the seams, the rules, this release's
    lanes and merge order.

### Lanes of 1.0 (in parallel from `v10-skel`)

| Lane | Owns | Agent-days | Depends on |
|---|---|---|---|
| **L0-FABRIC** | `platform/fabric/**`, `fabric/` build, `bending.fabric.mixins.json`, the `universal` task, CI's Fabric job | 6–8 | none (the biggest risk: start first) |
| **L1-CORE** | `bender/**` behaviour (progression, SP on kills, mastery XP, unlock/buy, loadout, ES, severance timer, Zaheer flag), `ability/AreaTargets`, `Raycasts`, `BendingRights`, `TempBlocks` (+ journal, recovery), `Hazards`, `Grip`, `Victims`, `command/`, `item/ElementScrollItem`, `compat/` | 4–5 | none |
| **L2-EARTH** | `earth/**`, `entity/RockEntity`, `SandStreamEntity`, `SteelKnifeEntity`, `LavaShurikenEntity`, `block/Quicksand|SteelPlate|TempLava|Wrecked*`, `EarthFxPayload` handlers, `client/render/EarthClient|EarthGestures|EarthActing|EarthFx`, Earth's entity renderers, `ArmourSlitOverlay`, `QuicksandOverlay`, `EarthArmourLayer`, `SteelArmourLayer`, `LayerReveal` | 7–9 | L1's `TempBlocks`/`Grip` API (published by S0 as interfaces; L2 codes against them and runs with the neutral stub until L1 merges: the integrator joins) |
| **L3-ANIM** | the animation core additions (A 14.1, 14.2, 14.3, 14.4: leg z keys, groups RIDE/HIT/AUTO and 4 slots, channel masks and body ownership, camera drop/eye offsets), `VictimPoses` (stagger, knockdown, stuck, lifted, soaked, chilled: A 12.1–12.3) with `HumanoidModelMixin`/`QuadrupedModelMixin`/`LivingEntityRendererMixin`, shared stances (`LAUNCHED`, `LIFTED`), run characters, `LAND_SOFT` variants, the awakening gestures (A 11.5), `AnimDebugCommands`/`AnimStateOverlay` wiring, `AnimationMathTests` extensions | 4–5 | none |
| **L4-FX** | `GrainParticle`, `DropletParticle` (A 14.5), `BendingRenderTypes` (`MATTER_*`), `BendingWorldFx` (the four-batch world renderer with the client ledger), `MatterShapes`, `ChainSim` (A 14.12, used from 1.2 but built now: cheap), camera shake service, overlay base classes, `EntityGlow` wiring, the fake-light option stub, `EffectDensity` | 3–4 | none |
| **L5-UI** | `ElementChoiceScreen`, `BendingScreen` + `SkillTreeTab`/`LoadoutTab`/`LooksTab`/`WaterTab` (the Water tab hidden until 1.2), `BendingHud` (every status line, cooldown sweeps, Sneak swap, SP/ES, placement option), the icon atlas (**all** icons of U 6, drawn now), action-bar messages, RTL layout, `HudTests`/`ScreenDataTests`/`LoadoutTests`/`TreeTests` | 5–6 | L1's progression API (interfaces from S0) |
| **L6-LOOKS** | the six Earth looks (L 1.1–1.6: skins, layers, face icons), the shared layer kinds (`TopknotLayer`, `CrownLayer`, `HairLayer`, `BeardLayer`, `RobeLayer`, `ShoulderLayer`, `BackLayer`, `HoodLayer`, `MaskLayer`, `ArmBandLayer`: built now for all releases), `Looks`/`Look`, skin application (`SuitSkins`-pattern mixin), `LooksTests`, the Element Scroll / Water Skin / Glider item art | 4–5 | none |
| **L7-SOUND** | `tools/sound/*` (the committed toolkit), the **shared and Earth** sounds of S 2–3 (67 events, ~100 files), `PowerLoopSounds` kinds for Earth, `EntitySoundLoop`, the Earth theme and the choice music (S 7), `SoundAssetTests` lists | 4–5 | none |
| **L8-DOCS** | `docs/wiki/` pages EN + HE for 1.0 (Home, Becoming a Bender, Earth, Controls, Looks, Items, Commands, Server Configuration, FAQ, Building), README EN/HE, `docs/DESIGN.md` 1.0 section, the Hebrew `docs/DESIGN.he.md` | 2–3 | the others' final numbers (last to merge) |

**Merge order** (integrator; branch `v10`): S0 → L0-FABRIC (so every later merge runs on both loaders) → L1-CORE →
L3-ANIM → L4-FX → L2-EARTH → L6-LOOKS → L5-UI → L7-SOUND → L8-DOCS; then the joins: `TempBlocks`/`Grip` neutral stubs
replaced by L1's (L2 compiled against the interface), the HUD reading Earth's status lines, the looks in the picker, the
sound events wired to the FX kinds, the icon atlas cells checked against `Ability`, one comma convention for the enum
regions. **Integration: 2–3 agent-days.** Then the **gate**: full suite both loaders (target ≈ 450 tests), parity diff
empty, the real-client checklist (T 14.2) on both loaders and cross-loader, a world moved between loaders keeps the
bender, a 4-bender Earth fight within budget (four earthbenders is the worst case for temp blocks), Enhanced installed
alongside (no crash, both HUDs, keys distinct), `mod_version 1.0.0`, tag `v1.0.0`. **Calendar: ~3 weeks.** Fallback
milestone if L0-FABRIC slips: **1.0-neoforge** (the NeoForge jar alone) ships with Earth; L0 lands in 1.0.1 without
touching gameplay.

### Lane briefs (paste into the agent's prompt; each begins with the shared preamble)

**Preamble (every lane):** "You are lane `<LANE>` of Bending 1.0, a Minecraft 1.21.1 mod (NeoForge + Fabric universal
jar, no runtime libraries, every model a `ModelPart`, every animation code-driven, English and Hebrew everywhere). Work in
your own git worktree on branch `v10-<lane>` from `v10-skel`. You own only the paths listed; in shared files write only
inside your `// ===== <lane> (1.0) =====` region; never edit lang files, config classes, `ModNetwork`, payload records,
`ModAttachments`, `ModTags`, `ClientSetup`, `TestKit` or `DataTests` outside your region: a needed change is a ten-line
`request(I)` commit on a branch `v10-skel-req-<lane>` for the seam keeper. Read `docs/plan/DESIGN.md`, `ANIMATION.md`,
`LOOKS.md`, `UI.md`, `AUDIO.md`, `TECH.md` and `docs/LANES.md` first; every number comes from the config option S0
defined (never a literal). Tests go in your own classes in your own batch (`@BendingTest(template = ARENA, batch =
"<lane>")`); a test that changes a server setting gets its own batch; run `./gradlew :neoforge:runGameTestServer
-Pbending.tests=<Your*>` and `:fabric:runGameTest -Pbending.tests=<Your*>` after deleting `run/world` and
`run/config/bending-server.toml`; the full suite twice before you ask to merge. Check every visual in a real client
(dedicated server, English and Hebrew clients, a second client watching, three render rounds) and put screenshots in
`docs/screenshots/qa/<lane>/`. Every payload handler is one static call into a client class. Commit often with clear
messages; when done, write `docs/lanes/<lane>.md`: what you built, what you left, what the integrator must join."

**L0-FABRIC:** "Implement the Fabric half of the platform layer per `TECH.md` 2.3 and the universal jar per 2.4:
`FabricPlatform`, `FabricHooks`, `FabricItemAccess`, `FabricParts`, `FabricAttributes` (with the mod's `creative_flight`
attribute), `FabricNet` (play channels + the configuration-phase `HelloPayload` handshake using `HelloPayload.check`),
`FabricAttach` (Fabric Attachment API, `SyncTo.SELF` → `targetOnly`), `FabricRegistrars`, `client/FabricClientRegistrar`,
`client/FabricClientServices`, `bridge/FabricBridge` with one adapter per event in `TECH.md` 2.5 (Fabric callbacks where
they exist, Fabric-only mixins in `bending.fabric.mixins.json` where they do not: `LivingEntity.hurt` amount,
`AbstractClientPlayer.getFieldOfViewModifier`, `Camera.setup`, `ItemInHandRenderer.renderArmWithItem/renderPlayerArm`,
`KeyboardInput.tick`, `Projectile.onHit`, `LivingEntity.knockback`, `LivingEntity.canBeAffected`), `FabricTestHooks` and
the `fabric-gametest` entry point. Build: `fabric/` with Fabric Loom and `officialMojangMappings()`, `common/` compiled
once, the `universal` merge task, CI running both loaders' GameTests and `tools/parity.py`. Prove it: `SkeletonTests`,
`PlatformSeamTests`, `NetworkTests` and `ConfigFileTests` pass on Fabric with the same results as NeoForge; a NeoForge client
joins a Fabric server and the reverse; a version mismatch is refused with NeoForge's words; `BridgeTests` for Fabric (the
pattern of Enhanced's `BridgeLivingTests`: post a native callback, assert the common event's getters and cancellation at
the right priorities). Document Fabric's limitations in `docs/LANES.md`. Do not touch gameplay packages."

**L1-CORE:** "Build the bender core per `DESIGN.md` 1 and `TECH.md` 5–8, 12–13: `BenderSave` (SavedData), `Benders.load/
save/sync`, SP on kills (bending damage types and the assist window, the per-mob table and formula, no SP from players),
mastery XP and levels (the curve, the buy cap 7 and costs), unlocks with gates (parent, mastery, SP, ES), ES, the
loadout and presets, `AbilityHandler` routing with Sneak and the refusal rules (dead, severed, paralysed, bloodbent,
meditating), `Cooldowns`, hunger costs, the Element Scroll and `ElementChoicePayload`, the awakening, `/bending` commands,
`AreaTargets` (copied), `BendingRights`, `TempBlocks` with the journal, caps, fluids, block-entity refusal, no drops, the
standing-player rule and crash recovery, `Hazards`/`HazardTicker`, `Grip` (Enhanced's `Telekinesis.hold` adapted for
`RockEntity`/metal targets), `Victims`, `compat/EnhancedHooks`. Tests: `ProgressionTests`, `LoadoutTests`, `TempBlockTests`,
`RightsTests`, `AreaTargetsTests`, `VictimTests`, `CommandTests`, `SaveTests` (a save written and reloaded byte-equal;
a player data record surviving a simulated loader move: the same `SavedData` bytes read on both loaders). No ability
behaviour: that is L2's."

**L2-EARTH:** "Build every Earth ability per `DESIGN.md` 2 and `ANIMATION.md` 3 (with 1, 2, 11, 12, 14 for the shared
rules): server logic in `earth/` (`RockThrow`, `EarthArmour`, `EarthPillar`, `EarthWall`, `SandBending`, `Quicksand`,
`LavaBending`, `MetalBending` with knives/pull/wall/armour/wreck/grip, the Rooted passive, the lava passive), the
entities and blocks listed in `TECH.md` 4 for Earth, `EarthFx` senders with the payload kinds 1–99, the client
`EarthClient` (stances `EARTH_IDLE/GRIP/ARMOURED/RIDE_UP/LAVA_HOLD/PULLING/METAL_HOLD`, every gesture of A 3 with the exact
keyframes, first-person offsets, prediction, cooldown routing), `EarthFx` client (every effect row of A 3 with its
counts, the armour `LayerReveal` slices, overlays), the renderers, the wrecked blocks' textures (L 1.7). Use L1's
`TempBlocks`/`Grip`/`Hazards` through the S0 interfaces (the neutral stubs make your tests pass only after L1 merges: the
integrator runs them together; until then test against a local fake in your batch). Tests: `EarthTests`, `EarthBlockTests`,
`EarthMetalTests`, `EarthAnimationTests` (A 14.2), one per ability per `TECH.md` 14.1's Earth rows, in batches `earth`,
`earth_blocks`, `earth_metal`, `earth_pvp`, `earth_anim`."

**L3-ANIM:** "Extend the copied animation system per `ANIMATION.md` 14.1–14.4 and 11–12: leg-z keys in `Gestures`
(`legs(rx, rz, lx, lz)`, a `legZ` track), groups `RIDE`/`HIT`/`AUTO` and 4 slots with the slot rules, channel masks
(`legsMaskedByAir`, `legsMaskedByWalk`) and `bodyOwner`/`rideOverride`, the first-person camera drop and eye-height offsets
(a `CameraMixin.setup` tail, reduced-motion gated), `VictimPoses` with the three mixins and the `bending:victim` reader
(stagger, knockdown, stuck, lifted, tumble, soaked, chilled, the puppet and paralysed poses as data so 1.2/1.3 only
switch them on), the shared stances `LAUNCHED`/`LIFTED`, the run characters of A 7.1, `LAND_SOFT` variants, the four
awakening gestures and `LOOK_CHANGE`, `InputLocks` (a `Live`-bit set), `/bending_anim` and the overlay, the pose library
constants of A 1.2 as `StancePoses`. Tests: `AnimationMathTests` extended (leg-z rules, yaw whole turns, hand-overs, 4
slots, masks), `VictimPoseTests` (angles within limits, TTL), `AnimationSignalTests` regions. Real client: every pose of
A 1.2 frozen and photographed on wide and slim models."

**L4-FX:** "Build the FX foundation per `ANIMATION.md` 0.4, 13, 14.5, 14.12 and `TECH.md` 9: `GrainParticle` and
`DropletParticle` with their options and sprites, `BendingRenderTypes` (the copied energy types + `MATTER_FLAT`/`MATTER_TEXTURED`),
`MatterShapes` (Enhanced's `SandShapes` with the water/ice additions), `BendingWorldFx` (one `RenderLevelStageEvent`
subscriber drawing every element's decorations in four batches from a bounded client ledger fed by the FX payloads and
pruned by block changes), `ChainSim`, the camera-shake service (Enhanced's `ArenaFx.shake`), `EffectDensity` (vanilla
Particles × the client option × the 48/96-block rules), the overlay base class (vignette, gradient, screen-space bolt),
`EntityGlow` wiring, `LightningFx` and `FlameBeamDrawer` **skeletons** (the geometry, tested, used in 1.3). Tests: pure
geometry tests (`FxMathTests`: ring/funnel/ribbon vertex counts, chain sim stability, density arithmetic), `ParticleTests`
(options codecs). Real client: a debug command `/bending_fx <effect>` playing each primitive."

**L5-UI:** "Build every screen and the HUD per `UI.md`: `ElementChoiceScreen`, `BendingScreen` with `SkillTreeTab` (the
graph canvas with pan/zoom, connectors, the detail panel, unlock/buy/cast), `LoadoutTab` (drag and drop, presets, auto),
`LooksTab` (faces, glider dyes, the draft badge), `WaterTab` (hidden until the Water element exists), `BendingHud` (every
line of U 3, the cooldown sweep, the Sneak swap, dimmed icons with reason glyphs, the SP count-up, placement and scale
options), the icon atlas with **every** icon of U 6 (draw them all now), the action-bar messages, RTL layouts with
`HudDraw.visual`, the settings entry. Use `PreviewPuppet` for the 3D preview with the animator's preview API. Tests:
`HudTests`, `ScreenDataTests`, `LoadoutTests` (payload round trip), `TreeTests` (pure rules through L1's interface),
`IconTests` (every ability and state has a cell). Real client: every screen in both languages at 854×480 and 1920×1080."

**L6-LOOKS:** "Draw and build the six Earth looks per `LOOKS.md` 0 and 1 and the shared layer kinds of L 0 (all of them,
for every release): skins with `tools/art/gen_looks.py` (pixel scripts, committed), each layer's `LayerDefinition` and
texture, `Looks`/`Look` registry with per-look `color`/`glow`/`metalColor`/`bloodColor`/`fireStyle`, the skin mixin
(`SuitSkins` pattern), the face icons, the Element Scroll, Water Skin (9 stages) and Air Glider (tinted) item art, the
Earth block textures of L 1.7. Tests: `LooksTests` (64×64 PNG, valid `SkinCheck`, every texture exists, both names,
slim flags). Real client: every look front/back/side, under the armour layers, in the picker."

**L7-SOUND:** "Build the sound pipeline per `AUDIO.md` 1 (commit `tools/sound/sfx.py`, `render.py`, `recipes/`), render
the shared and Earth events of S 2–3 (every recipe as specified, 2–3 takes where marked, mono 44.1 kHz Ogg, peaks ≤ −1
dBFS, loops seamless), the Earth theme and the choice piece of S 7 as stems, wire `PowerLoopSounds` kinds for Earth and
`EntitySoundLoop`, `ElementMusic` (stems, crossfade, `SelectMusicEvent`), replace the placeholders, empty the S0 placeholder
list in `SoundAssetTests`. Tests: `SoundAssetTests` (format, length, not placeholder, subtitles), `MusicTests` (stem
timing). Real client: every sound at 1/8/24 blocks; the loops follow and stop; the music under a fight."

**L8-DOCS:** "Write the player's guide for 1.0 in `docs/wiki/` (EN and `-HE` pages wrapped in `<div dir="rtl">`), the
README in both languages, `docs/DESIGN.md`'s 1.0 section from `docs/plan/DESIGN.md` with the shipped numbers, and
`SUGGESTIONS.md`'s decisions as made by Eilon recorded in `docs/DECISIONS.md`. Merge last; read every lane's `docs/lanes/*.md`."

---

## Release 1.1 — Air (כשפות אוויר)

**Contents:** every Air ability (D 3), the glider item, Spiritual Projection (the spirit entity and camera), the rides
(scooter), Zaheer's path, the six Air looks (Jae and Karthik as drafts until pictures arrive), Air sounds and the Air
theme, the Air wiki. Protocol **"2"**.

**S0-1.1** (seam keeper, 1–2 agent-days; branch `v11-skel` from `main` 1.0.x): the Air payload kinds 100–199 and
`SpiritPayload`/`SpiritMovePayload`/`GliderDyePayload` handlers to stubs, the `AirFxPayload` record, the `spirit`, `air_scooter`,
`tornado`, `air_blade` entity classes' shells become real classes (registered in 1.0, bodies now), `SpiritCamera` seam, the
`rideOverride` hook for the scooter and glider (L3 of 1.0 built the flag), the `Live` bits already exist, lang keys of 1.1,
the Air sound placeholders are registered (1.0 did), config exists; protocol "2" pinned; a test that the Water tab stays
hidden.

| Lane | Owns | Agent-days |
|---|---|---|
| **L1-AIR-A** | Air Blast, Air Shield, Air Blades, Tornado, Air Wheel: server (`air/`), entities (`AirBladeEntity`, `TornadoEntity`), `AirClient` gestures/stances/first person for them, their effects (`AirFx`), the shield's `ShieldStyle` and renderer wiring | 5–6 |
| **L2-AIR-B** | Air Scooter (vehicle + ride stance + renderer), Air Bubble (+ overlay), Air Glider (item, `FALL_FLYING` physics, dye, `GliderLayer`, the gliding profile and pose, Air Blast downward), Spiritual Projection (`SpiritEntity`, `SpiritCamera`, the possess/move payloads, the meditation stance and eye offset, `SpiritOverlay`, through-wall glows and cave outlines, Deep Meditation), Zaheer's path (the pet-death gate, True Flight profile, Suffocation hold + victim clutch, Air Vacuum, the push/damage factors, the unlock moment) | 7–8 |
| **L3-AIR-LOOKS** | the six Air looks (L 2), `eyes_closed` layer (A 14.9), `air_lattice`/`air_swirl` textures, glider art for 16 dyes, Jae's board (suggestion, behind the Jae look), the draft badge | 3–4 |
| **L4-AIR-SOUND** | S 4's 50 events, the Air theme stems, loops | 3–4 |
| **L5-AIR-DOCS** | wiki Air pages EN/HE, README rows, DESIGN 1.1 | 1–2 |

**Merge order:** S0 → L1-AIR-A → L2-AIR-B → L3 → L4 → L5; joins: the shield-burst vs tornado interaction, the scooter's
`rideOverride`, the glider's `BackLayer` vs robes. **Gate:** full suite both loaders (≈ 650), real-client check, a
4-bender fight with two airbenders (tornado + scooter budgets), the cross-element rows of `MatchupTests` for Air vs Earth.
**Calendar ~2.5 weeks.**

**Briefs:** the 1.0 preamble with `1.1`/`v11-*`; then, **L1-AIR-A:** "Build Air Blast, Air Shield, Air Blades, Tornado
and Air Wheel per `DESIGN.md` 3.1, 3.2, 3.4, 3.8 and `ANIMATION.md` 4.0–4.2, 4.4, 4.8 (gestures with exact keyframes,
first person, effects with counts, sounds by event name), the `TornadoEntity` (orbiting riders, drift, 4-tick position
payloads), `AirBladeEntity`, the Air Shield on Enhanced's shield pipeline (`ShieldStyle` + sphere passes + ripples +
burst), `AirFx` kinds 100–149. Tests per `TECH.md` 14.1 rows Air Blast/Shield/Blades/Tornado in batches `air`, `air_pvp`,
`air_anim`." **L2-AIR-B:** "Build Air Scooter, Air Bubble, Air Glider, Spiritual Projection and Zaheer's Path per
`DESIGN.md` 3.3, 3.5–3.7, 3.9 and `ANIMATION.md` 4.3, 4.5–4.7, 4.9, 10.1, 10.2, 10.5, 10.6, 11.3: the scooter as a vehicle
entity with the ride stance through `rideOverride`, the glider's elytra-based glide with the `GLIDER` flight profile and
the glider layer, the bubble and its overlay, the spirit entity possessed by the client's camera with
`SpiritMovePayload` validated server-side, the meditation stance with the eye-height offset, the spirit overlay and glows,
the pet-death gate (`LivingDeathEvent` of an `OwnableEntity` whose owner is an airbender; the option for proximity), True
Flight through the flight attribute, Suffocation as a `Hazards` hold with the victim's `CLUTCH` pose and air-supply drain,
Air Vacuum, the push/damage factors in `AirAbilities` shared math, the `TETHERS_CUT` moment. `AirFx` kinds 150–199.
Tests per the Glider/Projection/Bubble/Scooter/Zaheer rows, batches `air_rides`, `air_spirit`, `air_zaheer`."

---

## Release 1.2 — Water + Ice + Blood (כשפות מים, קרח ודם)

**Contents:** the water source system and the Water Skin, every Water, Ice and Blood ability, the Water tab and gauge,
the full-moon indicator and moon bells, the Water looks (Katara default), Water sounds and theme, `ChainSim` ropes in
use, the puppet poses. Protocol **"3"**.

**S0-1.2** (1–2 agent-days): payload kinds 200–299, the Water entity shells to classes, `WaterSkinItem` component logic
seam, `SourceFinder` seam filled by L1, the Water tab unhidden behind the element, moon helpers in `McApi`, protocol "3".

| Lane | Owns | Agent-days |
|---|---|---|
| **L1-WATER-A** | `WaterSource`/`SourceFinder` (open water, rain, the Skin, plants with drying), `WaterSkinItem`, Water Shot + Whip, Healing + Cleanse, Water Arms + Grab/Climb/Swing, the water passives and the full-moon factor, `WaterClient` for these, `WaterFx` kinds 200–239, the Water tab and gauge wiring (U 2.2d, 3.3), the moon indicator (U 3.6), `WaterArmsLayer` | 6–7 |
| **L2-WATER-B** | Water Octopus (layer + `ChainSim` tentacles + slaps/strikes/lash/grab), Water Spout (vehicle + ride + ram + freeze + Wave Ride), Ice (knives, spikes, freeze disc/feet/constructs, snow, wall, dome: temp ice blocks, `IceSpikeBlock`, the chilled/wet effects), `WaterFx` kinds 240–279 | 7–8 |
| **L3-BLOOD** | Blood Bending (the moon rule, ES rule, the hold with puppet movement by look, fling, the two-target M10), Bending Severance (the touch, the duration formula, the severed state across logout, the healable option), the `PUPPET`/`SEVERED` victim poses switched on (data from 1.0's L3), `BloodOverlay`, the victim overlays and input locks, `WaterFx` kinds 280–299, the blood sounds' wiring | 4–5 |
| **L4-WATER-LOOKS** | the four Water looks (L 3) incl. Ming-Hua's armless skin and Amon's mask/hood, the Water Skin layer, `water_foam`/`ice_frost`/`wet` textures | 3 |
| **L5-WATER-SOUND** | S 5's 60 events, the Water theme + moon bells stem | 3–4 |
| **L6-WATER-DOCS** | wiki Water/Ice/Blood pages EN/HE, DESIGN 1.2 | 1–2 |

**Merge order:** S0 → L1-WATER-A → L2-WATER-B → L3-BLOOD → L4 → L5 → L6; joins: the octopus and arms sharing `ChainSim`
parameters, Freeze acting on L2's constructs and L1's arms, the matchup rows Water vs Fire **prepared** (fire lands in
1.3: the factors are tested against a scripted "fire damage" source), the Water tab in the screen. **Gate:** full suite
(≈ 900), real client incl. a full-moon night and the Nether (no moon), a 4-bender fight with two waterbenders (octopus +
spout budgets). **Calendar ~3 weeks.**

**Briefs:** preamble `1.2`/`v12-*`; **L1-WATER-A:** "Build the water source system and Water Shot/Whip, Healing/Cleanse,
Water Arms per `DESIGN.md` 4.0–4.2, 4.5 and `ANIMATION.md` 5.0–5.2, 5.5 (the 3-tick source draw chained before every
gesture: use the `param` delay or implement A 14.6 chaining in `BenderAnimator`), the Water Skin item with its component,
refills and the HUD gauge (`UI.md` 3.3), the plant-drying temp states, the full-moon factor and indicator (`UI.md` 3.6),
`WaterArmsLayer` with the `RenderArmEvent` first-person override. Tests per the Water Shot/Healing/Water Arms rows and
`SourceTests` (each source in order, the Nether, rain, plants at M7, claims), batches `water`, `water_source`, `water_anim`."
**L2-WATER-B:** "Build Water Octopus, Water Spout and the three Ice cards per `DESIGN.md` 4.3, 4.4, 4.6 and
`ANIMATION.md` 5.3, 5.4, 5.6, 10.3: the octopus veil + 8 `ChainSim` tentacles with slap/strike/lash/grab, the spout vehicle
with the ride stance, ram, M8 freeze and Wave Ride, the ice temp blocks (`IceSpikeBlock` rise states, frosted-look ice
discs, walls, domes), knives/snow projectiles, Freeze on water/feet/constructs (the construct hook is an interface L1's
arms and your octopus implement), the `chilled`/`wet` effects. Tests per the Octopus/Spout/Ice rows, batches `water_shapes`,
`water_ice`, `water_rides`." **L3-BLOOD:** "Build Blood Bending and Bending Severance per `DESIGN.md` 4.7 and
`ANIMATION.md` 5.7, 12.4, 12.5, 2.2: the moon and ES gates, the hold as a `Hazards` object moving the victim by the
caster's look with speed caps, the victim's `PUPPET` pose and input lock, fling, two targets at M10, the severance touch
and formula, the severed state in `BenderSave` and the `severed` effect refusing every ability and passive, the HUD greying,
the overlays and heartbeat loops, `blood_refused` on a stronger waterbender. Tests per the Blood/Severance rows, batches
`blood`, `blood_moon` (sets the moon), `blood_pvp`."

---

## Release 1.3 — Fire + Lightning (כשפות אש וברק)

**Contents:** every Fire ability, lightning generation and redirection, the fire passives, Azula's blue fire, the five
Fire looks (Zuko default), Fire sounds and theme, the flame beam drawer and lightning FX in use, the paralysis pose, all
matchups live. Protocol **"4"**.

**S0-1.3** (1–2 agent-days): payload kinds 300–399, `FireballEntity` to a class, the `FLAME`/`BLUE_FLAME` beam styles
registered, the `paralysed` effect's client lock seam, `HandTracker.belly` (A 14.8), protocol "4".

| Lane | Owns | Agent-days |
|---|---|---|
| **L1-FIRE-A** | Fireball (+ small/great), Fire Punch (+ kick, combo, jab), Fire Whip (+ lash, wrap, cutting rocks/blades), Fire Circle (+ legs, jet flight profile), the fire passives (resistance, self-extinguish, sun), fires-where-allowed through `BendingRights.mayHurtBlock`, `FireClient` for these, `FireFx` kinds 300–349, the per-look fire palette switch (Azula) in every fire effect | 6–7 |
| **L2-FIRE-B** | Flamethrower + Fire Breath (two `ContinuousBeam`s per player, the `FlameBeamDrawer`, the breath lock and `HeatOverlay`), Fire Healing + Burn Out, Lightning Generation (charge hold, release, dual at M10, arc, `LightningFx`, thunder, the `paralysed` effect with `PARALYSED` pose and overlay, rods attracting, water spreading, no vanilla bolt), Lightning Redirection (the stance, the window, chance by mastery with seeded RNG, pass/fail/ground, the body path on `HandTracker.belly`, `RedirectPathOverlay`, vanilla lightning redirect, Enhanced's lightning if present via `is_lightning`), `FireFx` kinds 350–399 | 7–8 |
| **L3-FIRE-LOOKS** | the five Fire looks (L 4) incl. the crowns and `white_lotus_robe` tints, `fire_mouth`, `heat`, `scorch` textures | 3 |
| **L4-FIRE-SOUND** | S 6's 55 events incl. the blue-fire pitch variants, the Fire theme + blue stem | 3–4 |
| **L5-MATCHUPS** | every row of D 1.7 switched on and tested across all four elements (`MatchupTests`, `CrossElementTests`, `CoexistenceTests` with the 4-bender script of A 13), balance pass on the numbers with Eilon's feedback from 1.0–1.2 play (config defaults only) | 2–3 |
| **L6-FIRE-DOCS** | wiki Fire/Lightning pages EN/HE, DESIGN 1.3, the matchups page | 1–2 |

**Merge order:** S0 → L1-FIRE-A → L2-FIRE-B → L3 → L4 → L5 → L6; joins: water vs the beams and whip, lightning vs steel
and water constructs, the blue palette in L2's beams, redirection of L2's own bolts in PvP. **Gate:** full suite (≈ 1,200),
real client with Azula's blue fire in every effect, a thunderstorm, the Nether, a 4-bender fight of all four elements
(the A 13 budget measured), both loaders and cross-loader, `1.3.0`. **Calendar ~3 weeks.**

**Briefs:** preamble `1.3`/`v13-*`; **L1-FIRE-A:** "Build Fireball, Fire Punch, Fire Whip, Fire Circle and the fire
passives per `DESIGN.md` 5.0–5.2, 5.4, 5.5 and `ANIMATION.md` 6.0–6.2, 6.4, 6.5, 10.7 (the Fire Kick and leg sweep use
the leg-z keys; the Fire Circle spins with yaw; the Fire Jet is a flight profile), `FireballEntity`, the fire whip on
`ChainSim` with upward gravity, fires only through `BendingRights.mayHurtBlock` with the caps, the Azula palette switch
(`Looks.fireStyle(data)` read by every effect and sound pitch). Tests per the Fireball/Punch/Whip/Circle rows, batches
`fire`, `fire_blocks` (fires), `fire_pvp`, `fire_anim`." **L2-FIRE-B:** "Build Flamethrower/Fire Breath, Fire Healing,
Lightning Generation and Lightning Redirection per `DESIGN.md` 5.3, 5.6–5.8 and `ANIMATION.md` 6.3, 6.6–6.8, 12.5, 2.2:
two beams per player on the copied `ContinuousBeam`/`BeamRenderer` with the `FlameBeamDrawer`, the breath lock, the
lightning charge as a held gesture with input lock and a server-side raycast bolt (the mod's damage type in `is_lightning`,
no vanilla `LightningBolt`), `LightningFx` with branches and flashes, delayed `thunder_distant`, the `paralysed` effect
with its pose/overlay/lock, redirection as a `Hazards` window with mastery chance (seeded by game time + player for
tests), the body-path effect and screen overlay, Ground It. Tests per the Flamethrower/Fire Healing/Lightning/
Redirection rows, batches `fire_beams`, `lightning`, `lightning_pvp`."

---

## After 1.3: 1.4+ (suggestions, if Eilon says yes)

Each extra of D 8 is one lane in its own small release: 1.4 **Temples and Masters** (structures, the tutorial NPCs,
bending scrolls: 3 lanes), 1.5 **Bosses** (the four spirit bosses on the copied boss framework: 4 lanes + a framework
copy lane), 1.6 **Companions** (sky bison, lemur, polar bear dog, fire ferret: 2 lanes), 1.7 **Duels and dummies**, 1.8
**Later Minecraft versions** (the `r2` adapter and the matrix), 2.0 **The Avatar**.

## Roles

* **Seam keeper** (one agent per release, the S0 author): keeps `v1x-skel` green, merges `request(I)` branches within the
  day, pins seams with tests, answers "who owns this file" questions from `docs/LANES.md`.
* **Integrator** (one agent, may be the seam keeper after S0): merges in order, resolves conflicts by "keep both sides"
  (everything is additive) then `./gradlew build`, does the joins, runs the lanes' batches together then the full suite on
  both loaders twice, does or assigns the real-client checklist, writes the "how 1.x was put together" section, bumps
  `mod_version`, tags.
* **Lanes**: as briefed. A lane that needs another lane's code before it merges codes against the S0 interface and, if
  it must, reaches the other lane's accessors **by reflection until the merge** (Enhanced's L7 of 3.8 did this); the
  integrator replaces reflection with direct calls in the joins.

## Risks and how the plan holds them

| Risk | Hold |
|---|---|
| The Fabric half and the universal jar are new ground (no Enhanced precedent) | L0-FABRIC starts first with the biggest budget; 1.0-neoforge is the fallback milestone; every later release's gate runs both loaders |
| Saves across loaders | `SavedData` for everything persistent (T 2.4); `SaveTests` prove byte-equality; the real-client checklist moves a world |
| Enhanced's platform layer changes under us (its P1) | Bending copies at a pinned Enhanced commit (888609c) and records it in `docs/LANES.md`; a later re-sync is a lane |
| Temp blocks and claims on servers | `BendingRights` + the loader's events + the journal + caps; `RightsTests`/`TempBlockTests`; the Fabric limitation documented |
| Performance of four benders | per-tick caps in every effect row, measured at every gate (A 13) |
| Seven Havens looks without references | drafts behind a badge; Eilon's pictures turn them final in a one-day art task |
| Balance numbers wrong for an 11-year-old's fun | every number in config; L5-MATCHUPS of 1.3 and a balance pass after each release from Eilon's play |
| Animation and effect quality (the heart of the request) | the bible's keyframes are the tests' expected numbers; the real-client check per lane; `/bending_anim` for frame-by-frame review with Eilon |
