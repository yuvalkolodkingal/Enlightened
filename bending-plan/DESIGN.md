# Bending — Full Game Design

**Bending** (mod id `bending`) is a Minecraft mod in the world of *Avatar: The Last Airbender*, *The Legend of Korra*
and *Avatar: Seven Havens*. A player bends one element — Earth, Air, Water or Fire — earns skill points by beating
monsters with their bending, unlocks abilities on a skill tree, masters each ability by using it, picks a combat
loadout, and wears the look of a bender from the shows. It is built on the foundations of **Enhanced**
(`yuvalkolodkingal/Enhanced`): the same platform layer, the same hero plug-in shape, the same gesture and FX systems,
the same HUD, config and test kit, adapted for elements instead of heroes (see `TECH.md` for exactly what is copied).

Original design: **Eilon Kolodkin-Gal**. Everything Eilon asked for is kept. Where his design leaves a gap (a number,
a key, a Sneak variant, an edge case), it is filled in here and marked **(added)**; where something is only an idea
for him to decide, it is marked **(suggestion)**. The Hebrew summary of every decision he has to make is `SUGGESTIONS.md`.

Conventions in this document:
* **1 heart = 2 damage.** Every damage number is in hearts.
* Ticks: 20 ticks = 1 second. Cooldowns are given in seconds (the config stores ticks).
* **Mastery** is 1 to 10. "M" in a formula is the ability's mastery level. "Element strength" (ES) is defined in 1.4.
* Every number below is a **server config default** (`world/serverconfig/bending-server.toml`, sections named in
  `TECH.md`). Nothing is hard-coded.
* Keys are the **defaults**; everything is rebindable under *Options → Controls → Bending*.

---

## 0. Rules for every bender

| Rule | Detail |
|---|---|
| One element per player | A player bends exactly one of Earth, Air, Water or Fire, or none. The Avatar (all four) is **not** in the design (see 8.1, a later idea). |
| Permanent | The element, the skill points, the unlocked abilities and every mastery level survive death, logout and dimension change (saved on the player, kept on death, `TECH.md` 4). Only an operator command removes them (`/bending element set <player> none`), or the optional change rule of 1.6. |
| Combat keys **(added)** | Four **loadout slots**, on **X, C, Left Alt, K** by default (1.5), each with a plain and a **Sneak** use: up to 8 abilities ready at once. Loadout, skill tree and looks live on one screen, **O**. |
| Sneak variants | Every ability has a Sneak variant (its own name, its own numbers, the same cooldown slot unless stated). The server reads the Sneak flag the client sent with the key, as Enhanced does (`AbilityKeyPayload.sneak`). |
| Hold, press, toggle | An ability is *press* (once per press, then its cooldown), *hold* (while the key is down; stops on key up, death, screen, hunger 0, Spectator), or *on/off*. Every hold ability is released by `stopHold` when the player dies or loses the element. |
| Hunger | Bending costs exhaustion, as Enhanced's powers do: a press costs `pressExhaustion` 0.6 food-exhaustion, a hold `holdExhaustionPerSecond` 0.8. At 0 hunger nothing but Rock Throw, Air Blast, Water Shot and Fireball work (the "free" starters). |
| Creative and Spectator | Creative benders have no cooldowns and no costs, but every protection and rights rule still holds. Spectators bend nothing. |
| Who an area attack may hurt | One rule for every wave, ring, burst and tornado: Enhanced's `AreaTargets.mayHurt` copied as `bending.ability.AreaTargets` — never the caster, their pets, their team, Creative or Spectator players, armour stands; other players only with PvP on; villagers, golems, animals and others' pets only with the element's `hurtsPassives` option (**false** by default). |
| What a bending power may change in the world | One rule book, `BendingRights` (Enhanced's `BlockRights` adapted, `TECH.md` 6): spawn protection, the world border, Adventure mode, claim mods (through the loader's block events), Enhanced's own walls if present, and the `bending:unbendable` tag. Every moved, raised or melted block is **temporary** and reverts (1.8). |
| Looks | Each element has its character looks (`LOOKS.md`), each a full skin drawn for the mod plus `ModelPart` layers (hair, robes, plates). A look changes only how you look and the colour of your element (Azula: blue fire). Every element also has *Your own skin*. |
| Languages | Everything is in English and Hebrew, the Hebrew in the dub's words: כשף אדמה / כשפות אדמה, כשף אוויר / כשפות אוויר, כשף מים / כשפות מים, כשף אש / כשפות אש, כשפות דם, כשפות ברק. |

---

## 1. Core systems

### 1.1 Becoming a bender

**Default (Eilon's open question 1): the Element Scroll.**

* **Element Scroll** (item `bending:element_scroll`): crafted from 4 paper, 1 ink sac, 1 stick and **one item of each
  element** in the corners: a block of stone (Earth), a feather (Air), a water bucket (Water, returned) and a blaze
  powder (Fire). Cheap on purpose: Eilon wants everyone to be a bender early. **(added)**
* Right-click with no element: the **Element Choice screen** opens (`UI.md` 2.1): four cards with the element's symbol,
  colours, three sample abilities and the looks; a 3D preview of you in the element's first look. Choosing plays the
  **awakening**: a title ("You are an Earthbender" / "אתם כשפי אדמה"), the element's awakening sound, a burst of its
  particles, and a chat line to the server ("Eilon has become an Earthbender!"). The scroll is used up.
* With an element already: the scroll says so in the action bar and does nothing (the change rule is 1.6).
* Operators: `/bending element set <players> <earth|air|water|fire|none>`; `/bending points give <players> <n>`;
  `/bending mastery set <player> <ability> <1..10>`; `/bending unlock <player> <ability|all>`; `/bending look <players> <look>`.
* **(suggestion)** Alternative Eilon may prefer: a **spirit at first join** (a small glowing fox-spirit that appears
  once near a new player and offers the same screen). Not built unless chosen; the scroll stays available either way.

### 1.2 Skill points

**Skill points (SP)** are earned by defeating monsters **with your bending** and spent on the skill tree.

* A kill counts when the killing blow is a bending damage type (tag `bending:bending_damage`) **or** the monster was
  hit by your bending within the last **5 seconds** (`assistWindowTicks` 100) — so a rock that softens a zombie and a
  sword that finishes it still counts. A kill with no bending in it gives nothing.
* Points per monster (`[progression]`, every row configurable; a mob not listed uses the formula):

| Monster | SP | Notes |
|---|---|---|
| Zombie, Skeleton, Spider, Creeper, Drowned, Husk, Stray, Cave Spider, Silverfish, Slime, Zombified Piglin, Pillager, Phantom | 1 | |
| Enderman, Blaze, Witch, Piglin Brute, Guardian, Vindicator, Magma Cube, Hoglin, Zoglin, Bogged, Breeze | 2 | |
| Evoker, Ravager, Shulker, Illusioner, Ghast | 4 | |
| Elder Guardian | 10 | |
| Warden | 25 | |
| Wither | 40 | |
| Ender Dragon | 60 | once per dragon |
| Players | 0 | never: no PvP farming |
| Passive animals, villagers, golems | 0 | |
| Any other hostile mob (other mods) | `round(maxHealth / 20)`, at least 1, at most 50 | bosses of other mods by their health |
| **(suggestion)** The element bosses of 8.5 | 30 each | |

* The **first 3 SP of each element's tree are free** (given with the element), so a new bender is never stuck with one
  ability.
* The HUD shows your SP; the skill tree shows every ability's cost.
* No cap on SP. A full tree (every ability unlocked, every mastery bought to 10) costs about 160 SP per element
  (table in 7.3), which is roughly a Wither and two hundred zombies.

### 1.3 Mastery

Every unlocked ability has a **mastery level 1 to 10**. It rises in two ways:

1. **Use.** Each *successful* use (a hit, a wall that stopped something, a heal that healed, a ride that moved you)
   gives **1 mastery XP**; an unsuccessful one (a rock thrown into the void) gives 0, so spamming into the air teaches
   nothing. Levels need cumulative XP: 2: 10, 3: 25, 4: 50, 5: 90, 6: 150, 7: 230, 8: 340, 9: 480, 10: 660
   (`masteryXp` list). A training dummy (8.2) gives XP at half rate.
2. **Points.** On the skill tree, one level of mastery costs **SP = the level you are buying, halved and rounded up**
   (level 2: 1 SP ... level 10: 5 SP), bought one level at a time. Buying is capped at **level 7**: levels 8 to 10
   come only from use. ("The more skilled you are, the more earth you lift": the last levels are earned.)

What mastery scales is listed per ability (section 2 to 5); in general: **size** (rocks, walls, tornadoes, whips),
**count** (rocks and knives at once), **range**, **duration**, and a little **damage** (never more than +50% at 10),
and some abilities gain a **new form** at a threshold (fire breath at 7, dual-hand lightning at 10, the small fireball
option at 6).

### 1.4 Comparing benders: element strength (Eilon's "one rule")

**Element strength (ES)** of a player = **the sum of the mastery levels of every unlocked ability of their element**
(an unlocked ability counts at least 1). A fresh earthbender has ES 1; a full tree has ES 100. The HUD shows it next
to the SP.

Used by: **Quicksand** (fails on an earthbender whose ES ≥ yours), **Blood Bending** (fails on a waterbender whose ES ≥
yours), **Bending Severance** (duration by the target's ES, 4.5), and the **(suggestion)** duel ranks of 8.6. A tie
always favours the defender. ES is also what the full-moon-free rule of blood bending reads (4.5: "fully mastered" =
Blood Bending at mastery 10).

### 1.5 Combat loadout and keys

* **Four slots**, default keys **X**, **C**, **Left Alt**, **K**; each slot holds one ability, whose Sneak variant
  comes with it on **Sneak + key**. Eight abilities ready at once; the rest wait on the skill tree and are swapped in
  on the **Loadout tab** (`UI.md` 2.3), also by dragging in the inventory-style grid. Swapping has no cost and no
  cooldown, but a swapped-out ability keeps its running cooldown.
* **Why these keys (added).** Enhanced owns **R, G, V, B, N, H, Y, Z** (its five ability keys, Change Look, Build
  Magic, the Vision order) and **J** (Stone Wheel / Web Assist); vanilla owns W A S D, E, Q, F, T, L, P, Tab, 1–9,
  Shift, Ctrl, Space. Left of the keyboard that leaves **X** and **C** (vanilla's Creative toolbar save/load, which
  almost nobody uses; Enhanced's Build Magic uses them only with **Ctrl** held and only in build mode, so there is no
  clash), **Left Alt** (Enhanced's "second knob" uses it only inside build mode), and no fourth: **K** is the nearest
  free letter. Eilon chooses (SUGGESTIONS 3); alternatives are **I** for slot 4, or three slots only.
* **O** opens the **Bending screen** (tabs: Skill Tree, Loadout, Looks, and the Water Skin's level for waterbenders).
  **Sneak + O** cycles the look at once (like Enhanced's H). No other keys: the glider, the water skin and the scroll
  are items used with right-click.
* Pressing a slot key with nothing in it says "Empty slot: press O" once in the action bar.
* A held slot key shows the element's **ready stance** (`ANIMATION.md` 9) for as long as it is held with a *press*
  ability that is on cooldown; this is how Eilon's "idle stance while a bending key is held" is reached without a
  separate key.

### 1.6 Changing element (Eilon's open question 7)

**Default: no.** The element is for keeps, like an Enhanced power; an operator can reset it. **(suggestion)** If Eilon
wants a way: **Spirit Water** (a rare item from the Spirit Oasis structure of 8.4, or 1 per world from the Water
boss), used on yourself, removes the element and **every point and level with it**, so changing is a real restart; a
server option `allowElementChange` (false) turns the item on. The scroll then works again.

### 1.7 Element matchups

Eilon's two rules, then the **(added)** rest, every factor in config `[matchups]`:

| Attacker's element → defender / effect | Rule |
|---|---|
| **Water → Fire** | Any water ability that hits a **firebender** deals **×1.5**; water that touches a fire ability's effect **puts it out** (a Fire Whip or Flamethrower stream crossed by a Water Shot ends early; burning targets stop burning; a fireball hit by a water shot bursts harmlessly). Water Shot and Water Whip never miss a Fire Circle: they cut a gap in it. |
| **Fire → Ice** | Any fire ability deals **×1.5** to a target that is frozen (Freeze), and **melts** ice walls, ice knives in flight and frozen water on contact (an Ice Wall takes 1 fireball per 4 blocks). Water (the liquid abilities) is *not* weak to fire. |
| **Earth → Air** (added) | A rock in flight passes through an Air Blast and an Air Shield absorbs rocks only up to its strength (2.3); Earth Wall and Steel Wall stop Air Blades and Tornadoes completely. |
| **Air → Fire** (added) | An Air Blast or Tornado pushes a Fireball off course (it keeps flying in the new direction) and snuffs a Fire Circle within its radius; a firebender standing in a Tornado cannot start a hold ability. |
| **Lightning → Water** (added) | Lightning deals **×1.5** to a target that is in water, wet (rain, a Water Shot in the last 3 s) or a waterbender with an octopus or arms up; a Water Spout that lightning strikes collapses. |
| **Lightning → Metal** (added) | Lightning deals **×1.5** to Steel Armour and Iron Golems; **Earth Armour** (stone) takes **×0.5**. |
| **Earth ↔ Earth** | Rocks shatter on Earth Wall; a Rock Throw at an Earth Armour wearer deals ×0.5. |
| Fire → Fire | Firebenders take ×0.5 from fire (the passive 5.0). |

### 1.8 Temporary blocks, lag and griefing (every element)

Block-moving abilities never make permanent changes to the world:

* **Every block an ability raises, moves, melts or freezes is a *temporary block* (`TempBlocks`, `TECH.md` 6):** the
  original state is recorded, the new one placed, and a revert is scheduled. On revert the original comes back unless
  a player stands inside it (then the revert waits up to 10 s and finally pushes the player up). Containers, block
  entities, bedrock, portals, spawners, reinforced deepslate and the tag `bending:unbendable` are never touched; a
  wall grows around them.
* **Rights.** Before any cell changes, `BendingRights.mayChange(player, level, pos)` asks: spawn protection, the world
  border, Adventure/Spectator, the loader's block-place and block-break events (claim mods answer there), Enhanced's
  Hex and boss arenas if Enhanced is loaded (optional hook), and the dimension rules of 1.9. A refused cell is skipped;
  if more than half of a wall's cells are refused the wall is refused (the action bar says "Protected ground").
* **Budgets (per player, `[limits]`):** at most **1 wall**, **1 pillar**, **1 lava path**, **3 ice walls**, **40
  temporary blocks** standing at once (a bigger wall replaces the older one); server-wide **2,000** temporary blocks (a
  new one beyond that reverts the oldest). Projectile entities: at most **6 rocks**, **8 knives**, **2 tornadoes**,
  **1 spout**, **1 scooter** per player. A fight of **4 benders** therefore never exceeds ~160 temp blocks and ~60
  entities; every entity is `MobCategory.MISC`, `noSave()` where it may be, and discarded on logout.
* **Particles** are client-side with budgets per ability (`ANIMATION.md`, each ability's Effects row; the client option
  *Effect density* 0.25–1.0 scales them, and nothing is spawned beyond 48 blocks).
* **Crash safety.** Temporary blocks are journaled in the level's `bending:temp_blocks` attachment; on server start,
  every entry still standing is reverted (the lesson of Enhanced's `ArenaLoans`).
* **Griefing.** Because nothing is permanent, the only lasting change bending can make is damage to entities and the
  fires lit by fire abilities (**only** with gamerule `mobGriefing` on and `[fire] setsFires` true, on blocks the
  rights allow, at most 12 fire blocks per ability use, never in a claim).

### 1.9 Dimensions, water, lava and other mods' blocks

| Where | What changes |
|---|---|
| **Water** (the bender is in it) | Earth: no rocks (no earth under you unless you stand on the floor); Earth Armour crumbles when fully submerged; Earth Pillar works from the floor. Air: Air Bubble is the point; Air Blast works at half range; Scooter becomes a water-skimming ride at the surface. Water: everything works and the water supply is infinite. Fire: Fireball, Flamethrower and Fire Whip **do not work under water**; Fire Punch works at ×0.5; Lightning works and hurts the bender too (1 heart) unless they are an Azula/Ozai look... no: hurts everyone equally (added for fairness). |
| **Lava** | Earth: Lava Bending draws from real lava too (a Lava Path next to lava is free of cost); the lava resistance passive (2.7) applies. Water: a Water Shot into lava makes obsidian/cobble as vanilla does, temporary; healing works. Fire: full lava immunity only for Ozai at mastery 10 (**no**: no look changes rules; drop) — fire passive applies to lava damage ×0.5. Air: nothing special. |
| **The Nether** | Water: **no water source** except the Water Skin (it evaporates ×2 faster: 1 charge per 20 s) and the Nether's plants (warped fungus, nether wart, at high mastery); blood bending's full moon is **never** (no moon): only at mastery 10. Fire: all fire abilities deal **×1.25** (`netherFireBonus`). Earth: netherrack, soul sand, basalt, blackstone and magma are earth; soul sand is "sand" for sand bending; Lava Path is free beside lava. Air: nothing special; Spiritual Projection cannot cross dimensions. |
| **The End** | Earth: end stone is earth (no sand, no metal unless carried). Air: the void is the void: Scooter and Glider do not save you below y=0; Spiritual Projection may not fly below y=0. Water: the Skin only; the moon rule reads the Overworld's moon phase through the server clock (a full moon in the Overworld counts). Lightning: ×1.0. |
| **Rain and thunderstorms** | Waterbenders: rain is a water source (any ability, half cost). Lightning: ×1.25 in a thunderstorm; Lightning Redirection may redirect vanilla lightning. Fire: Fireball and Flamethrower deal ×0.8 in rain. |
| **Other mods' blocks** | A block is earth if it is in `bending:earth` (the mod's tag: stone, dirt, gravel, deepslate, clay, terracotta, mud, netherrack, basalt, blackstone, end stone, tuff, calcite, dripstone, cobblestone, all `minecraft:base_stone_*`, `c:stones`, `c:gravels`, `c:dirt`...), sand if in `bending:sand` (sand, red sand, soul sand, soul soil, suspicious sand, `c:sands`), metal if in `bending:metal` (iron/gold/copper/netherite blocks, anvils, chains, iron bars, rails, hoppers, cauldrons, lanterns, ores of iron/gold/copper, raw blocks, `c:ores`, `c:storage_blocks/*metal*`), a plant if in `bending:plants`. Other mods add to the tags with data packs; anything not tagged is not bent. |
| **Enhanced present** (optional) | Bending checks Enhanced's `Shelters`/`BossFights` through a soft hook if the class is present (`TECH.md` 9): no bending into a Hex that keeps you out, nothing bent in a boss arena. An Enhanced hero can also be a bender (two mods, two keysets); the design does not forbid it, a server option `allowWithEnhancedPower` (true) can. |

### 1.10 PvP, mobs and bosses in general

* **PvP off**: no bending damage to players, no quicksand, no blood bending, no suffocation on players; pushes
  (Air Blast) still push (they are not damage) unless `pushPlayersWithoutPvp` is false.
* **Bosses** (`minecraft:ender_dragon`, `wither`, `warden`, and the tag `bending:bosses`, which Enhanced's bosses join
  through a data pack): never lifted, never frozen in place, never blood-bent, never quicksanded, never suffocated;
  pushes are ×0.2; paralysis from lightning is 0.5 s; damage is full.
* **Iron golems, Enhanced's heroes in metal, armour-wearing mobs** count as metal for Metal Grip and Lightning ×1.5.
* **Players riding or in vehicles** are dismounted by a successful Air Blast or Water Spout push.

---

## 2. Earth — כשפות אדמה (release 1.0, the first element)

**Theme.** Hung Gar: rooted, low, patient, then explosive. Earthbending is **neutral jing**: wait and listen, then
strike. Colour: ochre and moss green `#7A8F3A` / `#C9B47A`, Toph's green and gold.

**Earth source.** Every Earth ability except Earth Armour's shed and the metal abilities needs **earth within reach**:
a block in `bending:earth` (or `bending:sand` for sand abilities, `bending:metal` for metal) within `earthReach` 6
blocks of the bender's feet or of the aimed point. Standing on a wooden floor over stone is fine (the stone is within
6); flying high is not. The HUD dims Earth icons when no earth is in reach.

**Passive (added, free with the element):** Rooted. While standing on earth and not moving, knockback taken is ×0.5.

### 2.1 Rock Throw — זריקת סלע (free; the starter)

| | |
|---|---|
| Key | Slot ability; **press**. Sneak: **Hold Rock** (אחיזת סלע) |
| Unlock | Free with the element |
| What | A rock tears out of the nearest earth in front of you (the ground under your aim, else under your feet) and flies where you look. It hurts the first thing it hits and shatters into gravel-coloured dust; the hole it left fills back in after `rockHoleRevert` 3 s (temporary block). |
| Damage | **2 hearts** +0.2 per mastery level above 1 (3.8 at 10). Knockback 1.0. A rock that hits a block breaks nothing. |
| Mastery | **Count and size.** M1–3: one 1-block rock. M4–6: **two** rocks (both at once, a half-tick apart, slightly spread). M7–9: **three**. M10: one **boulder** (2×2×2 blocks, 6 hearts, knockback 2.0, 60-tick cooldown) **or** three rocks: Sneak-free choice by tapping twice within 10 ticks (a double tap gives the boulder). Range 24 + M. Speed 1.6 blocks/tick. |
| Cooldown | 1.5 s (30 ticks); boulder 3 s |
| Sneak: Hold Rock | Lift one rock and hold it in the air where you look (like Enhanced's telekinesis: `Telekinesis.hold`), move it with your look, press again to throw it (same damage), Sneak + press to set it down gently (it becomes a block where `BendingRights` allows, else drops as its item... no: a temporary rock never drops; it crumbles back to where it came from). Holding costs 0.1 exhaustion/s. From M5 you hold **two** rocks, from M8 **three**, orbiting you. |
| Counters | An Earth Wall or Steel Wall stops it; an Air Shield absorbs it (eating 1/3 of its strength per rock); Water Octopus arms slap one rock per 2 s; a Fire Whip at M7+ cuts a rock in half (1 heart). Blocked by any shield of Enhanced the same way as a thrown block. |
| Edge cases | No earth in reach: "No earth here" in the action bar, no cooldown (a move with nothing to act on starts no cooldown, so no gesture lies). Under water: no. In a claim: the rock is lifted only from cells you may change; otherwise refused. Mobs: full damage. Bosses: full damage, knockback ×0.2. A rock never breaks glass or crops (it is an entity that stops at a block). |

### 2.2 Earth Armour — שריון אדמה

| | |
|---|---|
| Key | Slot ability; **on/off** (press forms it, press again sheds it). Sneak: **Shed Blast** (ניפוץ השריון) |
| Unlock | 4 SP (needs Rock Throw M2) |
| What | The earth within 3 blocks rises and wraps your whole body, leaving a slit for the eyes (the look: `LOOKS.md` 1.1, the first-person slit overlay: `ANIMATION.md` 2.2). It is a shell with its **own health**; blows hit the shell first. |
| Defence | Shell health **12 hearts** +1.5 per M (25.5 at 10); every hit on you is taken by the shell at ×1 until it breaks; while whole it also gives **+8 armour**, **+4 toughness**, knockback resistance 0.6, fire damage ×0.5 and lightning ×0.5 (1.7). It does not regenerate; shed and re-form to refresh. |
| Downside | Movement speed **×0.7** (×0.8 from M6, ×0.85 at M10); no sprinting at M1–3; you sink in water (and the shell crumbles when your head is under water); no Scooter-like speed. |
| Cost | 8 earth blocks within 3 (they are *not* removed: the armour is drawn from the ground's "spirit", so nothing is bent out of the world; **(added)** keeps claims and lag simple). Forming takes **20 ticks** (you may not move while it forms; crumbling is instant). |
| Duration | Until shed, broken or the player dies. `armourMaxSeconds` 0 = unlimited (a server may set a limit). |
| Cooldown | 20 s after it breaks or is shed |
| Mastery | Shell health (above); speed penalty eases; at M8 the slit's edges glow faintly and **Seismic Sense** (added): while standing still in armour, hostile mobs within 16 glow through walls for you only. At M10 the armour **re-forms itself once** after breaking if earth is in reach (another 20 s cooldown after that). |
| Sneak: Shed Blast | Burst the shell outward: a ring of stone shards to 4 blocks, **1.5 hearts**, knockback 1.5 (`AreaTargets`), and the armour is gone (cooldown as shed). |
| Counters | Lightning ×0.5 (good), Fire ×0.5, **Water Shot ×1.5 to the shell** (added: water erodes stone) and a frozen shell (Freeze) halves its armour until thawed; Metal Grip cannot grab stone. Air Blast pushes a wearer ×0.5 (rooted). |
| Edge cases | Forming while flying (Creative flight): refused. Under water: crumbles. Nether: netherrack counts. Bosses: the Warden's sonic boom ignores the shell (bypasses armour: `bypasses_armor` damage goes to you). PvP off: still protects from mobs. Other players see the full stone body; the wearer's own skin is hidden under it. |

### 2.3 Earth Pillar — עמוד אדמה

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Earth Launch** (זינוק אדמה) |
| Unlock | 2 SP |
| What | The ground under your feet rises as a 1×1 column of the earth you stood on (temporary), lifting you with it. Press again while on top to lower it. |
| Height | **3 blocks** +0.5 per M (up to 7.5 → 8 at M10); rise speed 0.5 block/tick (the player is carried smoothly, `ANIMATION.md` 2.3). |
| Duration | The pillar stands **15 s** (`pillarSeconds`), then sinks back at the same speed, carrying you down. If you step off it stays its full time (a bridge for a friend). |
| Cooldown | 3 s |
| Mastery | Height (above). M5: you may raise a pillar **under an aimed spot** within 12 blocks (an enemy standing there is lifted and takes 1 heart; a friend is lifted for free). M8: the pillar is **2×2**. M10: two pillars per use (one under you, one aimed). |
| Sneak: Earth Launch | The pillar fires you up like a catapult: a 2-block stub rises in 4 ticks and you are thrown upward with speed **1.2** +0.08 per M (about 8 blocks at M1, 14 at M10), no fall damage for the next landing on earth (the ground catches you with a soft dip). Cooldown 4 s. |
| Counters | A pillar is stone: Rock Throw breaks one block of it (the rider falls); fire does nothing; water erodes it (a Water Shot takes 5 s off its life). |
| Edge cases | Not standing on earth: refused. Something above you (a ceiling within height): the pillar stops under it, never crushes you. Another player standing on the same block: both ride. A mob: rides and is not hurt (except the aimed pillar). Under water: works from the floor and lifts you out. Claim: refused if the top cells are protected. Nether roof: stops at y=127. |

### 2.4 Earth Wall — קיר אדמה (the shield)

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Wall Push** (דחיפת קיר) |
| Unlock | 3 SP |
| What | A wall of the ground rises in **4 ticks** 3 blocks in front of you, facing you (snapped to the nearest of 8 directions), of the blocks under it (temporary). It is real blocks: arrows, rocks, fireballs, Air Blades and mobs stop at it. |
| Size | **3 wide × 2 high** at M1; +1 wide per 2 M, +1 high per 3 M: 5×3 at M5, 7×4 at M10. Thickness 1 (2 at M10). |
| Duration | **10 s** (Eilon), `wallSeconds` 200 ticks; then it sinks back. |
| Cooldown | 8 s (the HUD shows it; a new wall while one stands replaces it). |
| Mastery | Size (above). M6: the wall takes two rock hits per block instead of one. M10: thickness 2 and a parapet (the top row has gaps to shoot through). |
| Sneak: Wall Push | If your wall stands: it **slides forward** 6 blocks over 12 ticks (the temp blocks move one cell at a time), pushing creatures and players ahead of it (**1.5 hearts**, knockback 1.5, `AreaTargets`), then crumbles. If no wall stands: raises one and pushes at once (one cooldown of 10 s). |
| Counters | Rock Throw breaks one wall block per rock; a boulder breaks 4. Fire does nothing. Water Shot erodes (5 s off). Air Blast nothing. Lightning nothing. Metal Knives stick in it. A Lava Shuriken melts one block. Enhanced: a Chaos Beam breaks a block per second (through Enhanced's block-break event, which `TempBlocks` listens to: a broken temp block simply reverts early). |
| Edge cases | No earth under the wall's footprint: the wall is as wide as the earth found (at least 1 block, else refused). Blocks the wall would replace (grass, flowers, snow layers, water) are recorded and restored. A player or mob **inside** the footprint is pushed to the near side first (never suffocated). Claims: cells refused are left as gaps; more than half refused: no wall. Under water: a wall of the floor rises through the water (it is earth). Bosses: the Wither's skulls break it; the Dragon flies over. |

### 2.5 Sand Bending — כשפות חול

| | |
|---|---|
| Key | Slot; **press**: **Sand Stream** (זרם חול). Sneak: **Sand Wall** (קיר חול) |
| Unlock | 3 SP (needs Rock Throw M3 and Earth Wall) |
| Needs | `bending:sand` within reach (a desert, a beach, soul sand in the Nether, or **a stack of sand in your inventory**: 4 sand per use, returned... no: carried sand is used up: 4 per use **(added)**). |
| Sand Stream | Lift sand into the air and fling it in a cone: 12 + 2M pellets (entities pooled: one `SandStreamEntity` that hurts in a cone, drawn as many grains) to 10 blocks, **1 heart** total per target + **Blindness 2 s** (3 s at M6, 4 s at M10) + Slowness I 2 s. Cooldown 2.5 s. Mastery: more grains, longer cone (10 → 16), wider (20° → 35°). |
| Sand Wall | Like Earth Wall but of sand: rises in 3 ticks, same sizes, lasts **6 s**, and is **see-through from your side** (gaps every other block: you can shoot through it, so can they, but it stops rocks and arrows 70% of the time by random cell). Cooldown 6 s. |
| Counters | Water turns a Sand Wall to mud: a Water Shot ends it. Fire does nothing (glass is a **(suggestion)**: a Fireball turns a Sand Wall cell to glass, which stays as a temp block). Air Blast blows a Sand Wall away (ends it). |
| Edge cases | Soul sand works in the Nether. Suspicious sand is never bent (archaeology). Red sand gives red grains. A Sand Stream never blinds a boss or a player with PvP off. |

### 2.6 Quicksand — חול טובעני

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Sand Cloud** (ענן חול) |
| Unlock | 4 SP (needs Sand Bending M3) |
| What | The sand (or gravel, soul sand, mud, dirt at M8+) under an enemy within **12 blocks** turns to quicksand: a 3×3 patch of temporary **suspicious-sand-like** state (the mod's own `quicksand` block: a sand look, `SoulSand`-style sinking, no jumping) for **4 s** +0.4 per M (7.6 at 10). Whoever stands in it sinks to the chest, cannot jump, moves at ×0.2, and takes **0.5 heart per second**. It is harmless to the bender. |
| Limit (Eilon) | **Fails on an earthbender whose element strength ≥ yours**: their feet stay on top and the action bar tells you "Stronger earthbender". Also fails on bosses and flying targets. |
| Cooldown | 10 s |
| Mastery | Duration (above); patch 3×3 → 5×5 at M6; at M8 works on dirt and gravel too (not stone); at M10 holds **two** targets (two patches). |
| Sneak: Sand Cloud | A 6-block sphere of hanging sand around the aimed point for 5 s: everything inside has Blindness and takes 0.25 heart/s; projectiles entering it fall short. Cooldown 10 s. |
| Counters | Earth Pillar under yourself escapes it (the pillar lifts you out); an Air Jump or Fire Jet escapes; Water Shot at the patch turns it to mud, which frees everyone; a frozen patch (Freeze) is solid ground. |
| Edge cases | A patch never reaches under a block entity, a bed or a door. A quicksanded mob that was pathing stops pathing (we set no-AI for the duration: no, we just slow it; `Mob.getNavigation().stop()` once). Fall into it from above: counts as landing on soul sand (no extra fall damage). PvP off: no players. Nether: soul sand is the natural quicksand; works. |

### 2.7 Lava Bending — כשפות לבה

| | |
|---|---|
| Key | Slot; **press**: **Lava Path** (שביל לבה). Sneak: **Lava Shuriken** (שוריקן לבה 🌋) |
| Unlock | 6 SP (needs total Earth ES ≥ 20; "a rare art": Ghazan, Bolin) |
| Lava Path | A line of **temporary lava** (`bending:temp_lava`: looks and burns like lava, flows nowhere, never spreads fire, never makes stone) runs from your feet in the direction you look, 1 wide, **6 + M blocks** long (16 at M10), 1 block per tick, through earth only (it stops at water, air gaps and anything not earth). Lasts **12 s**, then the earth cools back (the blocks revert; the look: a crust that darkens). Creatures on it take lava damage; **the bender walks on their own path unharmed** while it lasts (fire resistance on the path's cells). Cooldown 12 s. |
| Lava Shuriken | A spinning disc of lava, **3 hearts** + burning 4 s, range 24, speed 1.5, cooldown 4 s; at M8 **two** discs; at M10 the disc **bounces once** off a block. Lights no fires (the burn is on the target). |
| Passive (Eilon) | **Partial fire and lava resistance** from mastery: M1–4 nothing; M5–7 fire and lava damage **×0.5**; M8–9 **×0.3**; M10 **lava immunity for 10 s after touching lava**, then ×0.3 (you can swim a little). |
| Mastery | Path length, two discs, bounce, the passive. Also M6: Lava Path **beside real lava** costs no cooldown (it draws from the lake). |
| Counters | Water: a Water Shot on the path turns that cell to cooled stone at once (and the shot hurts a lavabender's path's owner nothing); an Ice Wall across the path stops it. Air: nothing. Fire: a firebender takes ×0.5 from the path. |
| Edge cases | The path never crosses a claim border (it stops). Never under a block entity. In the End, end stone melts fine (it is earth). In water: refused. On a boat/minecart: no. Mobs: zombies burn, fire-immune mobs (blaze, magma cube, zombified piglins) ignore it. Items dropped on it burn as on lava (**(added)** no: temp lava never destroys items, to spare loot: items float on it). |

### 2.8 Metal Bending — כשפות מתכת (three cards)

**Unlock gate:** all three need Earth **ES ≥ 30** (Toph's invention came late) and a **Metal supply**. **Metal
charge (added):** a resource 0–30 shown on the HUD when a metal card is in the loadout. It is filled by **Metal Pull**
(below) from `bending:metal` blocks and ores within 8, 1 charge per metal block (an ore gives 1 charge and stays; a
block of iron gives 3 and stays), or **from your inventory**: 1 iron ingot → 2 charge, 1 iron nugget → 1 (consumed).
Knives, walls and armour spend charge; what a wall or armour used **returns** when it ends (the metal is yours), knives
return 1 charge each when picked up (they drop as the mod's `steel_knife` item... no: they *return to charge* when they
land within 16 of you after 2 s, else they are lost **(added)**).

#### 2.8a Steel Knives — סכיני פלדה. Sneak: Metal Pull — שליפת מתכת

| | |
|---|---|
| Unlock | 5 SP |
| Steel Knives | **3 knives** (M1) → 5 (M5) → 7 (M10), fanned 6° apart, **1.5 hearts** each, pierce one creature, range 32, speed 2.0, 1 charge each. They **stick in the target** (drawn, 3 s) and a stuck knife returns to you as charge when you press Sneak-key again within 5 s (Lin's recall). Cooldown 2.5 s. |
| Metal Pull | Hold up to 2 s: metal blocks and ores within **8 blocks** (+1 per 2 M) shed glowing filings that stream to your hands: +1 charge per block per second held, at most 10 per pull; **mobs in metal armour** lose nothing (that is Metal Grip's job). Also pulls iron ingots/nuggets lying as items within 8 into charge. Cooldown 6 s. |
| Counters | Earth Wall stops knives; Air Shield absorbs them (1/4 strength each); Water Octopus deflects one knife per 2 s; Fire melts none (steel); **Lightning** on a knife-stuck target ×1.5. A Steel Armour wearer takes ×0.5 from knives. |
| Edge | No charge: "No metal" and no cooldown. Knives never break blocks. They stop at the Hex wall if Enhanced says so. PvP off: no players. |

#### 2.8b Steel Wall — קיר פלדה. Sneak: Steel Armour — שריון פלדה

| | |
|---|---|
| Unlock | 5 SP |
| Steel Wall | Like Earth Wall but of **iron plates** (temporary `bending:steel_plate` blocks: metal look, blast-resistant, 2 wide × 2 high at M1 to 6×4 at M10), rising in 3 ticks **from your charge** (1 charge per 2 plates; returned when it ends). Lasts 10 s; rocks bounce off; Lava Shuriken dents it (no break); an Explosion breaks one plate (its charge is lost). Cooldown 8 s. |
| Steel Armour | **On/off**: plates (12 charge) cover the whole body but the eyes (`LOOKS.md` 1.2): shell **16 hearts** +1 per M, **+12 armour**, toughness 6, knockback resistance 0.8, speed ×0.8 (×0.9 at M8), **lightning ×1.5** (conducts), fire ×1.0, **a magnet to Magneto** (Enhanced's `Metal.isMetalHero` hook if present: he can lift you). Charge returns when shed. Cooldown 20 s after a break. Cannot be worn with Earth Armour. |
| Mastery | Sizes, shell; M10: the wall has a **door** cell you can walk through (others cannot: it opens for your UUID). |
| Edge | In water: steel armour sinks you faster (×1.5 gravity) but does not crumble. Nether: fine. Claims: plates are temp blocks with the same rights. |

#### 2.8c Wreck Metal — הרס מתכת. Sneak: Metal Grip — אחיזת מתכת

| | |
|---|---|
| Unlock | 6 SP |
| Wreck Metal | Aim at a **metal device** within 10 blocks and press: it crumples: the block becomes its wrecked form for `wreckSeconds` 60 (temporary: a **dark, bent** version of the block, `bending:wrecked_*` states: a wrecked hopper passes nothing, a wrecked rail derails, a wrecked piston does not move, a wrecked redstone component is off, a wrecked furnace stops smelting, a wrecked anvil is a wrecked anvil; doors and trapdoors of iron jam shut or open); then it mends. Contents are never lost (block entities keep their NBT; the temp block carries it). Costs 0 charge, **gives 1**. Cooldown 4 s. Mastery: M5 wrecks a **3×3** of devices at once (all connected metal within 1 of the aim); M8 **5×5**; M10 also **minecarts** (derailed and stopped 10 s) and **iron golems** (Weakness and Slowness 10 s, 2 hearts). |
| The device list (Eilon's open question 5; `bending:metal_devices` tag) | **Default:** hopper, hopper minecart, dropper, dispenser, piston, sticky piston, observer, redstone repeater, redstone comparator, redstone torch (as "electrical"), lever, iron door, iron trapdoor, iron bars, chain, lantern/soul lantern, bell, anvil (all), cauldron, rail (all four), minecart (entity), furnace, blast furnace, smoker, lightning rod, heavy/light weighted pressure plate, tripwire hook, grindstone, stonecutter, iron/copper/gold blocks (crumple into a dented look, cosmetic), copper bulb, copper grate, copper doors and trapdoors. **Never:** chests, barrels, shulker boxes, beacons, enchanting tables, anything not metal. Servers edit the tag. |
| Metal Grip | Hold a **metal thing** in your grip (Enhanced's `Telekinesis` in a `Mode.METAL`, `TECH.md` 2): a mob or player in **iron, chainmail, gold or netherite armour** (at least chestplate or 2 pieces), an iron golem (M8+), a minecart, an armour stand with armour, or a metal block you tore out (an iron block, an anvil, rails): lift it, move it with your look, press again to fling (3 hearts at speed), Sneak-press to set down. Range 12 + M. Hold time ≤ 6 s + M (a player struggles free after that), cooldown 8 s. **Fails** on a player with no metal armour and on bosses. Players feel it as Enhanced's `Live.LIFTED` lock (no movement input). |
| Counters | A gripped player who sheds armour (opens inventory, removes the piece) drops free. A Fire Jet or Air Jump breaks a grip (the lifter loses them). Earth Armour (stone) is not metal: immune to Metal Grip. |
| Edge | Wrecking in a claim: refused (it is a block change). Wrecking another player's hopper line on a PvP-off server: allowed by default (`wreckNeedsPvp` false), since it mends in a minute; a server flips it. Redstone contraptions: a wrecked component is simply "off"; nothing is destroyed. Bosses: never gripped. |

### 2.9 Earth loadout examples (added)

* **Beginner:** X Rock Throw / Hold, C Earth Wall / Push, Alt Earth Pillar / Launch, K empty.
* **Toph:** X Rock Throw, C Earth Wall, Alt Steel Knives / Metal Pull, K Earth Armour / Shed.
* **Bolin:** X Rock Throw, C Earth Pillar, Alt Lava Bending, K Earth Wall.

---

## 3. Air — כשפות אוויר (release 1.1)

**Theme.** Ba Gua Zhang: circles, evasion, never a straight line; the airbender is light and quick. Colours: saffron
and sky `#F2B134` / `#CFE8FF`. **Passives (added, free):** fall damage ×0.5 (feather-light landing, a dip), movement
speed +10%, and jumping 1.5 blocks high; you may **double-tap Space to hover** 1 s (not flight: a cushion of air) with a
3 s cooldown.

### 3.1 Air Blast — הדף אוויר (free)

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Air Jump** (קפיצת אוויר) |
| What | A wave of air from both palms in a **cone** (20° half-angle at M1 → 30° at M10) to **8 + M blocks**: every creature in it is thrown **4 + 0.6M blocks** away (knockback velocity 1.2 + 0.1M), takes **0.5 heart** (the fall does the rest), arrows and fireballs in the cone are blown off course, fires in it go out, and items are scattered. Pushes players even with PvP off (`pushPlayersWithoutPvp` true). |
| Cooldown | 2 s |
| Mastery | Distance and cone (above); M6: a blast at the ground **while gliding** throws you up 6 blocks (Eilon's "shooting air at the ground launches you higher"; also works without a glider); M10: the blast is a **sphere** when Sneak-free pressed while airborne (pushes all around). |
| Sneak: Air Jump | A burst under your feet: you leap **1.0 + 0.1M** blocks/tick upward (≈ 6 → 12 blocks), no fall damage on the next landing; mid-air it works once per 2 s as a second jump. Cooldown 2 s. |
| Counters | Earth Armour wearers are pushed ×0.5; Steel Armour ×0.6; a bender on an Earth Pillar is pushed off (M4+ of the pillar: the pillar's parapet holds them... no, keep simple: pushed). Bosses ×0.2. A Water Spout rider is knocked off. |
| Edge | Under water: half range, pushes the bender back too (the water resists). In the Nether/End: normal. Pushing a mob into the void or lava: that is the game. A pushed player is `hurtMarked` so the client takes the velocity. |

### 3.2 Air Shield — מגן אוויר

| | |
|---|---|
| Key | Slot; **on/off** (press raises, press again lowers). Sneak: **Shield Burst** (פיצוץ המגן) |
| Unlock | 4 SP |
| What | A spinning **sphere of air** (radius 1.6 → 2.4 at M10) around you: arrows, rocks, knives, fireballs and ice knives that enter it are flung away; melee blows and beams go through at ×0.75 (air is not a wall). It has **strength 6 hearts** +0.8 per M (13.2 at 10): each stopped projectile costs its damage; at 0 it **breaks and bursts** (below). Costs 0.3 exhaustion/s. |
| Burst (Eilon) | When it breaks, it bursts in **every direction**: everyone within **4 blocks** (+0.3 per M) is thrown 6 blocks and takes **3.5 hearts** (default; Eilon's 4 is an option: `airShieldBurstHearts`, SUGGESTIONS 2). Then cooldown 12 s. |
| Duration | Until lowered, broken or 20 s (`airShieldMaxSeconds`; 0 = unlimited). Lowering it by hand does **not** burst (and costs a 6 s cooldown). |
| Mastery | Radius, strength, burst radius (above); M7: moving at full speed is allowed (M1–6: speed ×0.8 while up); M10: it also **deflects Lightning** once (the bolt curves away; the shield breaks). |
| Sneak: Shield Burst | Burst it **on purpose** at full strength: the same push and damage, plus the strength left is added as +0.5 heart per 2 hearts left. Cooldown 12 s. Without a shield up: raises and bursts after 10 ticks (a quick 2-heart repel), cooldown 8 s. |
| Counters | Rocks eat it fastest (2 hearts a rock, 6 a boulder); Fire Whip strokes go through at ×0.75; a Tornado from another airbender cancels it (both end); Water Shot passes through at ×0.75 (it is a stream, not a projectile) **(added)**. |
| Edge | In water: refused (bubbles). Riding: works. Bosses: the Wither's skulls are flung; the Dragon's breath passes. Enhanced's shield rules: an Air Shield is a `ShieldStyle` dome drawn by Enhanced's `ShieldFx`-pattern renderer (`ANIMATION.md` 3.2). |

### 3.3 Air Glider — גלשן אוויר (item)

| | |
|---|---|
| Item | `bending:air_glider` (a staff that opens into wings): crafted from **2 sticks, 1 string... (added: 3 sticks, 2 string, 2 phantom membrane... no, too rare for a kid)**: **4 sticks, 3 wool, 1 feather** in a T shape. Colour: craft it again with **any dye** (16 colours; the wings and the cloth of your looks' glider match; default saffron). Durability: none (it is bending, not a tool). |
| Who | Only an airbender can open it (others hold a stick). It goes in the **off hand** or main hand. |
| Glide | Hold **Space in the air** with the glider in a hand (or Sneak + right-click to open it): you glide like an elytra (we reuse the elytra physics by setting `FALL_FLYING` while the glider is open, as Falcon's Glide does in Enhanced), but slower (×0.8) and never faster than 1.2 blocks/tick; no rocket boosting; **Air Blast downward launches you higher** (3.1, M6; at M1–5 of Air Blast it still lifts 3 blocks). Closing: Sneak, or landing. The pose: `ANIMATION.md` 10.2. |
| Duration | Unlimited (it is wind); exhaustion 0.2/s. |
| Edge | Under water: closes. In the End over the void: it glides, it does not save you from the void. Other dimensions: works. In rain: fine. A non-airbender in Creative: no. |

### 3.4 Air Blades — להבי אוויר

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Air Slice** (חיתוך אוויר) |
| Unlock | 4 SP |
| What | A **crescent of concentrated air** (a thin blade, drawn as a translucent arc, `ANIMATION.md` 3.4) flies where you look, range 20 + M, speed 2.0, **2.5 hearts** (Eilon) +0.1 per M (3.4 at 10), cuts through **leaves, tall grass, crops (not harvested: trampled... no, cut: drops them, as shears would, only where rights allow and `airBladesCutPlants` true), cobwebs and vines**, and passes through one creature to hit a second at ×0.5. |
| Cooldown | 1.5 s. M4: **two** blades (a V); M8: **three**; M10: blades **curve** toward the aimed creature (homing ±10°). |
| Sneak: Air Slice | A horizontal **sweep** around you: every creature within 3 blocks takes 2 hearts and is knocked back 2 blocks; the sweep also cuts the plants in the ring. Cooldown 3 s. |
| Counters | Earth Wall, Sand Wall, Steel Wall and Ice Wall stop it dead; an Air Shield (another airbender's) flings it; Fire Whip at M7 "burns" it (cancels one blade); Water Octopus arms do not stop it. Earth Armour ×0.75. |
| Edge | Under water: refused. It never breaks blocks; it never cuts a claim's crops (rights). Bosses: full damage. PvP off: no players. |

### 3.5 Spiritual Projection — השלכה רוחנית

| | |
|---|---|
| Key | Slot; **press** (begins; press again ends). Sneak: **Deep Meditation** (מדיטציה עמוקה) |
| Unlock | 6 SP (needs Air ES ≥ 15) |
| What (Eilon) | You sit in **meditation** (the body stays, a real player entity in a sitting pose, invulnerable? **no**: hittable, see below) and your **spirit** leaves it: a translucent blue copy of you (`SpiritEntity`, `TECH.md` 3) that **flies freely** (Creative-like flight, speed 0.08), **passes through blocks**, **sees through walls** (everything hostile within 32 glows, caves are drawn with x-ray-like outline of air pockets: `ANIMATION.md` 2.5), **cannot interact**, cannot be hurt, and is invisible to mobs. Range: **64 + 8M blocks** from the body (128 → 144... : 64 + 8M = 144 at M10); beyond it the tether pulls you back. Duration **30 + 6M s** (90 at M10), then you snap back. |
| The body | Sits where you were (the meditation pose, `ANIMATION.md` 10.5), takes damage normally; **any damage snaps you back at once** (Eilon), with Nausea 3 s. Mobs *do* target it. Friends see a glowing blue thread from the body toward the spirit (within 32). |
| Cooldown | 60 s after return (30 at M10) |
| Mastery | Range, duration; M5: the spirit can **open the Bending screen** (to plan a loadout) and **mark** a spot (a waypoint arrow on the HUD for 5 min); M8: you see **ores**' faint glow within 8 blocks of the spirit (Jinora's sense; `spiritSeesOres` true... default **false**, SUGGESTIONS); M10: the body is **shielded by a thin Air Shield** (6 hearts) while you are out. |
| Sneak: Deep Meditation | Sit in meditation without leaving: heal **0.5 heart per 2 s** and restore 1 hunger per 10 s for up to 20 s; any damage or movement ends it; cooldown 30 s. (Also the pose for photos.) |
| Edge | Never across dimensions (the spirit stops at a portal). In water: the body drowns normally (snaps you back at 2 bubbles). In the void: the spirit cannot fly below y = min + 1. Spectators cannot. PvP: a player who hits the body sends you back; the body counts as you for every rule. Logout: the spirit ends. Death of the body: you die where the body is. Server: the spirit is a server entity the client "possesses" (camera entity, `TECH.md` 3), so other players see the spirit and the body. |

### 3.6 Air Bubble — בועת אוויר

| | |
|---|---|
| Key | Slot; **on/off**. Sneak: **Air Cushion** (כרית אוויר, on/off) |
| Unlock | 2 SP |
| What (Eilon) | In water, a bubble of air closes around your head: you **breathe** (air refills to full and stays full), see clearly (no water fog), and the bubble is drawn around the head (`ANIMATION.md` 3.6). Costs 0.2 exhaustion/s. Lasts until turned off or you leave water; it also protects against the Drowned's trident? no, just breathing and clarity. At M6 the bubble covers **one friend** within 2 blocks too; at M10 it also gives Dolphin's Grace (swim speed). |
| Sneak: Air Cushion | **On/off** (default on once unlocked): a cushion of air breaks every fall: **no fall damage at all**, a landing puff. (The passive halves fall damage; this removes it.) Costs 0.5 exhaustion per saved fall. |
| Edge | Lava: no. The bubble pops under a Water Spout's push. Bosses: nothing. The Nether: no water, so nothing; the cushion works. |

### 3.7 Air Scooter — קטנוע אוויר (the air ball)

| | |
|---|---|
| Key | Slot; **on/off** (press to mount, press or Sneak-jump to dismount). Sneak: **Air Spout** (עמוד אוויר) |
| Unlock | 4 SP |
| What (Eilon) | A spinning ball of air under you (an `AirScooterEntity` you ride, 0.9 blocks high, `TECH.md` 3): **great speed**: 0.7 blocks/tick (= 14 blocks/s, a sprint is 5.6) +0.03 per M (0.97 at M10), climbs 1-block steps, **crosses water** at the surface, and runs along walls for 1 s if you ride into one at M6+. You steer with W A S D and the look; jumping hops. Exhaustion 0.3/s. Duration: until dismounted or 60 s (`scooterMaxSeconds`, 0 unlimited); cooldown 5 s after dismount. |
| Mastery | Speed; M4: 2-block steps; M6: wall ride 1 s; M8: a passenger can **sit behind you** (a friend right-clicks the ball); M10: the ball **bursts** on dismount (2 hearts, push 4 blocks, 4 s cooldown added). |
| Sneak: Air Spout | A column of spinning air lifts you straight up **4 + M blocks** and holds you there while held (hold ability, 0.4 exhaustion/s, up to 10 s): the airbender's "pillar". Let go: you drift down with the cushion. Cooldown 6 s. |
| Counters | Quicksand under the ball: it keeps going (air). A Water Shot at the rider ends the ride (knocks you off). Lightning hits normally. Earth Wall: you stop. |
| Edge | Under water: the ball surfaces you. Riding it in a 1-block-high gap: no (needs 2). Boats/minecarts: no stacking. Bosses: a ball hits nothing. In the End: it does not float over the void. |

### 3.8 Tornado — טורנדו

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Air Wheel** (גלגל אוויר) |
| Unlock | 5 SP (needs Air Blast M4) |
| What (Eilon) | A **small tornado** (a `TornadoEntity`: a spinning funnel 2 blocks wide at the base, **3 + 0.5M** high → 8 at M10) rolls where you look at 0.5 blocks/tick for **4 + 0.3M s**, then fades. Everything it touches is **lifted** 1–3 blocks and spun (held in orbit for up to 2 s), takes **1 heart** on entering and **0.5 heart/s** inside, and is flung out 6 blocks when it ends or they escape. Projectiles in it are flung too. Range (how far you can send it): 16 + M. |
| Cooldown | 8 s. At most **2** of yours at once (M8+; one before). |
| Mastery | Height, duration, count; M5: you may **steer** it while holding the key (it follows your look); M10: a **big** tornado (3 wide, 10 high, 2 hearts entering) when double-tapped. "Stronger bending makes the tornadoes, the shield and the other air abilities bigger, with longer range" (Eilon): yes, every size above is the mastery line. |
| Sneak: Air Wheel | A horizontal **wheel of wind** around your legs for 4 s: you **run at ×1.6** speed and every creature you run into is thrown 4 blocks (1 heart); Air Blades fired while wheeling cost no cooldown for the first two. Cooldown 10 s. |
| Counters | Earth Wall and Steel Wall stop it; a boulder dispels it; Fire Circle inside it becomes a **firenado** (**(suggestion)**: a tornado over a Fire Circle burns: +1 heart/s to everyone inside, 4 s); Water Spout vs Tornado: both end. Bosses are never lifted. |
| Edge | Under water: a whirlpool: half damage, pulls instead of lifts (it is still a tornado entity, in water). Claims: the tornado lifts no blocks (never; it is entities only). Items: lifted and dropped, never destroyed. A tornado never crosses a Hex wall (Enhanced hook). |

### 3.9 Zaheer's Path — דרכו של זהיר (two cards, gated)

**The gate (Eilon):** when **your tamed pet dies** (any `TamableAnimal` or `OwnableEntity` you own: a wolf, a cat, a
parrot, a horse you tamed... **(added: horses count; a sky bison of 8.7 counts)**; Eilon's open question 3, default:
**any death of any tamed pet counts**, config `zaheerAnyPetDeath` true, else only a pet that died *while you were
within 16 blocks*), you "cut your earthly tethers": the two cards below **unlock for free** (they appear on the tree
with a red thread icon), and from then on **every Air ability pushes less and hits harder**: knockback ×0.7, damage
×1.3 (`zaheerPushFactor`, `zaheerDamageFactor`). This is permanent (saved), unless `/bending zaheer reset`. The HUD shows
a small broken-thread icon. A server can turn the whole path off (`zaheerPath` true).

#### 3.9a True Flight — תעופה אמיתית

| | |
|---|---|
| Key | Slot; **on/off**; also **double-tap Space** like Enhanced's fliers. Sneak: **Hover** (hold: stand still in the air) |
| What | Creative-style flight (Enhanced's `applyPassives` flight attribute pattern), speed 0.06 (0.09 at M10), the flight profile `ANIMATION.md` 10.6 (Zaheer's cross-legged float at rest, arms back in flight). Exhaustion 0.4/s; at hunger 0 you glide down. |
| Mastery | Speed; M10: no exhaustion while hovering still. |
| Edge | Not in water (you swim). Not through blocks. In the End: fine. The Nether roof: no. |

#### 3.9b Suffocation — חניקה

| | |
|---|---|
| Key | Slot; **hold** (aim at a creature within 8 + M blocks, hold the key). Sneak: **Air Vacuum** (ואקום אוויר) |
| What (Eilon) | You pull the air out of the target's lungs: they take **1 heart per second**, Slowness II and cannot sprint or jump, their air meter drains as if drowning (the bubbles show), **unless they get far enough from you**: the hold breaks when they are **more than 8 + M blocks** away, or out of your sight for 10 ticks, or you take damage of 1+ heart. Max hold **6 + 0.4M s**, then cooldown **20 s**. One target at a time. Players only with PvP on; never bosses; never Creative. |
| Mastery | Range, hold time; M10: Slowness III and the target is **lifted** 0.5 block (Zaheer and the Earth Queen). |
| Sneak: Air Vacuum | A 3-block sphere where you aim is emptied of air for 3 s: everyone inside takes 1 heart/s and cannot jump; fires and Fire Circles inside go out at once; a Flamethrower aimed into it dies. Cooldown 15 s. |
| Counters | An Earth Pillar launch or Air Jump or Fire Jet breaks line-of-sight/distance; an Air Bubble (another airbender's) makes you immune; Earth Armour and Steel Armour do not help (the air is inside). A Water Shot at the suffocator breaks the hold (damage). |
| Edge | Under water: the target is already drowning: it just hurries it (0.5 heart/s). Mobs that do not breathe (undead, iron golems, blazes, endermen? — the tag `bending:no_breath`: undead, golems, blazes, magma cubes, slimes, wardens, shulkers, vexes, allays) are immune. |

### 3.10 Air loadout examples (added)

* **Beginner:** X Air Blast / Jump, C Air Blades / Slice, Alt Air Scooter / Spout, K Air Shield / Burst.
* **Tenzin:** Air Blast, Air Shield, Tornado, Spiritual Projection.
* **Zaheer:** Air Blast, Air Blades, True Flight, Suffocation.

---

## 4. Water — כשפות מים, with ice and blood (release 1.2)

**Theme.** Tai Chi: flowing, continuous, turning the enemy's force. Colours: sea blue and white `#3C8DD6` / `#E8F6FF`;
ice `#BFE9FF`; blood `#7A0E1E`.

### 4.0 The water source (Eilon)

A waterbender needs water. Every Water ability (not the ice-only shaping of existing ice, not blood bending) **draws
from one of three sources**, tried in this order; the HUD's **water gauge** shows which one you have (`UI.md` 3):

1. **Open water nearby**: any water block (source or flowing), rain, snow layers or a cauldron of water within
   **`waterReach` 8 blocks** (+1 per 2 M of the ability used), or you standing in water. Unlimited.
2. **The Water Skin** (item `bending:water_skin`): crafted from **3 leather and 1 string** (Eilon: "cow leather and a
   little string"). It holds **8 charges** (`waterSkinCharges`; shown as a bar on the item and on the HUD). A Water
   Shot costs 1 charge, a Whip 1, a Healing second 1 per 4 s, an Octopus 3 to form, Arms 2, an Ice Knife 1, an Ice
   Wall 3, a Spout 4; the Skin **refills** by right-clicking water (full at once), standing in rain (1 charge / 10 s),
   or at a cauldron (1 charge per level). You may carry several skins; the fullest is used. The Skin also **catches**
   water your abilities release (a shot that lands within 8 returns 0.5 charge at M6+).
3. **Plants and flowers** (Eilon; needs **mastery 7 in Water Shot or Healing**): with no other source, the ability
   draws from the nearest plant within 8 (`bending:plants`: crops, flowers, grass, ferns, leaves, saplings, bushes,
   cactus, kelp...): the plant **dries** (crops reset to age 0 — no: become **dead bush**/withered: a flower becomes a
   dead bush, tall grass vanishes, leaves turn to brown "dry leaves" temp state for 60 s, crops lose 2 growth stages; a
   cactus is a water source without drying, Hama's trick). One plant = 1 charge. Never in a claim (rights).

With **no source**: the action bar says "No water" (אין מים), no cooldown, and the gesture plays a dry reach.
Water that an ability leaves in the world (a shot's splash, a spout's pool) is **temporary water** that evaporates in
2–5 s (never a permanent water block, never flows).

**Passives (added, free):** swim speed +30%, no slowness in water, Water Breathing for 10 s after any Water ability
under water (not infinite: that is Air's). **Full moon:** every Water ability deals ×1.25 on a full-moon night
(`fullMoonFactor`), the HUD shows the moon (`UI.md` 3.6). **Day of the sun (suggestion):** a solar eclipse does not exist
in Minecraft; nothing.

### 4.1 Water Shot — יריית מים (free, "very easy to unlock")

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Water Whip** (שוט מים, hold) |
| What | A stream of water (a `WaterShotEntity`, drawn as a ribbon) flies where you look: **1.5 hearts** +0.15 per M (2.85 at 10), knockback 1.0, range 20 + M, speed 1.8; it **extinguishes** a burning target and any fire it passes (1.7), **×1.5 vs firebenders**, and **wets** the target (3 s; lightning ×1.5). Cost 1 charge. |
| Cooldown | 1 s. M4: two streams (one per hand) per press; M8: three; M10: the stream **arcs** over a wall (a lobbed shot when you aim above 30°). |
| Sneak: Water Whip | **Hold**: a whip of water hangs from your hand (length **4 + 0.6M** → 10 at M10) and lashes at what you look at every 10 ticks: **1 heart** per lash, pulls the target 1 block toward you per lash (Katara's whip), reach = length. Costs 1 charge to start, 0.4 exhaustion/s. Max hold 8 s, cooldown 4 s. |
| Counters | Fire **evaporates** the whip's tip (a Fire Whip crossing it ends both for 1 s); an Earth Wall stops a shot; an Air Shield lets it through at ×0.75; a Fireball hit by a shot bursts harmlessly (1.7). Freeze on your own shot in flight = an ice knife (**(suggestion)** combo at M10). |
| Edge | Under water: the shot is a pressure jet, range 8, no cost. Lava: a shot into lava hisses and does nothing to the lava (no obsidian: temporary rule). Nether: Skin only. Bosses: full damage (a Blaze takes ×2 as vanilla water would... no, vanilla does not; we give **×1.5 vs fire-type mobs**: blaze, magma cube, strider: `waterVsFireMobs`). Rain: half cost. |

### 4.2 Healing — ריפוי

| | |
|---|---|
| Key | Slot; **hold**. Sneak: **Cleanse** (טיהור, press) |
| Unlock | 3 SP |
| What (Eilon) | Hold: glowing water gathers on your hands and heals **yourself** (no target in sight within 4 blocks) or **the friend you look at** within 4 + 0.3M blocks: **1 heart per 2 s** at M1 (1 heart per 1.5 s at M5, **1 heart per second at M10**); also slowly removes Wither and Poison (halves their time every 2 s); costs 1 charge per 4 s and 0.5 exhaustion/s; the target stands in a glowing water veil (`ANIMATION.md` 4.2). Mobs: heals your pets and villagers; never hostile mobs. Players: anyone, PvP or not. Both of you must be mostly still (movement speed over 0.2 blocks/tick breaks it). |
| Cooldown | None (a hold); 20 s max hold then 2 s rest. |
| Mastery | Speed and range (above); M4: heals **two** friends in your look's cone (2 blocks apart); M7: plant-drawn source (4.0); M10: **revives a downed Enhanced healer**? no — M10: heals **1 heart/s** and removes **Weakness, Slowness, Blindness, Nausea** too. |
| Sneak: Cleanse | A splash on the target (or yourself): removes **Poison, Wither, Nausea, Blindness, Hunger** at once and 1 heart; cost 2 charges; cooldown 10 s. Also puts out a burning target. |
| Counters | None needed; a healer in water is hard to interrupt (the source never runs dry). Fire Healing (5.6) is the weaker twin. |
| Edge | Healing a boss: never. Healing yourself at full health: nothing happens, no cost (no mastery XP either). Nether: Skin only; the Skin empties fast (1.9). Spectators: no. |

### 4.3 Water Octopus — תמנון מים

| | |
|---|---|
| Key | Slot; **on/off**. Sneak: **Octopus Lash** (הצלפת תמנון) |
| Unlock | 5 SP (needs Water Shot M3) |
| What (Eilon) | Water wraps your body and **eight arms** (6 at M1, 8 at M5) wave around you (`ANIMATION.md` 4.3). **Defence:** each arm **slaps away one projectile** (an arrow, a rock, a knife, a fireball; not beams or blades) per 2 s within 2.5 blocks, and melee blows against you are **×0.75**. **Offence:** when you hit someone with your hand or a weapon within 3 blocks, **2 arms strike them too**: **+1 heart** (+0.1 per M) and a push of 1 block; also any creature that comes within 2 blocks and is hostile to you is lashed once per 2 s for 0.5 heart. Cost **3 charges** to form, 0.4 exhaustion/s; **speed ×0.85** while up. Lasts until turned off, until you leave water reach for 10 s (the arms need a source to keep their shape; the Skin counts), or 40 s (`octopusMaxSeconds`, 0 unlimited). Cooldown 10 s after it drops. |
| Mastery | Arms, damage; M4: reach 3 → 4; M7: arms also **catch a thrown rock and throw it back** (1 heart) once per 4 s; M10: the arms **grab**: a creature that hits you is **held** for 1 s (no movement) once per 5 s. |
| Sneak: Octopus Lash | All arms lash forward in a cone of 4 blocks: **2 hearts**, pull 2 blocks toward you; cooldown 4 s. Without an octopus up: forms it and lashes (5 charges). |
| Counters | Fire: a Flamethrower on you evaporates one arm per second (the octopus shrinks); Fire Circle removes 3 arms; Freeze from another waterbender freezes your arms solid for 2 s (you cannot act); Lightning ×1.5 on you (you are wet). Earth: rocks are slapped; a boulder breaks 2 arms. Air: Air Blast pushes you ×1 (the arms don't root you). |
| Edge | Under water: the octopus is free and the arms are 8 at any mastery (you are in your element). Nether: the Skin keeps it 10 s per 3 charges. Riding: the octopus drops. Bosses: the arms do not grab them. |

### 4.4 Water Spout — עמוד מים (the vortex)

| | |
|---|---|
| Key | Slot; **hold** (ride while held). Sneak: **Wave Ride** (רכיבה על גל) |
| Unlock | 6 SP (needs Water ES ≥ 15; "needs high mastery; otherwise very small") |
| What (Eilon) | A **vortex of water** rises under you from a water source **you stand in or beside** (within 3 blocks; the Skin makes a small one: height ×0.5): a spinning column (a `WaterSpoutEntity` you ride) that lifts you to a height of **2 + M blocks** (3 at M1 — "very small" — 12 at M10) and **moves with you**: W A S D steers at 0.3 blocks/tick over water, 0.15 over land (the spout drags its base; over land it loses 1 block of height per 2 s away from water). While riding you can use every other ability. Let go: the column collapses into a splash and you fall (the water cushions: no fall damage within 12 blocks). Cost 4 charges to start, 0.5 exhaustion/s, max ride 15 s +1 per M, cooldown 10 s. |
| Attack (Eilon: "ride onto an enemy and push them, dealing heavy damage") | **Steering the spout into a creature** rams it: **4 hearts** +0.3M (7 at 10), knockback 3 blocks, once per 2 s per creature; the spout's spin also pulls creatures within 2 blocks of the base in and lifts them 1 block, 0.5 heart/s. |
| Mastery | Height, ride time, ram damage (above); M5: works from the Skin at full height; M8: the spout **freezes** on command (Sneak-press while riding: it becomes a temporary ice pillar for 6 s and you stand on it); M10: two riders (a friend may jump on). |
| Sneak: Wave Ride | A **wave** rises from the water and carries you forward 12 + M blocks at 0.8 blocks/tick along the surface (Korra's surf), pushing creatures ahead of it 3 blocks (1 heart); works only from water (not the Skin). Cost 2 charges, cooldown 8 s. |
| Counters | Lightning on the spout collapses it (1.7); Fire does little (steam: ×0.5 to the rider); Freeze from another waterbender freezes the column (you are stuck on top 2 s); an Air Blast knocks you off; a Tornado meeting a Spout ends both; Earth: a Rock Throw at the base does nothing, at the rider full damage. |
| Edge | Lava: never (no source). The Nether: Skin-only small spout. The End: no water unless you brought it. Under a ceiling: the spout stops below it. Bosses: a ram deals full damage, no lift. |

### 4.5 Water Arms — זרועות מים

| | |
|---|---|
| Key | Slot; **on/off**. Sneak: **Arm Grab** (אחיזת זרוע) |
| Unlock | 4 SP |
| What (Eilon) | Your two arms are wrapped in long arms of water (Ming-Hua): **reach** for melee becomes **4 + 0.5M blocks** (4.5 → 9 at M10; vanilla is 3) against players, animals and monsters; **attack damage +1 heart** (+0.1 per M) on every hand blow (an empty hand deals 1 heart base + the bonus: a water fist); attack speed +20%. You may **not hold items** in your hands while the arms are on (your hands are water: the hotbar still works for selection, items are shown on your back... simpler: items in hand are not drawn and do nothing; `ANIMATION.md` 4.5). Cost 2 charges, 0.2 exhaustion/s; lasts until off or 60 s (`armsMaxSeconds`, 0 unlimited); needs a source within reach every 10 s. Cooldown 6 s. |
| Mastery | Reach, damage; M4: the arms **climb**: hold Space at a wall within reach and you are pulled up 4 blocks (Ming-Hua's climbing), 2 s cooldown; M7: the arms **swing**: Sneak + Space at a ceiling pulls you up to it; M10: arms **freeze on hit** for +0.5 heart (ice tips). |
| Sneak: Arm Grab | Grab the creature you look at within reach and **pull it to you** (1 heart, it lands 1 block in front of you); players only with PvP; bosses never. Cooldown 4 s. |
| Counters | Fire evaporates the arms if you are held in a Flamethrower 2 s (they drop; 6 s cooldown); Freeze from another waterbender freezes them (you cannot attack 2 s); Lightning ×1.5 on you. Earth Armour takes ×0.5 from water fists. |
| Edge | Under water: the arms are free and reach +2. Nether: the Skin feeds them (1 charge / 10 s). Bosses: full damage, no grab. A wrapped hand cannot use a bow, shield, or eat (open the Bending screen or toggle off first); the HUD says so. |

### 4.6 Ice — קרח (three cards)

#### 4.6a Ice Knives — סכיני קרח. Sneak: Ice Spikes — חודי קרח

| | |
|---|---|
| Unlock | 3 SP (needs Water Shot M2) |
| Ice Knives | Water freezes in your hand into **3 knives** (M1) → 5 (M6) → 7 (M10), **1.5 hearts** each, range 24, speed 2.0, pierce 0; a hit **wets and chills**: Slowness I 3 s (II at M8); the knives shatter into ice particles. Cost 1 charge per 3 knives, cooldown 2 s. **Fire ×1.5 vs the chilled target** (1.7). |
| Ice Spikes | A line of **temporary ice spikes** (the mod's `ice_spike` block, 1 block, pointed) erupts from the ground where you aim, 5 + M long, 1 block per tick, through any ground (not only water: the spikes are made of your water): **2 hearts** to anything they rise under + knockback up; the spikes stand 5 s then melt. Cost 3 charges, cooldown 6 s. |
| Counters | Fireball melts knives in flight (1.7); Earth Wall stops them; Air Shield flings them; a Flamethrower melts spikes in 1 s. Steel Armour ×0.5 from knives. |
| Edge | Nether: knives melt after 8 blocks (half range), spikes last 2 s. Lava: a spike into lava does nothing. Bosses: full. |

#### 4.6b Freeze — הקפאה. Sneak: Snow Throw — זריקת שלג

| | |
|---|---|
| Unlock | 3 SP |
| Freeze (Eilon: "freeze water") | Aim at **water**: the surface within a **3 + M radius** disc (4 → 13 at M10) turns to **temporary ice** (frosted ice look, 20 s +2 per M, then it melts back; you may walk across — a bridge; boats stop). Aim at a **creature standing in or on water, or wet** (Water Shot in the last 3 s, rain): its **feet freeze**: it cannot move for **2 + 0.2M s** (4 at M10), takes 0.5 heart; players with PvP only; bosses never. Aim at **another waterbender's water construct** (octopus, arms, spout, whip): it **freezes solid** 2 s (4.3–4.5). Aim at a Fire Circle or lava path: a hiss, nothing (Fire melts Ice). Cost 1 charge per 10 blocks of ice (free on open water), cooldown 4 s. |
| Snow Throw | A **snowball volley**: 4 + M snowballs (the mod's `snow_burst` projectiles, 0.5 heart each, Slowness I 2 s, extinguish fire, ×3 vs blazes like vanilla snowballs), range 16, cooldown 2 s. From M6 the snow **settles**: a 3×3 of temporary snow layers where the volley lands (slows walkers). Cost 1 charge. |
| Counters | Fire melts the ice disc (a fireball melts a 3×3 hole), a firebender frees frozen feet in 1 s by any fire ability (the self-cast Fire Circle does it); Earth Pillar under frozen feet frees you; Air Jump frees you. |
| Edge | Deep ocean: the disc freezes the top layer only. Nether: nothing to freeze unless you brought water; the frozen-feet use works on wet targets (rain does not exist there: only Water Shot wets). Freezing a creature mid-air: no (needs ground). Frozen players keep looking around and can bend. |

#### 4.6c Ice Wall — קיר קרח. Sneak: Ice Dome — כיפת קרח

| | |
|---|---|
| Unlock | 4 SP (needs Freeze M2) |
| Ice Wall (Eilon: "between you and an enemy") | A wall of **temporary ice** rises in 3 ticks **between you and the creature you look at** (at the midpoint, perpendicular to the line; with no creature, 3 blocks ahead facing you): 3×2 at M1 → 7×4 at M10, lasts **10 s**, see-through (ice), stops projectiles and creatures; **fire melts it** (1 fireball per 4 blocks, a Flamethrower 1 block/s: 1.7); rocks break 1 block each. Cost 3 charges (free on open water), cooldown 8 s. |
| Ice Dome | A dome of ice (radius 2 → 3 at M10) closes over you (and friends within it) for 8 s: nothing gets in or out but Fire melts a hole in 2 s; you may bend from inside (Ice Knives through your own dome: the dome lets your projectiles out). Cost 5 charges, cooldown 15 s. |
| Edge | Nether: ice lasts half as long. Lava: an ice wall touching lava hisses away in 1 s. Claims: temp blocks, same rights. Water: a wall in water is an ice wall in water (fine). |

### 4.7 Blood Bending — כשפות דם (two cards, gated)

**The limits (Eilon):** 1. It **never works on a waterbender whose element strength ≥ yours** (they stand firm, their
eyes glow blue at you). 2. **Until you fully master it** (Blood Bending at **mastery 10**), it works **only on a
full-moon night** (moon phase 0 in `level.getMoonPhase()`, between 13000 and 23000 of the day; the HUD shows the moon
and a red tint grows in the corners while you blood-bend, `ANIMATION.md` 2.2). In the Nether and the End there is no
moon: only M10 works there. Server options: `bloodMoonOnly` true, `bloodWorksOnPlayers` true (with PvP), `bloodBending`
true.

**Unlock:** 8 SP and Water **ES ≥ 40** (Hama taught it to a master). Mastery rises by use **only on full moons** until
M10 (so reaching M10 takes about 66 full-moon nights of practice... too slow: mastery XP is **×4** for blood bending
uses, so ~165 successful puppet-moves, roughly 15 full moons; **and** it may be bought with SP up to 7 as any ability).

#### 4.7a Blood Bending — כשפות דם. Sneak: Blood Fling — הטלת דם

| | |
|---|---|
| What (Eilon) | **Hold** the key on a creature or player within **10 + M blocks**: they are **lifted** 1 block and **move like a puppet**: your look moves them (they float where you aim, up to 6 blocks from where they stood per second), they cannot move, attack, bend or use items; their limbs twist (`ANIMATION.md` 12.4, the Live bit `BLOODBENT` their client reads to lock input, as Enhanced's `PinLock`). Damage **0.5 heart per second** held. Max hold **4 + 0.6M s** (10 at M10), then cooldown **25 s**. One target (two at M10: the second within 4 blocks of the first). Cost 0.6 exhaustion/s, no water. |
| Fails on | a waterbender with ES ≥ yours; bosses; Creative/Spectator; players with PvP off; the undead, golems and anything in `bending:no_blood` (undead, iron/snow golems, blazes, magma cubes, slimes, wardens, shulkers, vexes, allays, endermen? — endermen have blood: yes they are bent). |
| Sneak: Blood Fling | While holding: **fling** the puppet where you look at speed 1.6 (**3 hearts** on landing + fall damage), ending the hold (cooldown as above). Without a hold: grab and fling at once (2 s of hold then fling: a quick 2.5-heart throw). |
| Mastery | Range, hold time, two targets; the M10 full-moon freedom. |
| Counters | **Another waterbender's Blood Bending** on you: the stronger wins (ES), a tie favours the defender; a **firebender's Lightning** hitting the blood bender breaks every hold; **damage of 2+ hearts** to the blood bender breaks the hold; Earth Armour wearers are bent (flesh under stone) but Steel Armour at M10 holds 50% of the time (Toph's... no: no exception; removed). An Air Bubble: no help. Spiritual Projection: a spirit cannot be bent; a meditating body can. |
| Edge | Under water: works. Through walls: needs line of sight to start, keeps the hold through 1 block. A bent player's client locks movement input (`MovementInputUpdateEvent`), their camera is free. Riding: dismounted. The Nether/End: M10 only. A mob bent over lava: its death is yours (SP counts). |

#### 4.7b Bending Severance — שלילת כשפות (full mastery)

| | |
|---|---|
| Unlock | Appears when Blood Bending is **M10** (free then). |
| What (Eilon) | **Up close** (within **2 blocks**, Amon's touch: you must be bending the target with 4.7a, then press this card's key, or press it with the target within 2 and not held: it does both: a 1 s hold then the touch), you take the target's bending away for a time **by their strength**: `severanceSeconds = 600 × 1 / (1 + ES/10)` → **ES 1 ≈ 9 minutes** ("about 10 minutes for a weak bender"), **ES 40–60 ≈ 1.5–2 minutes** ("1–2 for a fairly skilled one"), ES 100 ≈ 55 s. The target's HUD greys out, every ability refuses ("Your bending is gone"), their looks stay, their points stay; the time runs down on their HUD; **a Healing (4.2) of 20 s at M10 by another waterbender, or Fire Healing M10 for 40 s, restores it early (suggestion; `severanceHealable` true)**. Cooldown **120 s**. Players only with PvP; also works on **(suggestion)** Enhanced heroes if present (their powers via `Powers.setPower`? **no**: never; only bending). |
| Fails on | the same list as 4.7a, plus anyone with no element. |
| Edge | The severed may still fly with a glider? **No**: the glider needs bending. They can still use the Water Skin to drink? There is no drinking. They keep passives? **No**: passives are bending too (the airbender falls like anyone). Death does not restore it early (the timer is saved on the player). |

### 4.8 Water loadout examples (added)

* **Beginner:** X Water Shot / Whip, C Ice Knives / Spikes, Alt Healing / Cleanse, K Freeze / Snow.
* **Katara:** Water Shot, Healing, Water Octopus, Ice Wall.
* **Ming-Hua:** Water Arms, Ice Knives, Water Shot, Freeze.
* **Amon:** Blood Bending, Bending Severance, Water Shot, Ice Knives.

---

## 5. Fire — כשפות אש, with lightning (release 1.3)

**Theme.** Northern Shaolin: sharp, linear, fast, driven by the breath. Colours: flame orange and gold `#FF6A1F` /
`#FFD23F`; **Azula's blue fire** `#3FA4FF` / `#CFF2FF` (the look changes every fire colour, `LOOKS.md` 4.3); lightning
white-blue `#DDF4FF`.

**Passive (Eilon): fire hurts firebenders less:** fire, lava and burning damage **×0.5** (`fireResistFactor`; ×0.3 at
Fire ES ≥ 50), burning time ×0.5; you can **extinguish yourself** by sneaking (1 s). **(added)** The sun: by day fire
abilities deal ×1.1, at night ×1.0 (Sozin's Comet is a **(suggestion)** rare event).

### 5.1 Fireball — כדור אש (free)

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Small Fireball** (כדור אש קטן) |
| What | A ball of fire from the fist, range 24 + M, speed 1.6, **2 hearts** +0.2 per M (3.8 at 10) + burning **3 s** (4 at M6); it bursts on impact in a 1.5-block flash (0.5 heart to others nearby) and lights **no fires** unless `[fire] setsFires` and `mobGriefing` (then up to 3 fire blocks, rights). |
| Size (Eilon) | The ball grows with mastery: radius 0.3 → 0.9 blocks (hitbox and look), the burst 1.5 → 3 blocks at M10 (1 heart to bystanders). |
| Cooldown | 1.2 s (24 ticks). M5: two balls (one per fist) per press; M8: three; M10: a **great fireball** on double-tap (radius 1.2, 5 hearts, burst 4, cooldown 4 s). |
| Sneak: Small Fireball (Eilon: "a very skilled bender can choose to keep it small so it deals little damage") | From **M6**: a marble of fire, **0.5 heart**, no burning, no burst, lights a campfire/candle/TNT? — lights **campfires and candles** (not TNT, not blocks), range 16, cooldown 0.5 s: for lighting things and for play. Before M6 the Sneak use is a normal fireball with half damage. |
| Counters | Water Shot bursts it harmlessly (1.7); Air Blast deflects it; an Earth Wall or Ice Wall stops it (ice melts: 1 ball per 4 blocks); Air Shield flings it (eats 2 hearts). A waterbender takes ×1.0; a frozen target ×1.5. |
| Edge | Under water: refused. Rain: ×0.8. Nether: ×1.25. Bosses: full; the Wither, blazes, magma cubes are fire-immune (vanilla): 0 damage, the HUD says "Immune". Items on the ground: never burned. |

### 5.2 Fire Punch — מכת אש

| | |
|---|---|
| Key | Slot; **on/off** (fists blaze). Sneak: **Fire Kick** (בעיטת אש, press) |
| Unlock | 2 SP |
| What | While on, every **empty-hand melee hit** is a fiery fist: **+1.5 hearts** (+0.1 per M) and burning 2 s, a flash of flame, and the hit **ignores 2 armour points per M**... no: **ignores 20% of armour** (fire goes through the gaps). Reach 3 (vanilla). Costs 0.1 exhaustion per hit. Lasts until off or 60 s (0 unlimited). Cannot hold items while on (the fists are the weapon). |
| Mastery | Damage; M4: the third hit in a row **knocks back 2 blocks** (a combo); M7: every hit also throws a **small fire jab** 3 blocks ahead (0.5 heart) if the fist misses; M10: hits do **+2.5 hearts**. |
| Sneak: Fire Kick | A spinning kick with a crescent of flame: **2 hearts** to everyone in a 2.5-block arc in front, knockback 2, burning 2 s; cooldown 3 s; usable mid-air (a flip). |
| Counters | Water Octopus arms take the hit (×0.75) and put the fire out on you? no: on the target. A frozen target ×1.5. Steel Armour ×1.0, Earth Armour ×0.5. |
| Edge | Under water: ×0.5 and no burning. Bosses: full. PvP off: no players. |

### 5.3 Flamethrower — להביור

| | |
|---|---|
| Key | Slot; **hold**. Sneak: **Fire Breath** (נשיפת אש, hold; from M7) |
| Unlock | 4 SP (needs Fireball M3) |
| What (Eilon: "a continuous jet of fire from both hands") | Hold: two jets of fire from the palms (Enhanced's `ContinuousBeam` in a flame style, `TECH.md` 2): range **8 + 0.6M** (14 at M10), a cone 1.5 blocks wide at the end, **1 heart per 0.5 s** per creature in it (damage type with no cooldown, as Enhanced's beams) + burning 3 s; melts ice walls 1 block/s and snow; lights no fires unless `setsFires`. Cost 0.8 exhaustion/s; max hold **6 s** then 4 s rest (the breath); you move at ×0.6 while firing. |
| Mastery | Range; M4: 1 heart per 0.4 s; M7: **Fire Breath** unlocked; M10: the jet **pushes** targets back 0.5 block/s. |
| Sneak: Fire Breath (Eilon: "breathe fire from the mouth, bigger and stronger") | From **M7**: a single wider jet from the mouth: range 10 + 0.6M, cone 2.5 wide, **1.5 hearts per 0.5 s**, max hold 4 s, rest 6 s, you cannot move while breathing (head back, the dragon's breath pose). Before M7, Sneak is the normal jet from one hand at ×0.5 (a "warm-up"). |
| Counters | A Water Shot across the jet puts it out for 1 s and the waterbender takes nothing extra; an Air Blast bends the jet aside; a Tornado snuffs it; Earth Wall stops it; an **Air Vacuum** kills it. Firebenders take ×0.5. |
| Edge | Under water: refused. Rain: ×0.8. Nether: ×1.25. Bosses: full. Lighting TNT: never. A jet into a Lava Path: nothing. |

### 5.4 Fire Whip — שוט אש

| | |
|---|---|
| Key | Slot; **hold** (the whip is in your hand while held). Sneak: **Whip Lash** (הצלפת שוט, press) |
| Unlock | 4 SP |
| What (Eilon) | A whip of fire hangs from your hand (length **5 + 0.7M** → 12 at M10) and **strikes what you look at** every 12 ticks at full reach: **1.5 hearts** per stroke (+0.1 per M) and **everything it hits catches fire** (burning 4 s; **blocks** too: 1 fire block at the hit point if `setsFires` and rights; a Fire Whip on an Ice Wall melts 2 blocks per stroke). Cost 0.5 exhaustion/s, max hold 10 s, cooldown 4 s. Movement ×0.8 while held. |
| Mastery | Length; M4: two whips (one per hand) when double-tapped: strokes alternate every 6 ticks; M7: a stroke **cuts a rock in flight** (2.1) or burns an Air Blade; M10: the whip **wraps**: a struck creature is pulled 2 blocks toward you (Zuko's catch). |
| Sneak: Whip Lash | One **long crack** at full length +3: **3 hearts**, burning 4 s, knockback 2; cooldown 3 s. |
| Counters | Water Whip crossing it: both end for 1 s; Water Shot on the wielder: the whip is out for 2 s; Air Shield: strokes at ×0.75; an Earth Wall blocks it (the whip strikes stone and scorches it: a smoke puff). |
| Edge | Under water: refused. Rain: strokes ×0.8 and no block fires. Nether: ×1.25. PvP off: no players. Flammable blocks around you: the whip's own body lights nothing; only the strike point does, and never in a claim. |

### 5.5 Fire Circle — מעגל אש

| | |
|---|---|
| Key | Slot; **press**. Sneak: **Fire Jet** (סילון אש, hold) |
| Unlock | 5 SP (needs Fire Punch M3) |
| What (Eilon: "spin with your hands to send a ring of fire into everyone around you; a skilled bender also uses the legs, for stronger fire") | You spin (the gesture: 360° in 12 ticks, `ANIMATION.md` 5.5) and a **ring of fire** expands from you to **4 + 0.4M blocks** (8 at M10) over 10 ticks: everyone it crosses takes **2 hearts** (M1–5, hands only) or **3 hearts** (**M6+**: the leg sweep joins: a second, lower ring 4 ticks later) + burning 3 s + knockback 2; the ring **snuffs** incoming Water Shots... no: water beats fire: a Water Shot cuts a gap (1.7). Melts ice within the radius, puts nothing out. Fires: up to 12 fire blocks on the ring's edge if `setsFires` (never in claims). |
| Cooldown | 8 s. M10: a **double circle** (press twice within 10 ticks: the second ring at ×0.5 damage). |
| Sneak: Fire Jet | **Hold**: jets from your feet and palms **lift you**: 0.9 blocks/tick upward at first then hover-climb 0.2/tick, steerable (W A S D at 0.3), for up to **3 + 0.3M s** (6 at M10): Ozai's and Azula's jet-flight, short bursts, not true flight; fall damage from the end of a jet ×0.5. Cost 1 exhaustion/s, cooldown 6 s. Also breaks a grip (Metal Grip, Quicksand, frozen feet). |
| Counters | Water: a Water Shot into the ring cuts it; a Water Spout rider is above it; Air Blast pushes the ring? no: snuffs the part of the ring inside its cone (1.7); Air Vacuum kills it; a Tornado over a Fire Circle makes a **firenado** (3.8). Earth Armour ×0.5. Ice Wall: melts a 2-block hole. |
| Edge | Under water: refused. In rain: ×0.8. Nether: ×1.25. A ring never crosses a Hex wall or a claim's edge (the fire blocks). Bosses: full damage, knockback ×0.2. |

### 5.6 Fire Healing — ריפוי באש

| | |
|---|---|
| Key | Slot; **hold**. Sneak: **Burn Out** (צריבת רעל, press) |
| Unlock | 3 SP |
| What (Eilon: "heal others and help against poison; a bit weaker than water healing") | Hold on a **friend** you look at within 4 blocks (or yourself with no friend in sight): warm golden fire-light flows into them: **1 heart per 3 s** at M1 (1 per 2 s at M5, 1 per 1.5 s at M10: always a step behind Water's), and **Poison and Wither tick slower** (halved) while held; costs 0.5 exhaustion/s; max hold 20 s, rest 2 s; both must be still. (Chi-based healing from the dragons' teaching in *Korra*.) |
| Sneak: Burn Out | Burns the **Poison and Wither** out of the target at once (and 0.5 heart of damage to them: it hurts), plus Hunger; cooldown 15 s. |
| Mastery | Speed; M10: removes Weakness and Slowness too, and restores 1 hunger per 10 s. |
| Edge | Never hostile mobs or bosses. In water: works (it is chi, not flame). Nether: works. PvP: heals anyone. |

### 5.7 Lightning Generation — יצירת ברק

| | |
|---|---|
| Key | Slot; **hold to charge, release to fire** (min charge 10 ticks). Sneak: **Lightning Arc** (קשת ברק, press) |
| Unlock | 8 SP (needs Fire ES ≥ 30, "cold-blooded fire") |
| What (Eilon) | Hold: the two-finger form circles (`ANIMATION.md` 5.7, Iroh's path: both arms in slow circles that meet at the stomach then extend), sparks crawl on your arms, and on release a **bolt** (a `LightningBoltFx` line with 2–4 branches, instantaneous raycast to **range 32 + M**, stopped by blocks) hits the first creature on the line: **4.5 hearts** (Eilon) +0.15 per M (5.85 at M9), **paralysis 2 s** (Eilon; a `bending:paralysed` effect: no movement, no attack, no bending; players' input locked; bosses 0.5 s), **×1.5 vs wet/water/metal**, ×0.5 vs Earth Armour (1.7); a bolt that hits water spreads to everyone in that water within 4 blocks (×0.5). Charging longer (up to 30 ticks) adds nothing but the look; you cannot move while charging (Slowness... locked). |
| Cooldown | **20 s** (Eilon: "a long cooldown"; `lightningCooldown` 400 ticks). |
| Mastery | Damage, range; M5: paralysis 2.5 s; **M10: both hands** (Eilon: "fully mastered: shoot it with both hands for 6 hearts"): the bolt is two bolts that join, **6 hearts**, 3 s paralysis, cooldown 25 s; the dual-hand form is the default at M10 and a tap (release before 10 ticks) gives the single bolt. |
| Sneak: Lightning Arc | A short crackling arc from your fingers to **one creature within 6 blocks**: **1.5 hearts**, paralysis 0.5 s, cooldown 5 s (a jab, Mako-style). |
| Lightning Redirection (5.8) | **A bolt aimed at a firebender who redirects** goes back at you (5.8). |
| Counters | An Earth Wall or Steel Wall stops it (the steel wall **conducts**: anyone touching the wall takes 1 heart); an Ice Wall is shattered (4 blocks); an Air Shield at M10 deflects one bolt; a lightning rod within 8 of the aimed point **draws** the bolt (vanilla's rod: a nice touch: `lightningRodsAttract` true). Standing in water yourself: you take **1 heart** when you fire (a hazard). |
| Edge | **Vanilla lightning:** the bolt is the mod's own FX and damage type (`bending:lightning`, in `minecraft:is_lightning` so creepers it hits become **charged**, villagers do not turn into witches? — vanilla does that on a real `LightningBolt` entity only: we spawn **no** vanilla bolt, so no witches, no fires, no copper oxidation changes: **(added)** "pure" lightning). Thunderstorm: ×1.25. Nether: ×1.0 (not fire). Bosses: full damage, 0.5 s paralysis. PvP off: no players. A paralysed player can still open the Bending screen (cannot act). |

### 5.8 Lightning Redirection — הטיית ברק

| | |
|---|---|
| Key | Slot; **hold** (the stance: Iroh's redirection path). Sneak: **Ground It** (הארקה, press) |
| Unlock | 8 SP (needs Fire ES ≥ 35; "needs high mastery") |
| What (Eilon) | Hold the key as a bolt comes: while the stance is held, **any lightning that hits you** (another firebender's bolt or arc, **vanilla lightning**, Thor's beam if Enhanced is present and in `is_lightning`, Love's) **enters your arm, passes through your stomach** (the red-hot path drawn on your body and in first person, `ANIMATION.md` 5.8) **and leaves the other arm** where you look, **within 10 ticks of the hit** (you must release the key, or it fires along your look automatically at 10 ticks at M8+). Success chance **by mastery**: M1 20%, M2 30% ... **M9 100%** (`redirectChance` list; Eilon: "unskilled: the lightning will most likely hit you; skilled: 100% back"); on failure you take the bolt at ×1.25 (it went the wrong way through the heart: Iroh's warning) and the stance drops. On success you take **0** and the bolt leaves at **the same damage** (6 hearts if it was 6), with your paralysis duration, **at the attacker if you look at them, or anywhere else you look** (Eilon: "100% back at the attacker, or in another direction"). |
| Hold | The stance may be held up to **5 s** (then 2 s rest, no cooldown: it is a stance); while held you move at ×0.5 and cannot use other abilities; it costs 0.3 exhaustion/s. At M10 you may also hold it **1 s after** a bolt begins its charge in your sight (the HUD flashes the attacker's charge: "Lightning!" when a firebender within 40 blocks starts charging at you). |
| Sneak: Ground It | Send the lightning you hold **into the ground**: no damage anywhere, a crater of sparks (no blocks), and **you heal 1 heart** (Zuko's lesson: the energy passes safely). Also discharges a vanilla thunderstorm bolt harmlessly. |
| Mastery | The chance (above); M5: the window is 15 ticks; M8: auto-fire at the window's end; M10: redirect **two bolts** in one hold (Azula's dual bolt, 5.7 M10, counts as one bolt of 6). |
| Edge | Standing in water: the redirected bolt also shocks you 1 heart (1.7). Steel Armour on: redirection **fails always** (it conducts through the armour: the HUD says so). Vanilla lightning (a storm strike): redirectable, 2.5 hearts, sets no fire, charges a creeper you aim at. Bosses: a redirected bolt deals full damage to them. |

### 5.9 Fire loadout examples (added)

* **Beginner:** X Fireball / Small, C Fire Punch / Kick, Alt Flamethrower / Breath, K Fire Whip / Lash.
* **Zuko:** Fireball, Fire Whip, Fire Circle / Jet, Lightning Redirection.
* **Azula:** Fireball (blue), Flamethrower, Fire Circle / Jet, Lightning.
* **Iroh:** Fire Breath (Flamethrower), Fire Healing, Lightning, Lightning Redirection.

---

## 6. Progression

### 6.1 The skill tree (one per element)

Each element's tree is a **graph of cards** (`UI.md` 2.2): tiers from the root (the free starter) outward. An ability
can be unlocked when **its parent is unlocked and at the required mastery** and you have the SP. Unlocking gives the
ability at mastery 1 and puts it in the first empty loadout slot if any. Mastery levels are bought on the same card.

**Earth tree**
```
Rock Throw (free)
├─ Earth Pillar (2)                      [Rock Throw M1]
├─ Earth Wall (3)                        [Rock Throw M2]
│   └─ Sand Bending (3)                  [Earth Wall M1, Rock Throw M3]
│       └─ Quicksand (4)                 [Sand Bending M3]
├─ Earth Armour (4)                      [Rock Throw M2]
└─ Lava Bending (6)                      [ES ≥ 20]
Metal Bending (needs ES ≥ 30):
    Steel Knives (5) ─ Steel Wall (5) ─ Wreck Metal (6)   [each needs the previous at M1]
```
**Air tree**
```
Air Blast (free)
├─ Air Bubble (2)
├─ Air Blades (4)                        [Air Blast M2]
├─ Air Shield (4)                        [Air Blast M2]
├─ Air Scooter (4)                       [Air Blast M3]
│   └─ Tornado (5)                       [Air Blast M4, Scooter M1]
└─ Spiritual Projection (6)              [ES ≥ 15]
Zaheer's Path (a pet's death): True Flight (free), Suffocation (free)
```
**Water tree**
```
Water Shot (free)
├─ Healing (3)
│   └─ Water Octopus (5)                 [Water Shot M3, Healing M1]
├─ Water Arms (4)                        [Water Shot M2]
├─ Ice Knives (3)                        [Water Shot M2]
│   └─ Freeze (3)                        [Ice Knives M1]
│       └─ Ice Wall (4)                  [Freeze M2]
├─ Water Spout (6)                       [ES ≥ 15]
└─ Blood Bending (8)                     [ES ≥ 40]
    └─ Bending Severance (free)          [Blood Bending M10]
```
**Fire tree**
```
Fireball (free)
├─ Fire Punch (2)
│   └─ Fire Circle (5)                   [Fire Punch M3]
├─ Flamethrower (4)                      [Fireball M3]
├─ Fire Whip (4)                         [Fireball M2]
├─ Fire Healing (3)
├─ Lightning Generation (8)              [ES ≥ 30]
└─ Lightning Redirection (8)             [ES ≥ 35]
```

### 6.2 The mastery curve

| Level | XP needed (cumulative uses) | SP to buy (optional, ≤ level 7) | Typical time for an active player |
|---|---|---|---|
| 1 | 0 | — | at unlock |
| 2 | 10 | 1 | first evening |
| 3 | 25 | 2 | |
| 4 | 50 | 2 | a week of play |
| 5 | 90 | 3 | |
| 6 | 150 | 3 | |
| 7 | 230 | 4 | a month |
| 8 | 340 | — (use only) | |
| 9 | 480 | — | |
| 10 | 660 | — | the long game |

Mastery XP is also shown as a thin bar under the ability's icon on the HUD. A level-up plays the element's chime and
a small particle ring, and says "Rock Throw: mastery 5" in the action bar.

### 6.3 Full-tree cost (for tuning)

| Element | Unlock SP (all cards) | Buying every mastery to 7 | Total |
|---|---|---|---|
| Earth | 1+2+3+3+4+4+6+5+5+6 = 39 (the first 3 free) | 10 cards × 15 = 150 | ~186 |
| Air | 2+4+4+4+5+6 = 25 (Zaheer's two free) | 9 × 15 = 135 | ~157 |
| Water | 3+5+4+3+3+4+6+8 = 36 | 10 × 15 = 150 | ~183 |
| Fire | 2+5+4+4+3+8+8 = 34 | 8 × 15 = 120 | ~151 |

(Buying a card's levels 2–7 costs 1+2+2+3+3+4 = 15 SP.) At 1 SP per zombie and 25 per Warden, a full tree is a long
but reachable goal; the numbers are Eilon's open question 6 and all in `[progression]`.

---

## 7. Balance tables (one row per ability)

Hearts; range in blocks; cooldown in seconds; "M" = scales with mastery (the formula is in the ability's section).

### 7.1 Earth

| # | Ability | Key | Type | Cost SP | Dmg (M1 → M10) | CD | Range | Duration | Sneak variant (dmg, CD) | Counters |
|---|---|---|---|---|---|---|---|---|---|---|
| E1 | Rock Throw | slot | press | 0 | 2 → 3.8 (boulder 6) | 1.5 (3) | 24+M | — | Hold Rock (same, —) | Earth Wall, Air Shield, Octopus |
| E2 | Earth Armour | slot | on/off | 4 | shell 12 → 25.5; +8 armour | 20 after | self | ∞ (0=∞) | Shed Blast (1.5, 20) | Water ×1.5, Freeze |
| E3 | Earth Pillar | slot | press | 2 | aimed 1 | 3 | self / 12 | 15 s | Earth Launch (0, 4) | Rock breaks it |
| E4 | Earth Wall | slot | press | 3 | — | 8 | 3 ahead | 10 s | Wall Push (1.5, 10) | Rocks, Lava Shuriken, erosion |
| E5 | Sand Bending | slot | press | 3 | Stream 1 + blind | 2.5 | 10 → 16 | wall 6 s | Sand Wall (—, 6) | Water, Air Blast |
| E6 | Quicksand | slot | press | 4 | 0.5/s | 10 | 12 | 4 → 7.6 s | Sand Cloud (0.25/s, 10) | stronger ES, Pillar, Air Jump |
| E7 | Lava Bending | slot | press | 6 | lava; shuriken 3 + burn | 12 / 4 | 6+M / 24 | 12 s | Lava Shuriken (3, 4) | Water, Ice Wall |
| E8 | Steel Knives | slot | press | 5 | 1.5 × 3 → 7 | 2.5 | 32 | — | Metal Pull (—, 6) | Earth Wall, Air Shield, Steel Armour |
| E9 | Steel Wall | slot | press | 5 | — | 8 | 3 ahead | 10 s | Steel Armour (shell 16 → 25, 20) | Explosions, Lightning ×1.5 |
| E10 | Wreck Metal | slot | press | 6 | golem 2 (M10) | 4 | 10 | 60 s wrecked | Metal Grip (fling 3, 8) | shed armour, Fire Jet |

### 7.2 Air

| # | Ability | Type | Cost SP | Dmg | CD | Range | Duration | Sneak variant | Counters |
|---|---|---|---|---|---|---|---|---|---|
| A1 | Air Blast | press | 0 | 0.5 + throw 4 → 10 blocks | 2 | 8+M cone | — | Air Jump (—, 2) | Earth/Steel Armour ×0.5 |
| A2 | Air Shield | on/off | 4 | burst 3.5 (or 4) | 12 broken / 6 lowered | r 1.6 → 2.4 | 20 s (0=∞) | Shield Burst (3.5, 12) | rocks eat it, Tornado |
| A3 | Air Glider | item | craft | — | — | — | ∞ | — | — |
| A4 | Air Blades | press | 4 | 2.5 → 3.4 | 1.5 | 20+M | — | Air Slice (2, 3) | any wall, Fire Whip M7 |
| A5 | Spiritual Projection | press | 6 | — | 60 (30) | 64+8M | 30+6M s | Deep Meditation (heal, 30) | hit the body |
| A6 | Air Bubble | on/off | 2 | — | — | self (+1 friend M6) | ∞ in water | Air Cushion (on/off) | Spout push |
| A7 | Air Scooter | on/off | 4 | burst 2 (M10) | 5 | speed 14 → 19 b/s | 60 s (0=∞) | Air Spout (—, 6) | Water Shot, walls |
| A8 | Tornado | press | 5 | 1 + 0.5/s | 8 | 16+M | 4 → 7 s | Air Wheel (1, 10) | walls, boulder, Spout |
| A9 | True Flight | on/off | pet | — | — | — | ∞ | Hover | — |
| A10 | Suffocation | hold | pet | 1/s | 20 | 8+M | 6 → 10 s | Air Vacuum (1/s, 15) | distance, damage, Bubble |

### 7.3 Water

| # | Ability | Type | Cost SP | Dmg | CD | Range | Duration | Water cost | Sneak variant | Counters |
|---|---|---|---|---|---|---|---|---|---|---|
| W1 | Water Shot | press | 0 | 1.5 → 2.85 (×1.5 fire) | 1 | 20+M | — | 1 | Water Whip (1/lash, 4) | Fire Whip, walls |
| W2 | Healing | hold | 3 | heal 0.5/s → 1/s | — | 4 | 20 s hold | 1 / 4 s | Cleanse (—, 10) | — |
| W3 | Water Octopus | on/off | 5 | +1 → +1.9 per hit | 10 | 2.5 → 4 | 40 s (0=∞) | 3 | Octopus Lash (2, 4) | Flamethrower, Freeze, Lightning |
| W4 | Water Spout | hold | 6 | ram 4 → 7 | 10 | h 3 → 12 | 15+M s | 4 | Wave Ride (1, 8) | Lightning, Air Blast, Freeze |
| W5 | Water Arms | on/off | 4 | +1 → +1.9; reach 4.5 → 9 | 6 | reach | 60 s (0=∞) | 2 | Arm Grab (1, 4) | Flamethrower, Freeze |
| W6 | Ice Knives | press | 3 | 1.5 × 3 → 7 + slow | 2 | 24 | — | 1 / 3 | Ice Spikes (2, 6) | Fire ×1.5 melt, walls |
| W7 | Freeze | press | 3 | 0.5 + root 2 → 4 s | 4 | 16 | ice 20+2M s | 1 / 10 | Snow Throw (0.5 × 4+M, 2) | Fire, Pillar, Air Jump |
| W8 | Ice Wall | press | 4 | — | 8 | midpoint | 10 s | 3 | Ice Dome (—, 15) | Fire melts, rocks |
| W9 | Blood Bending | hold | 8 (ES≥40) | 0.5/s | 25 | 10+M | 4 → 10 s | 0 | Blood Fling (3, 25) | stronger ES, Lightning, 2 hearts |
| W10 | Bending Severance | press | free @M10 | — | 120 | 2 | 9 min → 55 s | 0 | — | ES, Healing M10 |

### 7.4 Fire

| # | Ability | Type | Cost SP | Dmg | CD | Range | Duration | Sneak variant | Counters |
|---|---|---|---|---|---|---|---|---|---|
| F1 | Fireball | press | 0 | 2 → 3.8 (great 5) + burn | 1.2 (4) | 24+M | — | Small Fireball (0.5, 0.5) | Water Shot, Air Blast, walls |
| F2 | Fire Punch | on/off | 2 | +1.5 → +2.5 + burn | — | 3 | 60 s (0=∞) | Fire Kick (2, 3) | Octopus, Earth Armour |
| F3 | Flamethrower | hold | 4 | 1 per 0.5 s (0.4 @M4) | 4 rest | 8 → 14 | 6 s hold | Fire Breath (1.5/0.5 s, 6 rest; M7) | Water Shot, Air Blast, Vacuum |
| F4 | Fire Whip | hold | 4 | 1.5 → 2.4 per stroke + fire | 4 | 5 → 12 | 10 s hold | Whip Lash (3, 3) | Water Whip, Water Shot |
| F5 | Fire Circle | press | 5 | 2 (3 from M6) + burn | 8 | r 4 → 8 | — | Fire Jet (—, 6) | Water Shot gap, Air Vacuum |
| F6 | Fire Healing | hold | 3 | heal 0.33/s → 0.67/s | — | 4 | 20 s hold | Burn Out (−0.5 to target, 15) | — |
| F7 | Lightning | charge/release | 8 (ES≥30) | 4.5 → 5.85; 6 dual @M10 + paralysis 2–3 s | 20 (25) | 32+M | — | Lightning Arc (1.5, 5) | walls, Redirection, Air Shield M10 |
| F8 | Lightning Redirection | hold | 8 (ES≥35) | returns the bolt (20% → 100%) | — | 5 s hold | — | Ground It (heal 1) | Steel Armour fails it |

---

## 8. Fun extras (suggestions; Eilon decides)

Everything here is **(suggestion)**: not in Eilon's spec, proposed because it fits the shows. Each is a separate lane in
`PLAN.md` so saying no costs nothing.

### 8.1 Being the Avatar (later idea, not built)
A fifth choice that bends all four elements, with the Avatar State (all abilities ×1.5 for 20 s when below 3 hearts,
glowing eyes and tattoos, the pose), the past lives as advice lines, and a world rule of **one Avatar per world**. It
changes the balance of everything above, so it is a 2.x idea: the data model keeps room for it (`BenderData.elements`
is a set, not a single value; `TECH.md` 4).

### 8.2 Training dummies
A block (`bending:training_dummy`, crafted from hay, sticks and a pumpkin... a wooden post with a straw body) that
takes bending hits and gives **mastery XP at half rate**, up to mastery **5** (you cannot master anything on a dummy),
shows the damage number above it for 1 s, and counts hits per minute. A **metal dummy** (iron bars + the dummy) trains
metal and lightning; a **water trough** next to it is a water source.

### 8.3 Bending scrolls in structures
Four kinds of **Bending Scroll** (Earth/Air/Water/Fire), found in the loot of strongholds, jungle temples (Earth),
desert pyramids (Fire), ocean monuments and shipwrecks (Water), and the end cities and tall mountains' structures of 8.4
(Air). Reading one of **your** element gives **3 SP** (or 1 mastery level to a chosen ability, up to 7); another
element's scroll can be traded to a bender of that element or **sold** to the Tutorial Master (8.4) for 1 SP.

### 8.4 Element temples and a tutorial master per element
Four structures, one per element, rare (one per ~2,000 blocks): the **Earth Kingdom gatehouse** (walls, a Dai Li
training ground, badgermole statues) in mountains and savannas; the **Air Temple** (a mountaintop spire with
spinning gates, a glider launch) on peaks; the **Water Tribe pavilion** (an ice dome on a frozen ocean, a healing hut,
a spirit oasis pool) in frozen oceans; the **Fire Sages' temple** (a volcano-side pagoda with braziers and the Sun
Warriors' dragon stairs) in badlands and basalt deltas. Each holds a **Tutorial Master** NPC (an element master in the
look of a teacher: a Bumi-like earth master, Gyatso-like monk, a Pakku-like elder, a Jeong Jeong-like hermit; **not**
named after the characters, so Eilon's looks stay the player's) who: explains the element in 6 short lines (EN/HE),
gives the first free scroll (8.3) once per player, trains (a dummy field), and sells looks' cosmetics for scrolls.
Enhanced's `AsgardTalkScreen`/NPC goal patterns are the model.

### 8.5 A boss per element
Four spirit bosses in the temples' depths (Enhanced's boss framework pattern, `TECH.md` 2): the **Badgermole King**
(Earth: tunnels, rock walls, seismic slams; drops the Earth boss scroll: +30 SP and the **Lava Bending** unlock for
free), the **Storm Spirit** (Air: a tornado that must be dispelled by Air Blasts; drops the Air scroll and a **sky
bison egg**, 8.7), **Koh-like "the Faceless"** (Water: in the spirit oasis, moves only when you are not looking at
him, steals your look for the fight; drops the Water scroll and **Spirit Water**, 1.6), and the **twin dragons Ran and
Shaw** (Fire: a dance, not a fight: match their fire with Fire Circle and Fireball in rhythm; they teach the **true
colours**: a rainbow fire cosmetic). Each 30 SP, once per player.

### 8.6 Duels and ranks
`/bending duel <player>`: a consented 1v1 with PvP on inside a 24-block ring for 3 minutes, no SP for the kill, a
**rank** per element on a scoreboard (Elo-like: `BenderRank`), shown as a small badge on the HUD. Pro-bending ring
(three zones, push the other team back) as a 2.x idea.

### 8.7 Companions
**Sky bison** (a flying mount from the Storm Spirit's egg or found at the Air Temple; six legs, a saddle for 3,
`ModelPart` model, flies at 0.6 blocks/tick, lands to graze; Appa's "yip yip" is a synthesised call), **winged lemur**
(a shoulder pet like a parrot, fetches dropped items), **polar bear dog** (Naga: a ride and a guard, from the Water
pavilion), **fire ferret** (Pabu: a shoulder pet that warms you, puts out your burning). Each is a `TamableAnimal`, so
its death counts for Zaheer's path (3.9), which is a story beat on purpose.

### 8.8 Small things that fit
Sozin's Comet (a 1-day event every ~40 days: fire ×2, the sky red); the **Day of Black Sun**... no eclipse exists;
**Pai Sho** table block (decor); **Cabbage merchant** (a wandering trader variant whose cart you must not break);
**Tea** (Iroh's jasmine tea: a drink that restores 2 hunger and gives 10 s of ×1.1 bending); **White Lotus tile** item
as the badge of a player who unlocked every card.

---

## 9. What is deliberately not in 1.x

* The Avatar (8.1), chi-blocking, combustion bending (Sparky Sparky Boom Man), seismic sense as a full x-ray (only
  Earth Armour's M8 glow), platinum, spirit portals, the Spirit World as a dimension, Vaatu/Raava, mecha tanks, bending
  for mobs (no bending zombies). Each is a clean later lane.
