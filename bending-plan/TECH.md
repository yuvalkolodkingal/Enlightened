# Bending — Technical Architecture

Mod id **`bending`**, group `com.kolodkingal.bending`, Minecraft **1.21.1**, **one universal jar for NeoForge 21.1 and
Fabric** (Fabric API allowed on Fabric only), Java 21, no runtime libraries, every model a `ModelPart`, every animation
code-driven. Built on Enhanced's foundations: this document says, system by system, what is **copied** (same code, new
package), what is **adapted** (same shape, element-specific content) and what is **new**.

## 0. The honest starting point: what Enhanced has today

Enhanced `main` (3.9.0, commit 888609c) has the **loader-neutral platform layer** of its port step P0: `platform/`
(`Platform`, `Net`, `Attach`, `Registrars`, `EventHub` + `@Subscribe` processor, `ClientRegistrar`, `ClientServices`,
`Keys`, `LoaderHooks`, `ItemAccess`, `Parts`, the ~80 common event classes), a complete **NeoForge implementation**
(`platform/neoforge/*`, the bridge `GameBindings`/`ClientBindings`, `NeoTestHooks`), a `portLint` Gradle task that
forbids NeoForge API outside `platform/neoforge/**`, the mod's own TOML config backend, and the Fabric **handshake**
(`HelloPayload`). It has **no Fabric implementation, no Fabric Loom build, no universal jar**, on any branch; the
"PLAN2" the code cites is not in the repository, and nothing documents how later Minecraft versions are added beyond
the `r1` adapter naming and the per-`<mc>-<loader>` parity report.

So Bending's universal jar requires **building the Fabric half** (section 2.3), which either Enhanced's own P1 lane
does first (and Bending copies it) or Bending's **P0 skeleton** does (and Enhanced copies it back). `PLAN.md` puts it in
Bending's **release 1.0, lane L0-FABRIC**, with the NeoForge-only jar as the fallback milestone so Earth is never
blocked by it. **Open question for Yuval:** share PLAN2 (and `SCRATCH/port`) if they exist; the plan below is written to
match the platform layer as it stands.

## 1. Coexistence with Enhanced (both mods in one game)

| Thing | Enhanced | Bending | Rule |
|---|---|---|---|
| Mod id / namespace | `enhanced` | `bending` | every id, texture, sound, tag, lang key, advancement, damage type is `bending:*` |
| Java package | `com.kolodkingal.enhanced` | `com.kolodkingal.bending` | no class name is shared on the classpath; the platform layer is a **copy**, not a dependency (two copies of `EventHub` coexist: each mod registers its own listeners on its own hubs; the NeoForge bridges each add their own native listeners) |
| Mixins | `enhanced.mixins.json`, package `…enhanced.mixin` | `bending.mixins.json`, package `…bending.mixin` | the same injection points on `PlayerModel.setupAnim` and `PlayerRenderer.setupRotations` are **additive** (`@Inject` at HEAD/TAIL, no cancels), so both animators run; Bending's `HeroPoses.apply` runs after Enhanced's and only touches a player who **has an element**, Enhanced's only a player with a power; a player with both gets Enhanced's pose then Bending's layered on (acceptable; the option `bending-client [animation] yieldToEnhanced` true makes Bending skip a player who has an active Enhanced gesture) |
| Payload channel | `enhanced:*`, version "14" | `bending:*`, version "1" (`ModNetwork.PROTOCOL_VERSION`), optional channel `bending:music` "1" | distinct `CustomPacketPayload.Type` ids; each mod's handshake is its own |
| Attachments | `enhanced:power`, … | `bending:bender`, `bending:victim`, `bending:temp_blocks`, `bending:world` | distinct keys; saved side by side on the player and the level |
| Config files | `enhanced-server.toml`, `enhanced-client.toml` | `bending-server.toml`, `bending-client.toml` (+ `<world>/serverconfig/bending-server.toml`) | the copied `Configs` backend writes its own files; the config screen lists its own |
| Key defaults | R G V B N H Y Z J (+ Ctrl chords, Alt/Del in build mode) | **X C Left-Alt K**, **O** | no default clash (1.5 of `DESIGN.md`); contexts `IN_GAME` |
| HUD | top-left panel | bottom-left panel (configurable corner) | no overlap by default |
| Creative tab | `enhanced` | `bending` | |
| Sound registry | `enhanced:*` | `bending:*` | |
| Soft hooks | — | `BendingCompat.enhanced()` (section 13) | Bending asks Enhanced's `Shelters`/`BossFights`/`Metal.isMetalHero` through reflection-free optional classes loaded only if `ModList`/`FabricLoader` says Enhanced is present |

## 2. Platform and build

### 2.1 Copied from Enhanced as is (`com.kolodkingal.bending.platform.*`)

`Platform`, `Loader`, `LoaderHooks`, `ItemAccess`, `Parts`, `EntityAttributesCompat`, `CommonTags`; `net/Net`,
`NetService`, `PacketContext`; `attach/Attach`, `AttachKey`, `AttachService`, `SyncTo`; `reg/Registrars`, `Registrar`,
`RegEntry`, `ItemRegistrar`/`ItemEntry`, `BlockRegistrar`/`BlockEntry`, `ComponentRegistrar`, `RegService`; `ext/ItemExt`,
`EntityExt`, `BlockEntityExt`; `event/*` (every common event, `EventHub`, `Events`, `Subscribe`, `Subscriber`, `Priority`,
`Side`, `Cancellable`, `TriState`) and `event/client/*`; `client/ClientRegistrar`, `ClientServices`, `Keys`, `KeyContext`,
`KeyMod`, `HudHeights`, `BlockModels`; **the whole `platform/neoforge/*`** (`NeoPlatform`, `NeoSetup`, `NeoNet`, `NeoAttach`,
`NeoPersistentData`, `NeoRegistrars`, `NeoHooks`, `NeoItemAccess`, `client/NeoClientRegistrar`, `client/NeoClientServices`,
`bridge/NeoForgeBridge`, `Binding`, `GameBindings`, `ClientBindings`, `ClientConvert`, `ListenerOrderRecorder`,
`ListenerOrderCommand`); the **annotation processor** (`processor/` → `SubscribeProcessor` generating
`GeneratedSubscriptions`, with `tools/listener-order.txt` recorded once from Bending's own listeners); the **config
backend** `config/core/*` (`ConfigSpec`, `ConfigFile`, `Configs`, `ConfigWatcher`, `Toml`, `ConfigSyncPayload`) and
`config/neoforge/ConfigNeoForge`; the **GameTest adapter** `platform/test/*` (`EnhancedTest` → `BendingTest`, `TestHooks`,
`TestIndex`, `TestFilter`, `ParityReporter`, `EnhancedTests` → `BendingTests`) and `platform/neoforge/test/*`; the
`portLint` task (rules `neoforge`, `subscribe`, `hidden`, `impl`, plus a new rule **`fabric`**: `\bnet\.fabricmc\.` outside
`platform/fabric/**`); `META-INF/services` files for the three SPIs. Only the mod id strings and the two `@Mod`
entry points change.

### 2.2 The entry points

* NeoForge: `Bending` (`@Mod("bending")`: `NeoSetup.init(modBus)`, `NeoRegistrars.register(...)` for each registrar in
  order, `ModAttachments.init()`, `ModNetwork.register()`, `Configs.register(SERVER, BendingConfig.SPEC)`, `Configs.register(CLIENT,
  BendingClientConfig.SPEC)`, `ConfigNeoForge.install(modBus)`) and `client/BendingClient` (`@Mod(value="bending",
  dist=CLIENT)`: `ClientSetup.register(NeoClientRegistrar.install(container, modBus)); ClientRegistrar.get().configScreen(ConfigScreen::new)`).
* Fabric: `platform/fabric/BendingFabric implements ModInitializer` and `platform/fabric/client/BendingFabricClient
  implements ClientModInitializer`, doing the same through the Fabric implementation (2.3), plus `BendingFabricGameTest`
  for the test adapter.
* `fabric.mod.json` (entrypoints `main`, `client`, `fabric-gametest`; `depends: fabricloader ≥ 0.16, fabric-api *,
  minecraft 1.21.1`; `mixins: bending.mixins.json`) and `neoforge.mods.toml` (as Enhanced's template) **both** in the jar's
  `META-INF`/root; each loader ignores the other's.

### 2.3 The Fabric half (NEW; lane L0-FABRIC of 1.0)

| Class (`platform/fabric/…`) | Implements | How |
|---|---|---|
| `FabricPlatform` | `Platform` | `loader()` FABRIC; `isPhysicalClient` via `FabricLoader.getEnvironmentType()`; `configDir` `FabricLoader.getConfigDir()`; `modResource` via `ModContainer.findPath`; `currentServer` from a `ServerLifecycleEvents` listener; `fakePlayer` `FakePlayer.get(level, profile)` (Fabric API's `net.fabricmc.fabric.api.entity.FakePlayer`); `hooks()`, `itemAccess()`, `parts()`, `attributes()` below |
| `FabricHooks` | `LoaderHooks` | `blockPlaceCancelled`: no Fabric place event exists: post the mod's own `BlockEvent.EntityPlaceEvent` on the hub **and** Fabric's `PlayerBlockBreakEvents`/claim mods' common pattern (`UseBlockCallback` is for interaction, not placement): call `BlockEvent.EntityPlaceEvent` on the hub only, and ask **Fabric Permissions API**-free claim mods through their own ("protection") events if present: documented limitation: on Fabric, claim mods that hook `PlayerBlockBreakEvents.BEFORE` are honoured for breaks (we post it); for places we post the hub event only. `blockBreakCancelled`/`blockBreakEventCancelled`: `PlayerBlockBreakEvents.BEFORE` + the hub event. `projectileImpactCancelled`: the hub event only (Fabric has none). `finalizeMobSpawn`: `mob.finalizeSpawn(...)` directly. `enderPearlTeleportCancelled`: hub only. `isShield`: `stack.getItem() instanceof ShieldItem || FabricShieldTag`. `drownsInFluidAtEyes`: `entity.isEyeInFluid(FluidTags.WATER) && !canBreatheUnderwater` |
| `FabricItemAccess` | `ItemAccess` | Fabric Transfer API `ItemStorage.SIDED` wrapped to the item-handler-like methods |
| `FabricParts` | `Parts` | `EnderDragonPart` only (no generic multipart on Fabric) |
| `FabricAttributes` | `EntityAttributesCompat` | swim speed: none on Fabric (returns null → callers skip); creative flight: the mod's own attribute `bending:creative_flight` registered on Fabric and applied through `Abilities.mayfly` in `PowerTicker`'s equivalent (`BenderTicker.applyPassives` sets `mayfly` directly when the attribute is absent) |
| `FabricNet` | `NetService` | `PayloadTypeRegistry.playC2S()/playS2C().register(type, codec)` for each message and `ServerPlayNetworking.registerGlobalReceiver` / `ClientPlayNetworking.registerGlobalReceiver` with a `Context` wrapping the Fabric context; the version handshake: **`HelloPayload`** registered in the **configuration phase** (`PayloadTypeRegistry.configurationS2C/C2S`, `ServerConfigurationNetworking.registerGlobalReceiver`, a `ServerConfigurationNetworking.ConfigurationTask`): the server sends its `HelloPayload.local()`, the client answers, `HelloPayload.check` decides (the exact words NeoForge shows); optional channels: `ServerPlayNetworking.canSend(player, type)` for `hasChannel`; the senders through `ServerPlayNetworking.send` / `PlayerLookup.tracking/world/around` |
| `FabricAttach` | `AttachService` | Fabric API **Attachment API** (`AttachmentRegistry.create(id, b -> b.initializer(def).persistent(codec).copyOnDeath().syncWith(syncCodec, AttachmentSyncPredicate.all()/targetOnly()))`): the same semantics as NeoForge's (`SyncTo.SELF` → `targetOnly()`); `persistentData` → a `CompoundTag` attachment `bending:persistent_data` |
| `FabricRegistrars` | `RegService` | `Registry.register` at `onInitialize` in declaration order; `entityAttributes` via `FabricDefaultAttributeRegistry.register`; `spawnEgg` via `SpawnEggItem` (Fabric allows a deferred type through `FabricSpawnEggItem`… use vanilla `SpawnEggItem` with the type registered first: Bending registers entity types before items) |
| `FabricPersistentData` | | the attachment mode only |
| `client/FabricClientRegistrar` | `ClientRegistrar` | `EntityRendererRegistry.register`, `BlockEntityRendererFactories.register`, `EntityModelLayerRegistry.registerModelLayer`, `LivingEntityFeatureRendererRegistrationCallback.EVENT` (→ `addLayers` with a `LayerContext` over the `EntityRendererProvider.Context`), `ParticleFactoryRegistry`, `KeyBindingHelper.registerKeyBinding`, `HudRenderCallback` (→ `guiLayer`; anchors mapped to a draw order: Fabric has one callback, so the layers are drawn in registration order **after** vanilla's HUD; `BELOW DEBUG_OVERLAY` ≈ after everything but the debug screen: acceptable), `ColorProviderRegistry.ITEM`, `BuiltinItemRendererRegistry`, `DimensionRenderingRegistry.registerDimensionEffects`, `ResourceManagerHelper.registerReloadListener`, `ClientCommandRegistrationCallback`, `configScreen` → a **ModMenu** entry point if present (`modmenu` optional) |
| `client/FabricClientServices` | `ClientServices` | `newKey` → `KeyMapping` (Fabric has no conflict contexts or modifiers: `KeyContext` kept by the mod's own check in `ClientEvents.handleKeys`, `KeyMod` checked with `Screen.hasControlDown()` etc.); `matches` = `mapping.matches(key)` && the context's `isActive`; `hudLeftHeight` → the mod's own tally (Fabric has none: Bending reads vanilla's `Gui` fields via an accessor mixin); `renderBatched` → `BlockRenderDispatcher.renderBatched` |
| `bridge/FabricBridge` | the event bridge | Fabric's events are callbacks, not a bus with priorities: the bridge registers **one Fabric callback per bound event** that posts to the hub at **all priorities in order** (`EventHub.post(E)`); cancellation maps to the callback's return value (`InteractionResult.FAIL` / `false`); ~40 bindings for the events Bending uses (section 2.5), each a small adapter: `ServerTickEvents`, `ServerLivingEntityEvents.ALLOW_DAMAGE/AFTER_DEATH`, `PlayerBlockBreakEvents`, `UseBlockCallback`, `UseItemCallback`, `UseEntityCallback`, `AttackEntityCallback`, `ServerPlayerEvents.AFTER_RESPAWN/COPY_FROM`, `ServerEntityEvents.ENTITY_LOAD/UNLOAD`, `ServerPlayConnectionEvents.JOIN/DISCONNECT`, `EntityElytraEvents`, `ServerLifecycleEvents`, client: `ClientTickEvents`, `WorldRenderEvents.AFTER_PARTICLES/START`, `HudRenderCallback`, `ClientPlayConnectionEvents`, `ClientPreAttackCallback`; events with **no Fabric equivalent** (`LivingIncomingDamageEvent` amount changes → `ServerLivingEntityEvents.ALLOW_DAMAGE` + a **mixin** on `LivingEntity.hurt` for the amount; `ComputeFovModifierEvent` → a mixin on `AbstractClientPlayer.getFieldOfViewModifier`; `ViewportEvent.ComputeCameraAngles` → a mixin on `Camera.setup`; `RenderHandEvent`/`RenderArmEvent` → mixins on `ItemInHandRenderer`; `MovementInputUpdateEvent` → a mixin on `KeyboardInput.tick`; `ProjectileImpactEvent` → a mixin on `Projectile.onHit`; `LivingKnockBackEvent` → `LivingEntity.knockback` mixin; `MobEffectEvent.Applicable` → `LivingEntity.canBeAffected`) are **Fabric-only mixins** in `bending.fabric.mixins.json` (`platform/fabric/mixin/`), each posting the common event. This is the bulk of L0-FABRIC. |
| `FabricTestHooks` | `TestHooks` | `fabric-gametest` entry point: `FabricGameTest`-annotated generator class → `BendingTests.enhancedTests()`; `mockPlayerWithConnection` as Fabric's `GameTestHelper.makeMockServerPlayerInLevel` plus a `Connection` on an `EmbeddedChannel` (the NeoForge recipe, loader-free) |

**Fabric-side limitations documented in-game** (a line in the Fabric config screen): claim mods are honoured for breaks
and interactions, placements only through Bending's own event; swim speed passive uses an attribute Bending registers.

### 2.4 The universal jar (NEW; L0-FABRIC with the build owner)

Gradle **multi-project**: `common/` (compiles against vanilla via **VanillaGradle** (or NeoForge's ModDevGradle in
"vanilla mode", `neoForge { version = … }` with `neoFormVersion`), Mojang mappings, `src/main/java` = everything but
`platform/neoforge` and `platform/fabric`; the mixins; the processor applied here), `neoforge/` (ModDevGradle 2.0.148,
depends on `common` as `compileOnly` + the `platform/neoforge` sources, `neoforge.mods.toml`, runs `runClient`, `runServer`,
`runGameTestServer`), `fabric/` (**Fabric Loom** 1.8+, `mappings loom.officialMojangMappings()` (so the code is one
mapping; Loom remaps the built jar to intermediary), depends on `common`, Fabric API, `fabric.mod.json`, runs
`runClient`, `runServer`, `runGameTest`). **One jar**: the `universal` task **merges** the two loader jars with
**Forgix**-style relocation (a build-time Gradle plugin `io.github.pacifistmc.forgix`, 1.2.x; **not** a runtime
library) producing `build/libs/bending-<version>.jar` that contains both loaders' classes (the common code appears
twice: once in Mojang names for NeoForge, once remapped to intermediary for Fabric, relocated to `…bending.neoforge.*`
and `…bending.fabric.*`? **No**: Forgix keeps the packages and resolves the duplicate common classes by **keeping both
loaders' trees under their own relocated roots and rewriting the metadata files to point at them**; the jar is about 2×
a single-loader jar, acceptable). CI builds all three and tests both: `./gradlew :neoforge:runGameTestServer
:fabric:runGameTest` and compares the parity reports `build/gametest/1.21.1-neoforge.json` and `1.21.1-fabric.json`
(`ParityReporter` already names files per loader; a `tools/parity.py` diff fails the build on a test that passes on one
loader only). **Cross-play**: both loaders speak the same `bending:*` payloads with the same codecs and the same
`PROTOCOL_VERSION`; **saves move** because every attachment is the mod's own id with the mod's own `Codec` on both
loaders (NeoForge's `AttachmentType` and Fabric's Attachment API both store under the mod's id in the entity's/level's
NBT: NeoForge under `neoforge:attachments`, Fabric under `fabric:attachments`: **not the same NBT path**: a world moved
between loaders would **lose** attachments). **Mitigation (NEW, S0 of 1.0): `SaveBridge`**: the mod also writes a mirror
of each persistent attachment into the entity's/level's **persistent-data compound under its own key**
(`bending:save` → the `CompoundTag` attachment `bending:persistent_data`, which is itself stored by each loader's
attachment system... circular). **Correct design:** Bending stores its **player data in the level's own storage**, not in
attachments: a `SavedData` (`BendingSavedData` in the Overworld's `data/bending_players.dat`, keyed by player UUID,
`Codec`-encoded) for everything that must survive a loader move (element, SP, unlocks, mastery, loadouts, look, zaheer,
severance timer, presets), and uses **attachments only for the synced runtime view** (`bending:bender` synced, not
persisted, rebuilt from the saved data at login/respawn). `SavedData` is vanilla and identical on both loaders. The
level-side data (`bending:world`: moon-bells state, severed timers of offline players, temp-block journal) is `SavedData`
too (`data/bending_world.dat` per dimension). **This is the save format of section 5.**

### 2.5 Events Bending uses (what the bridges must bind)

Server: `ServerTickEvent.Post`, `LevelTickEvent.Post`, `PlayerTickEvent.Post`, `EntityTickEvent`, `LivingIncomingDamageEvent`,
`LivingDamageEvent.Pre/Post`, `LivingDeathEvent`, `LivingKnockBackEvent`, `LivingFallEvent`, `LivingChangeTargetEvent`,
`MobEffectEvent.Applicable`, `EntityJoinLevelEvent`, `EntityLeaveLevelEvent`, `PlayerEvent.PlayerLoggedIn/Out/Respawn/Changed
Dimension/Clone`, `PlayerInteractEvent.RightClickItem/RightClickBlock/EntityInteract`, `AttackEntityEvent`, `BlockEvent.BreakEvent/
EntityPlaceEvent`, `ProjectileImpactEvent`, `EntityStruckByLightningEvent`, `EntityMountEvent`, `LivingEntityUseItemEvent`,
`ItemTossEvent`, `RegisterCommandsEvent`, `ServerStarting/Started/Stopping/StoppedEvent`, `OnDatapackSyncEvent`,
`LevelEvent.Load/Save/Unload`, `EntityMobGriefingEvent`, `ExplosionEvent.Detonate`. Client: `ClientTickEvent.Post`,
`InputEvent.Key/MouseButton.Post`, `MovementInputUpdateEvent`, `RenderLevelStageEvent`, `RenderPlayerEvent.Pre/Post`,
`RenderHandEvent`, `RenderArmEvent`, `RenderGuiLayerEvent.Pre`, `ViewportEvent.ComputeCameraAngles/ComputeFogColor/RenderFog`,
`ComputeFovModifierEvent`, `ClientPlayerNetworkEvent.LoggingIn/LoggingOut`, `SelectMusicEvent`, `ScreenEvent.Opening`,
`FMLClientSetupEvent`-equivalent.

### 2.6 Later Minecraft versions

Enhanced's code names its GameTest adapter "the **r1** (1.21.1) adapter" and writes parity reports per `<mc>-<loader>`,
which is the only trace of the method: **one release line per adapter**. Bending follows it explicitly: `common/` targets
the lowest supported version; version-specific vanilla API differences live behind a tiny **`mc/` adapter package**
(`platform/mc/McApi` with `r1` as the only implementation in 1.x: `getMoonPhase`, particle option constructors, `Camera`
accessors, anything that changed name between 1.21.1 and 1.21.x), and a new Minecraft version is added as a **Gradle
property matrix** (`-Pmc=1.21.4` selecting `neo_version`/`fabric_version`/`parchment`), a new `r2` adapter where
something moved, and the parity reports `1.21.4-neoforge.json` / `1.21.4-fabric.json` compared against `1.21.1-*`. The
universal jar is per Minecraft version (one jar per MC version, each universal across loaders), as every multi-loader
mod does. **Confirm against PLAN2 if it exists.**

## 3. Packages

```
com.kolodkingal.bending
  Bending, BendingConfig (server spec: the element configs' define()), client/BendingClient, client/BendingClientConfig
  platform/…                      (copied + fabric/)
  registry/   ModItems, ModBlocks, ModBlockEntities, ModEntities, ModDamageTypes, ModEffects, ModSounds, ModParticles,
              ModTags, ModDataComponents, ModCreativeTab, ModAttachments, ModStructures (suggestions), ModAdvancements
  bender/     Element (enum), BenderData (synced runtime view), BenderSave (SavedData record per player), BenderProgress
              (SP, unlocks, mastery: pure), ElementStrength, Loadout, Looks (variants per element), Look (record),
              Benders (get/set/sync helpers = Enhanced's Powers), BenderTicker (= PowerTicker), AbilityHandler (key router),
              Ability (enum of every card: id, element, slot act, sneak act, unlock cost, parent, gate), Mastery (curve),
              Cooldowns, Live (momentary bits), Toggle (saved bits), SkillTree (graph), ZaheerPath, Severance
  ability/    AreaTargets, BendingRights, Raycasts, TempBlocks (+ TempBlock, TempBlockJournal), Hazards (server-side
              objects base: Hazard, HazardTicker), Grip (= Telekinesis.hold for rocks/metal), SourceFinder (water sources),
              MetalCharge, Victims (the bending:victim attachment writer)
  earth/      EarthServer (ElementServer plug-in), EarthAbilities, RockThrow, EarthArmour, EarthPillar, EarthWall,
              SandBending, Quicksand, LavaBending, MetalBending (knives, wall, armour, wreck, grip), EarthEvents, EarthFx
              (server-side payload sender)
  air/        AirServer, AirAbilities, AirBlast, AirShield, AirBlades, SpiritualProjection, AirBubble, AirScooter, Tornado,
              ZaheerAbilities (flight, suffocation), Glider (item logic), AirEvents, AirFx
  water/      WaterServer, WaterAbilities, WaterSource, WaterShot, Healing, WaterOctopus, WaterSpout, WaterArms, Ice
              (knives, freeze, wall), BloodBending, WaterSkinItem, WaterEvents, WaterFx
  fire/       FireServer, FireAbilities, Fireball, FirePunch, Flamethrower, FireWhip, FireCircle, FireHealing, Lightning,
              LightningRedirection, FireEvents, FireFx
  entity/     RockEntity, SandStreamEntity, SteelKnifeEntity, LavaShurikenEntity, AirBladeEntity, AirScooterEntity,
              TornadoEntity, SpiritEntity, WaterShotEntity, WaterSpoutEntity, IceKnifeEntity, SnowBurstEntity, FireballEntity
  block/      QuicksandBlock, SteelPlateBlock, IceSpikeBlock, TempLavaBlock, WreckedBlock (+ 8 variants), TrainingDummyBlock (sugg.)
  item/       ElementScrollItem, WaterSkinItem, AirGliderItem, BendingScrollItem (sugg.)
  network/    ModNetwork, AbilityKeyPayload, BendingActionPayload (unlock/buy/loadout/look/preset), ElementScreenPayload,
              ElementChoicePayload, EarthFxPayload, AirFxPayload, WaterFxPayload, FireFxPayload, BeamPayload, SpiritPayload,
              GliderDyePayload, HelloPayload, ConfigSyncPayload, PayloadCodecs
  config/     EarthConfig, AirConfig, WaterConfig, FireConfig, ProgressionConfig, MatchupsConfig, LimitsConfig, ExtrasConfig
  command/    BendingCommand (/bending element|points|mastery|unlock|look|zaheer|duel|debug)
  client/     ClientSetup, ClientEvents, KeyBindings, BendingHud, overlays (ArmourSlitOverlay, SpiritOverlay, BloodOverlay,
              ParalysisOverlay, QuicksandOverlay, BubbleOverlay, HeatOverlay, RedirectPathOverlay), SpiritCamera,
              InputLocks (= PinLock generalised), PowerLoopSounds, music/ (ElementMusic, MusicSources…), screen/
              (ElementChoiceScreen, BendingScreen + tabs SkillTreeTab, LoadoutTab, LooksTab, WaterTab, CreatorUi, EditorTab,
              Popup, PreviewPuppet), render/ (the copied animation + FX toolkit, ElementClient plug-ins EarthClient/
              AirClient/WaterClient/FireClient with *Gestures/*Acting/*Fx, entity renderers, layers, BendingWorldFx,
              VictimPoses, LayerReveal, ChainSim, LightningFx, FlameBeamDrawer), particle/ (EnergyParticle, GrainParticle,
              DropletParticle)
  mixin/      the copied client mixins (PlayerModelMixin, PlayerRendererMixin, AbstractClientPlayerMixin, ItemInHandRendererAccessor,
              CapeLayerMixin, HumanoidArmorLayerMixin, LivingEntityRendererMixin [VictimPoses], MinecraftMixin+LevelRendererMixin
              [EntityGlow], ParticleAccessor), new: HumanoidModelMixin, QuadrupedModelMixin, CameraMixin, GuiAccessor;
              server: PlayerMixin (if needed), MobInvoker
  compat/     BendingCompat, EnhancedHooks (optional)
```

## 4. Registries (what S0 registers; the lanes fill behaviour)

**Items:** `element_scroll`, `water_skin` (durability-style bar via a `WATER` data component 0–8), `air_glider` (a
`DYE` component: `DyeColor`), suggestions: `bending_scroll_earth|air|water|fire`, `training_dummy`.

**Blocks (temporary states are real blocks with a `TempBlocks` ledger):** `quicksand` (a `SoulSand`-like block:
`getBlockSupportShape`, `entityInside` slows and sinks), `steel_plate`, `ice_spike` (3 `RISE` states), `temp_lava` (a
`LiquidBlock`-look non-flowing block: it extends `Block` with lava's render and `entityInside` lava damage via
`DamageSources.lava`, `lightLevel 15`, not a fluid: no spreading), `wrecked_hopper|rail|piston|redstone|iron_door|
furnace|anvil|metal_block` (8 `WreckedBlock`s carrying the original state in their block entity `WreckedBlockEntity`:
`original` + the original NBT), suggestions: `training_dummy`, `metal_dummy`. **Block entities:** `wrecked`, `training_dummy`.

**Entities** (all `MobCategory.MISC`, `noSummon` where internal):

| Entity | Size | Tracking / interval | Saved | Notes |
|---|---|---|---|---|
| `rock` (`RockEntity extends TelekineticBlockEntity`-pattern) | 0.9 (boulder 1.9) | 10 / 1 | no | carries a `BlockState`, held/launched, damage by speed |
| `sand_stream` | 1.0 | 8 / 2 | no | a cone hazard with a position; 10 ticks |
| `steel_knife` | 0.25 | 6 / 5 | no | sticks in targets (follows them for 3 s) |
| `lava_shuriken` | 0.6 | 8 / 2 | no | bounces once at M10 |
| `air_blade` | 0.9 wide 0.3 tall | 8 / 2 | no | passes one creature |
| `air_scooter` | 1.0 × 0.9 | 10 / 1 | no | a vehicle (`canAddPassenger`), steered by the rider's input (the client sends `Input` through vanilla's vehicle movement) |
| `tornado` | 2.0 × height | 12 / 2 | no | carries riders (orbit), drifts, re-sent every 4 ticks |
| `spirit` | 0.6 × 1.8 | 12 / 1 | no | `noPhysics`, the camera entity of its owner |
| `water_shot` | 0.4 | 8 / 2 | no | |
| `water_spout` | 2.0 × height | 10 / 1 | no | a vehicle |
| `ice_knife` | 0.25 | 6 / 5 | no | |
| `snow_burst` | 0.25 | 6 / 5 | no | a snowball variant |
| `fireball` | 0.3–1.2 | 8 / 2 | no | |

Hazards that are **not entities** (server-side `Hazard` objects ticked by `HazardTicker` from `LevelTickEvent.Post`,
Enhanced's `SandHazards` pattern): walls' lifetimes, pillars, quicksand patches, lava paths, ice discs, the Air Blast
cone (one tick), Air Slice, Fire Circle rings, the Air Vacuum sphere, the Suffocation hold, the Blood hold, the
Lightning charge/bolt, the redirection window, beams (`ContinuousBeam`: flamethrower, fire breath).

**Damage types** (`data/bending/damage_type/*.json`, all in the tag `bending:bending_damage`; the lightning ones also in
`minecraft:is_lightning`; the fire ones in `minecraft:is_fire`; beams in `bypasses_cooldown` + `no_knockback`): `rock`,
`boulder`, `earth_wall_push`, `shed_blast`, `pillar`, `sand`, `quicksand`, `lava_path`, `lava_shuriken`, `steel_knife`,
`metal_fling`, `wreck`; `air_blast`, `air_shield_burst`, `air_blade`, `air_slice`, `tornado`, `scooter_burst`, `suffocation`,
`air_vacuum`; `water_shot`, `water_whip`, `octopus`, `spout_ram`, `water_arm`, `ice_knife`, `ice_spike`, `snow`, `blood`,
`blood_fling`; `fireball`, `fire_punch`, `fire_kick`, `flamethrower`, `fire_breath`, `fire_whip`, `fire_circle`, `lightning`,
`lightning_arc`, `redirected_lightning`. Each has `death.attack.bending.<id>` and `.player` lines in both languages.

**Effects:** `paralysed` (no movement/attack/bending; the client locks input), `severed` (bending refused; duration =
the timer), `chilled` (slowness + ×1.5 fire), `wet` (3 s; ×1.5 lightning), `zaheer_tether_cut`? (no: saved flag), `entombed`-like
`stuck` (quicksand/frozen feet: no jump, speed ×0.2).

**Particles:** `energy` (copied), `grain`, `droplet`.

**Tags** (`data/bending/tags/`): block `earth`, `sand`, `metal`, `metal_devices`, `plants`, `unbendable`, `water_sources`
(water, cauldron, snow layers); entity_type `bosses`, `no_breath`, `no_blood`, `fire_mobs`; damage_type `bending_damage`,
`water_damage`, `fire_damage`, `lightning_damage`; item `metal_charge` (ingots/nuggets → charge value in a data map? no data
maps on Fabric: a `metal_charge_2` and `metal_charge_1` tag pair).

**Data components:** `water` (int 0–8), `glider_dye` (`DyeColor`).

**Advancements** (`bending:` tab): become each element, first unlock, mastery 10, full tree, Zaheer's path, a severance,
redirect a bolt, the extras.

**Creative tab** `bending` with the three items.

## 5. Data and save format

### 5.1 `BenderSave` (vanilla `SavedData`, `data/bending_players.dat` in the Overworld; one record per player UUID)

```
Codec<BenderSave> = RecordCodecBuilder:
  element: Element (string; "none")
  points: int                              // unspent SP
  pointsEarned: int                        // lifetime SP (for stats)
  unlocked: List<String>                   // ability ids (bending ability id "earth.rock_throw")
  mastery: Map<String, MasteryEntry{level:int, xp:int, bought:int}>
  loadout: List<String> (4 entries, "" for empty)
  presets: List<Preset{name:String, slots:List<String>}> (≤ 4)
  look: String ("" = default)
  gliderDye: String (DyeColor name)
  zaheer: boolean, zaheerAt: long
  severedUntil: long (game time, 0 = none), severedBy: UUID?
  elementChosenAt: long
  version: int (1)
```
Written on change (dirty) and on world save; read at login; **`BenderData`** (the synced runtime attachment, Enhanced's
`PowerData` shape: `element`, `look`, `loadout` ids as ordinals of `Ability`, `toggles` (int bits), `live`/`liveHigh`
(two longs of `Live` bits), `cooldownUntil[4]` + `sneakCooldownUntil[4]`, `armourHealth`, `metalCharge`, `shieldStrength`,
`waterCharge` (the fullest skin, for the HUD), `spiritId`, `heldEntityId`, `points`, `elementStrength`, `severedUntil`,
`zaheer`, the mastery levels as a `byte[]` by ability ordinal (for the HUD bars), `dirty`) is **not saved**: it is rebuilt
from `BenderSave` at login/respawn/dimension change by `Benders.load(player)` and synced to everyone who sees the player
(`SyncTo.ALL`) through the copied `Attach.builder("bender", BenderData::new).sync(STREAM_CODEC).build()`. Because the
stream codec sends ordinals, **every ability of one version must be in the same build** (Enhanced's rule for `Power`);
the saved ids are strings, so the order never matters on disk. The `/bending` commands edit `BenderSave` and call
`Benders.load`.

### 5.2 `WorldSave` (`SavedData` per dimension, `data/bending_world.dat`)

`tempBlocks: List<TempBlock{pos, original:BlockState(+nbt), placed:BlockState, revertAt:long, owner:UUID, kind:byte}>`
(the journal: reverted at load), `hazardsForRecovery` (none: hazards die with the server), `moonBellsHeard: Set<UUID>`
(cosmetic), `offlineSevered: Map<UUID, long>` (so a severed player who logs out stays severed: also in `BenderSave`).

### 5.3 `bending:victim` (attachment on any `LivingEntity`, synced, not saved)

`kind: byte` (NONE, STAGGER, KNOCKDOWN, STUCK_SAND, STUCK_ICE, LIFTED, TUMBLE, CLUTCH, SPIN, SOAKED, CHILLED, PUPPET,
PARALYSED, SEVERED), `until: long`, `caster: int` (entity id, for the puppet's claw-follow), `param: float`. Written by
`Victims.mark(entity, kind, ticks, caster)`; cleared by TTL in `BenderTicker`'s entity tick (only entities with the
attachment are ticked: a small set kept in `Victims`).

### 5.4 Config files

`config/bending-server.toml` (overridden by `<world>/serverconfig/bending-server.toml`; synced to clients in the
configuration phase by the copied `ConfigSyncPayload`) with sections: `[bending]` (general: `allowWithEnhancedPower`,
`pushPlayersWithoutPvp`, `pressExhaustion`, `holdExhaustionPerSecond`, `effectRadius` 64), `[progression]` (points per
mob list `pointsPerMob = ["minecraft:zombie=1", …]`, `pointsFormulaDivisor` 20, `pointsCap` 50, `assistWindowTicks`,
`freePoints` 3, `masteryXp` list, `masteryBuyCap` 7, `masteryBuyCost` list, unlock costs per ability `unlockCost.<id>`,
gates `gate.<id>`), `[matchups]` (every factor of 1.7), `[limits]` (every cap of 1.8 and `ANIMATION.md` 13), `[earth]`,
`[air]`, `[water]`, `[blood]`, `[fire]`, `[lightning]` (every number of sections 2–5 of `DESIGN.md`, keys lowerCamelCase as
Enhanced: `rockDamage`, `rockCooldown`, `wallSeconds`, `armourShellHealth`, …), `[extras]` (the suggestions' switches, all
`false` until built), `[music]`. `config/bending-client.toml`: `UI.md` 2.3.

## 6. Temporary blocks: `TempBlocks`

```
TempBlocks.place(ServerLevel, ServerPlayer owner, BlockPos, BlockState placed, int ticks, Kind kind) -> boolean
TempBlocks.placeAll(level, owner, Map<BlockPos, BlockState>, ticks, kind) -> int placed   // walls: all-or-nothing by the half rule
TempBlocks.revert(level, pos) / revertAll(level, owner, kind) / revertOldest(level)
TempBlocks.isTemp(level, pos) -> Optional<TempBlock>
TempBlocks.tick(level)   // LevelTickEvent.Post: reverts due entries; waits up to 200 ticks while a player stands inside, then pushes them up
TempBlocks.onBroken(level, pos)  // BlockEvent.BreakEvent / a changed state: the ledger entry is dropped (the original is NOT restored: whoever broke it gets nothing: temp blocks have no drops: `WreckedBlock`/`SteelPlate`/`IceSpike`/`Quicksand`/`TempLava` loot tables are empty, and a temp **earth** block (a real stone/dirt state) is placed with the ledger flag `noDrops`, which `LivingDropsEvent`… no: `BlockDropsEvent` cancels drops for ledgered positions)
```
Rules: a cell is placed only if `BendingRights.mayChange(owner, level, pos)` (section 7); never over a block entity,
bedrock, portal frames, end portal, reinforced deepslate, command blocks, `bending:unbendable`, or another mod's block
unless it is in the tags; the **original state and block-entity NBT** are recorded (`original`, `originalNbt`) and
restored exactly (`setBlock(pos, original, UPDATE_ALL | UPDATE_SUPPRESS_DROPS)`, NBT reloaded); **flags** on placement
`UPDATE_CLIENTS | UPDATE_KNOWN_SHAPE | UPDATE_SUPPRESS_DROPS` (Enhanced's `ArenaLoans.FLAGS`) so pistons, water and
redstone around are not triggered by the temp block beyond the visual update (no neighbour updates: a temp wall does
not break a redstone line; it also means a temp wall does not update observers: fine). **Fluids:** a temp block placed
in water records the water and restores it; a temp lava cell adjacent to water never makes stone (temp lava is not a
fluid). **Budgets**: per player and global caps (`[limits]`); exceeding reverts the oldest of that player. **Journal**:
every entry is mirrored into `WorldSave.tempBlocks` on placement and removed on revert (dirty flag; vanilla saves it
with the level), and `ServerStarted`/first level tick reverts every entry still in the journal (crash recovery). **A
temp block hit by an Enhanced power**: Enhanced breaks blocks through the loader's break event or `destroyBlock`; both
reach `onBroken` (`BlockEvent.BreakEvent` for the former; a `LevelTickEvent` check of `level.getBlockState(pos) !=
placed` for the latter, every 20 ticks per entry), which drops the ledger entry: the hole stays as Enhanced left it (a
wall destroyed by Thor is destroyed; what was under it is gone too: accepted, documented).

## 7. `BendingRights` (Enhanced's `BlockRights`, adapted)

```
mayChange(ServerPlayer, ServerLevel, BlockPos)      // for temp blocks: vanillaRights && !protectedByOthers && postBreak-equivalent (BlockEvent.BreakEvent of the current state) && postPlace-equivalent (EntityPlaceEvent)
mayHurtBlock(...)                                    // for fires: + mobGriefing + [fire] setsFires
vanillaRights(player, level, pos)                    // mayBuild, mayInteract (spawn protection, border), blockActionRestricted
protectedByOthers(player, level, pos)                // EnhancedHooks (Hex wall, boss arenas, Asgard, Mirror Dimension) if present
dimensionAllows(level, kind)                         // the 1.9 rules
```
Every ability that touches a block calls `TempBlocks.place` (which asks `mayChange`); fires call `mayHurtBlock`.

## 8. The element plug-in shape (Enhanced's `HeroServer`/`HeroClient`, adapted)

**Server `ElementServer`** (one per element, registered in `ElementServers`): `element()`, `onKey(player, data, slot,
down, sneak)` (the router: `Loadout.abilityIn(slot)` → the ability's `press/hold/toggle` handler; Sneak selects the
variant), `tick(player, data)`, `everyPlayerTick`, `applyBody` (passives, attributes: speed, armour, flight), `endAll`
(element removed/severed), `stopHold(data)`, `onLogin/Logout/Death/Respawn/ChangeDimension/ServerStopped`, plus
`onHit(attacker, target, event)` (for Fire Punch, Octopus, Water Arms: `LivingDamageEvent.Pre`), `onHurt(victim, event)`
(armour shells, shields, redirection, passives). Abilities are **objects**: `interface AbilityLogic { void press(ctx);
void release(ctx); void tick(ctx); void end(ctx); }` keyed by the `Ability` enum, with `AbilityContext(player, data,
level, now, sneak, mastery, config)`; every number read from config through `ctx.config().get(...)` at use (never
cached: Enhanced's rule). **Cooldowns** per slot in `BenderData.cooldownUntil[slot]` and `sneakCooldownUntil[slot]`
(the Sneak variant shares the slot but has its own timer so the HUD can show both); `Cooldowns.start(data, slot, sneak,
ticks)`; an ability that did nothing starts no cooldown (so no gesture lies: Enhanced's `AnimationSignalTests` rule).

**Client `ElementClient`** = Enhanced's `HeroClient` methods (`hudStatus`, `shieldStyle`, `eyes`, `stanceTargets`, `edges`,
`takeoffGesture`, `lean`, `drop`, `ambient`, `aimed`, `predictedGesture(s, data, slot, now, sneaking)`, `registerGestures`,
`cooldownGesture(slot, sneak, left)`, `flagGesture`, `liveGesture`, `grabGesture`, `flingGesture`, `startedElsewhere`,
`gestureStarted`, `firstPerson`, `flightParticles`, `flightRibbons`, `tick`, `clear`) plus `onFx(payload)`; registered in
`ElementClients`.

**`AbilityHandler.onKey`**: as Enhanced's: drops presses of a player who cannot act (dead, severed, paralysed, bloodbent,
meditating body), always passes releases; calls `ElementServers.of(data.element()).onKey`.

## 9. Server/client split and sync, per effect kind

| Effect kind | Server | Wire | Client |
|---|---|---|---|
| Player body state (toggles, holds, cooldowns, charges, held ids) | `BenderData` fields set by abilities | the synced attachment (every change batched into one sync per tick by `Benders.syncIfDirty`) | gestures from edges (`GestureRoutes`), stances from `Live`/`Toggle`, HUD |
| One-shot world FX (impacts, bursts, rings, stomps, forming, tells) | `ElementFx.send(level, near, kind, id, a, b, c, d)` | `<Element>FxPayload(byte kind, int id, Vec3 a, Vec3 b, int c, int d)` to players within `effectRadius` 64 (`Net.sendToPlayersNear`) | `<Element>Fx.handle` → a deterministic client simulation from `age` (Enhanced's `SandFx` pattern); gestures via `startedElsewhere` kinds |
| Continuing world objects with position (walls, pillars, quicksand, lava path, ice disc) | temp blocks (the world itself syncs them) + one FX payload at start for the dust/tell | blocks | the `BendingWorldFx` draws decorations (dust, collars, shimmer) from a client-side ledger fed by the payload and pruned when the block changes |
| Moving entities (rocks, blades, knives, balls, spouts, tornado, spirit, shots) | entities with `SynchedEntityData` for what the renderer needs (state, size, phase, rider) | vanilla entity sync | entity renderers; the tornado's funnel position is interpolated from 4-tick payloads (as Enhanced), the rest from entity positions |
| Beams (flamethrower, breath) | `ContinuousBeam` (copied) | `BeamPayload(shooter, style, range, on)` on start/stop and every 8 ticks | `BeamRenderer` with the flame drawer; the client raycasts the end itself |
| Holds with a victim (blood, suffocation, metal grip, puppet) | `Live` bit on the caster + `bending:victim` on the target | both synced attachments | threads drawn between caster and victim; `VictimPoses`; the victim's input lock from its own attachment |
| Shields (air shield, armour shell health) | `BenderData.shieldStrength` / `armourHealth` | the attachment | `ShieldStyle` dome / the armour layer's cracks; hit ripples via FX payload kinds `SHIELD_HIT`/`ARMOUR_HIT` |
| Lightning | the bolt is instantaneous on the server (raycast, damage) | `FireFxPayload BOLT(from, to, c=branches seed, d=flags)` to 64 blocks; `THUNDER` to 24–64 | `LightningFx` 8-tick bolt, flash, thunder delayed by distance |
| Spirit | `SpiritEntity` owned by the player; the server tells the client to possess it | `SpiritPayload(POSSESS/RELEASE, entityId)` | `SpiritCamera` sets `mc.setCameraEntity(spirit)`; input steers the spirit (the client sends vanilla movement for the spirit as a "vehicle-like" controlled entity: the spirit is a `PlayerRideableJumping`-free `Entity` whose motion is sent through a `SpiritMovePayload(Vec3 wanted)` 20/s, the server moves it within range and validates speed) |
| Rides (scooter, spout) | vehicles | vanilla passenger sync + input via `ServerboundPlayerInputPacket` (vanilla sends the rider's input for vehicles) | vehicle controlled client-side as a boat is (the entity's `controlVehicle`) |
| Screens | `ElementScreenPayload`, `BendingActionPayload` (serverbound: UNLOCK/BUY_MASTERY/LOADOUT/PRESET/LOOK/CHOOSE), results through the attachment | | |
| Sounds | `level.playSound` (vanilla sync) for one-shots; loops client-side from `Live`/`Toggle` predicates (`PowerLoopSounds`) and entity-bound loops | vanilla | |
| Config | the server's TOML text in the configuration phase (`ConfigSyncPayload`) | | the client reads the same values (prediction uses them) |

**Prediction**: the client plays a gesture at once on a key press when its own `BenderData` and the synced config say the
ability will work (`predictedGesture`), swallowing the server's edge within 12 ticks (copied `GestureTable` logic); refused
uses play out harmlessly.

## 10. Network: `ModNetwork`

One required channel `PROTOCOL_VERSION = "1"` (bumped once per release: "2" in 1.1, "3" in 1.2, "4" in 1.3: a 1.0 client
cannot join a 1.1 server), one optional music channel. Messages: serverbound `ability_key(int slot, boolean down,
boolean sneak)`, `bending_action(byte action, String id, List<String> slots, String name)`, `element_choice(Element)`,
`spirit_move(Vec3)`, `glider_dye(DyeColor)`; clientbound `element_screen()`, `earth_fx`, `air_fx`, `water_fx`, `fire_fx`
(the `(byte kind, int id, Vec3 a, Vec3 b, int c, int d)` record; kinds per element in ranges: Earth 1–99, Air 100–199,
Water 200–299, Fire 300–399, shared 400+), `beam(int shooter, String style, float range, boolean on)`, `spirit(byte op, int
id)`, `config_file(String name, String text)` (configuration phase), `hello` (configuration phase, Fabric only). Every
handler is **one static call into a client class** (Enhanced's rule, so the dedicated server never loads it).

## 11. Key bindings (`client/KeyBindings`)

`SLOTS[4]` = `Keys.newKey("key.bending.slot_" + n, KeyContext.IN_GAME, KEYSYM, {X, C, LEFT_ALT, K}[n], "key.categories.bending")`,
`MENU` = O. `ClientEvents.handleKeys` is Enhanced's (press-time Sneak capture, `consumeClick`, `SENT_DOWN`, prediction),
with `MENU` opening `BendingScreen` and Sneak + `MENU` sending `bending_action(LOOK_NEXT)`.

## 12. Commands

`/bending element get|set <players> <element|none>`, `points get|give|set <players> <n>`, `mastery get|set <player>
<ability> <level>`, `unlock <player> <ability|all>`, `loadout <player> <slot> <ability>`, `look <players> <look>`,
`zaheer open|reset <player>`, `sever <player> <seconds>|restore <player>`, `temp revert_all [radius]`, `debug
listener_order|anim|fx`, suggestions: `duel`, `dummy`. Operators only (level 2) except `/bending help`.

## 13. Enhanced compatibility (`compat/`)

`BendingCompat.enhanced()` is true when the mod list has `enhanced`; `EnhancedHooks` (loaded only then, in its own
class so the JVM never resolves Enhanced's types otherwise) asks: `Shelters.keepsOut(level, actor, from, to)` before an
ability's line crosses a Hex/ward/barrier (the ability stops there, as Enhanced's beams do), `BossFights.protects(level,
pos)` in `BendingRights`, `Metal.isMetalHero(player)` ← Bending **answers** it for Steel Armour wearers through a registered
predicate if Enhanced exposes one (today it is a seam inside Enhanced: a one-line `request(I)` to Enhanced to add
`Metal.addMetalRule(Predicate<Player>)`; until then, no interop), `Powers.power(player) != NONE` for `allowWithEnhancedPower`.
Enhanced never needs Bending.

## 14. Test strategy

### 14.1 GameTests (the copied adapter; `@BendingTest(template, batch, timeoutTicks)`; `TestKit` copied with `Fixture`,
`player(x,y,z, Element)`, `press(player, slot, sneak)` **added** as a kit helper (`AbilityHandler.onKey` down+up with the
Sneak flag), `hold(player, slot, ticks)`, `tickBender(player)` (`player.doTick()` + `BenderTicker.tick`), `config`,
`pvp`, `everyTick`, `heardSounds`, `sentPackets`, `particlesIn`, `tempBlocksOf(player)`, `moon(level, phase, night)`)

Shared classes: `SkeletonTests` (the ability list and order, `PROTOCOL_VERSION`, the plug-ins, every ability has an
icon/lang/damage type, no lang sentinels, `Live` bits fit), `DataTests` (both lang files equal keys and placeholders,
every sound has a file and subtitle, every look a 64×64 skin, every item/block/effect/damage type named), `SoundAssetTests`,
`AnimationMathTests` (+ the new leg-z/roll rules), `AnimationSignalTests` (a move with nothing to act on starts no
cooldown; a working move starts it and raises its `Live`), `NetworkTests` (hello, caps), `PlatformSeamTests`, `ConfigFileTests`,
`TempBlockTests` (place/revert/journal recovery/caps/rights/fluids/block entities refused/no drops/standing player
pushed up), `RightsTests` (spawn protection, border, Adventure, a listener cancelling the break event, dimension rules),
`ProgressionTests` (SP per mob, assist window, no SP from players, formula for unknown mobs, unlock gates, mastery XP
curve, buying caps, ES), `LoadoutTests`, `MatchupTests` (each factor), `AreaTargetsTests`, `VictimTests` (attachment TTL).

Per ability (one class per element lane; each ability at least these, in the lane's batch `earth`, `earth_blocks`,
`earth_pvp`…; state-heavy ones in a batch of their own with `bossFixture`-style `fixture` that clears hazards):

| Ability | Tests (names say what they check) |
|---|---|
| Rock Throw | `aRockLiftsFromEarthAndHurtsAZombie`, `noEarthMeansNoRockAndNoCooldown`, `masteryFourThrowsTwoRocks`, `masteryTenBoulderOnDoubleTap`, `theHoleRevertsAfterThreeSeconds`, `holdRockFollowsTheLookAndThrows`, `holdRockSetsDownWithoutDrops`, `aRockShattersOnAnEarthWall`, `aRockNeverBreaksGlass`, `pvpOffSparesPlayers`, `aClaimRefusesTheLift` (an `EventHub` `BreakEvent` listener cancels), `underWaterIsRefused`, `aBossTakesDamageAndLittleKnockback` |
| Earth Armour | `formsOnEarthInTwentyTicksAndLocksInput`, `theShellTakesHitsFirst`, `speedIsReduced`, `crumblesUnderWater`, `shedBlastHurtsAndPushes`, `lightningHalvedWaterTimesOnePointFive`, `masteryEightSeismicSenseGlows` (the glow predicate), `masteryTenReformsOnce`, `cooldownAfterBreak`, `wardenSonicBoomBypasses` |
| Earth Pillar | `risesUnderTheFeetAndCarriesThePlayer`, `stopsUnderACeiling`, `sinksAfterFifteenSeconds`, `aimedPillarAtMasteryFive`, `launchThrowsUpAndSparesTheFall`, `notOnEarthRefused`, `claimRefused` |
| Earth Wall | `risesInFourTicksThreeByTwo`, `sevenByFourAtMasteryTen`, `stopsArrowsAndRocks`, `sinksAfterTenSeconds`, `aPlayerInTheFootprintIsPushedOut`, `halfRefusedMeansNoWall`, `wallPushSlidesAndHurts`, `aNewWallReplacesTheOld`, `blockEntitiesAreNeverReplaced`, `waterUnderItIsRestored` |
| Sand Bending | `sandStreamBlindsAndHurts`, `needsSandOrInventorySand`, `sandWallLastsSixSeconds`, `waterEndsASandWall`, `soulSandCountsInTheNether` |
| Quicksand | `sinksAZombieForFourSeconds`, `failsOnAStrongerEarthbender`, `tieFavoursTheDefender`, `neverABoss`, `patchRevertsAndFrees`, `pillarEscapes`, `sandCloudBlinds` |
| Lava Bending | `pathRunsThroughEarthOnly`, `stopsAtWaterAndClaims`, `theBenderWalksTheirPathUnharmed`, `pathCoolsAndReverts`, `shurikenHurtsAndBurns`, `twoDiscsAtMasteryEight`, `passiveResistanceByMastery`, `lavaImmunityTenSecondsAtTen`, `itemsNeverBurn`, `netherFreeBesideLava` |
| Steel Knives / Metal Pull | `threeKnivesAtOneSevenAtTen`, `knivesCostChargeAndReturn`, `pullFillsChargeFromOresWithoutRemovingThem`, `pullTakesIngots`, `noChargeNoKnivesNoCooldown`, `knivesStickAndRecall`, `earthWallStopsKnives` |
| Steel Wall / Steel Armour | `plateWallFromCharge`, `chargeReturnsWhenItEnds`, `explosionBreaksAPlate`, `armourConductsLightning`, `armourAndEarthArmourExclude`, `masteryTenDoorOpensForTheOwner` |
| Wreck Metal / Metal Grip | `aHopperIsWreckedAndPassesNothing`, `aRailDerails`, `contentsSurviveTheWreckAndMend`, `mendsAfterSixtySeconds`, `theTagDecidesTheList`, `threeByThreeAtFive`, `gripLiftsAnArmouredZombie`, `gripRefusesUnarmouredPlayers`, `gripRefusesBosses`, `sheddingArmourFrees`, `flingHurts` |
| Air Blast | `pushesInAConeByMastery`, `pushesPlayersWithoutPvpWhenAllowed`, `deflectsArrowsAndFireballs`, `extinguishesFires`, `airJumpLaunchesAndSparesTheFall`, `blastWhileGlidingLiftsAtMasterySix`, `halfRangeUnderWater`, `bossesBarelyMove` |
| Air Shield | `flingsProjectilesAndPaysStrength`, `meleeGoesThroughAtThreeQuarters`, `breaksAndBurstsForThreeAndAHalfHearts`, `theBurstOptionIsFour`, `loweringNeverBursts`, `manualBurstAddsLeftover`, `refusedInWater`, `tornadoCancelsIt`, `masteryTenDeflectsLightningOnce` |
| Air Glider | `onlyAnAirbenderOpensIt`, `glidesSlowerThanElytra`, `dyeRecolours`, `closesInWater`, `blastDownLaunches` |
| Air Blades | `twoAndAHalfHearts`, `passesOneCreature`, `cutsPlantsWhereAllowed`, `twoBladesAtFourThreeAtEight`, `wallsStopIt`, `airSliceSweeps` |
| Spiritual Projection | `theBodySitsAndTheSpiritFlies`, `rangeByMastery`, `damageToTheBodySnapsBack`, `durationEnds`, `cannotCrossDimensions`, `theSpiritCannotInteract`, `logoutEnds`, `deepMeditationHeals`, `masteryTenShieldsTheBody` |
| Air Bubble | `breathesUnderWater`, `coversAFriendAtSix`, `cushionRemovesFallDamage`, `popsUnderASpout` |
| Air Scooter | `mountsAndMoves`, `speedByMastery`, `climbsOneBlockSteps`, `crossesWater`, `wallRideAtSix`, `passengerAtEight`, `burstAtTenOnDismount`, `waterShotKnocksOff` |
| Tornado | `liftsAndHurts`, `twoAtEight`, `steeringAtFive`, `wallsStopIt`, `bossesNeverLifted`, `liftsNoBlocks`, `itemsSurvive`, `whirlpoolUnderWater` |
| Zaheer's Path | `aPetDeathOpensIt`, `onlyNearbyWithTheOption`, `flightWorks`, `suffocationHurtsUntilDistance`, `breathlessMobsImmune`, `damageBreaksTheHold`, `airVacuumSnuffsFire`, `pushLessHitHarder`, `resetCommand` |
| Water Shot | `needsASource`, `theSkinIsUsedWhenNoWater`, `plantsAtMasterySevenDry`, `oneAndAHalfHearts`, `timesOnePointFiveOnFirebenders`, `extinguishesAndWets`, `whipLashesAndPulls`, `twoStreamsAtFour`, `lobAtTen`, `rainIsASource`, `netherSkinOnly` |
| Healing | `healsAFriendByMastery`, `healsSelfWithNoTarget`, `movementBreaksIt`, `removesPoisonSlowly`, `cleanseAtOnce`, `neverHostiles`, `fullHealthCostsNothing` |
| Water Octopus | `formsWithThreeCharges`, `slapsProjectiles`, `extraArmsOnHits`, `lashPulls`, `flamethrowerEvaporatesArms`, `freezeStiffens`, `lightningTimesOnePointFive`, `dropsWithoutASource` |
| Water Spout | `tinyAtOneTallAtTen`, `ramHurts`, `overLandItShrinks`, `lightningCollapsesIt`, `freezeAtEight`, `waveRideCarries`, `needsWaterNotSkinBelowFive` |
| Water Arms | `reachAndDamageByMastery`, `noItemsInHand`, `grabPulls`, `climbAtFour`, `flamethrowerDropsThem`, `freezeStiffens` |
| Ice | `knivesSlowAndShatter`, `fireMeltsKnivesInFlight`, `spikesEruptAndMelt`, `freezeMakesABridgeThatMelts`, `freezeRootsWetFeet`, `freezeStiffensConstructs`, `snowVolley`, `iceWallAtTheMidpoint`, `fireMeltsFourBlocksPerBall`, `iceDomeCloses`, `netherHalfLife` |
| Blood Bending | `onlyOnAFullMoonBelowTen`, `atTenAnyNight`, `noMoonInTheNetherBelowTen`, `failsOnAStrongerWaterbender`, `puppetMovesWithTheLook`, `holdTimeByMastery`, `flingHurts`, `lightningBreaksTheHold`, `twoHeartsBreakTheHold`, `undeadImmune`, `bossesImmune`, `twoTargetsAtTen`, `masteryXpTimesFour` |
| Bending Severance | `needsMasteryTen`, `upClose`, `durationByStrength`, `theSeveredCannotBend`, `survivesLogout`, `healingRestoresEarlyWhenAllowed`, `expires` |
| Fireball | `twoHeartsAndBurning`, `sizeByMastery`, `twoAtFiveThreeAtEight`, `greatFireballAtTen`, `smallFireballFromSix`, `waterBurstsIt`, `refusedInWater`, `rainReduces`, `netherBoosts`, `fireMobsImmune`, `noFiresWithoutTheOption` |
| Fire Punch | `fistsAddDamageAndBurn`, `ignoresSomeArmour`, `comboAtFour`, `jabAtSeven`, `kickArcs`, `noItemsWhileOn` |
| Flamethrower | `twoJetsHurtEveryHalfSecond`, `rangeByMastery`, `meltsIce`, `breathAtSevenLocks`, `waterShotPutsItOut`, `vacuumKills`, `exhaustion` |
| Fire Whip | `strokesEveryTwelveTicksAndIgnite`, `lengthByMastery`, `blockFiresOnlyWhereAllowed`, `cutsARockAtSeven`, `wrapsAtTen`, `lashCracks`, `waterWhipCancels` |
| Fire Circle | `ringExpandsAndHurts`, `legsFromSix`, `waterCutsAGap`, `meltsIce`, `jetLiftsAndSteers`, `jetBreaksAGrip`, `doubleAtTen` |
| Fire Healing | `slowerThanWater`, `poisonTicksSlower`, `burnOutCleansesAndHurts` |
| Lightning | `chargeLocksAndFires`, `fourAndAHalfHeartsAndParalysis`, `dualAtTen`, `wetAndMetalTimesOnePointFive`, `earthArmourHalves`, `rodsAttract`, `inWaterShocksTheCaster`, `chargedCreeper`, `noVanillaBoltNoFireNoWitch`, `arcJab`, `cancelOnRelease` |
| Lightning Redirection | `chanceByMastery` (seeded RNG), `successReturnsFullDamageAtTheLook`, `failureHurtsAndDrops`, `windowTenTicks`, `autoFireAtEight`, `groundItHeals`, `steelArmourAlwaysFails`, `vanillaBoltRedirectable`, `twoBoltsAtTen` |

Cross-element: `MatchupTests` (every row of 1.7), `CrossElementTests` (a rock vs an air shield, a tornado vs a spout, a
fire circle under a tornado, water on a lava path, lightning on a steel wall, blood vs blood), `CoexistenceTests` (two
benders' caps are independent; four benders' entity and temp-block counts stay under the limits in a scripted fight).

Enhanced's rules carry over: one batch per lane; a test that changes a server setting (`f.config`, `f.pvp`, the moon)
in a batch of its own; keep flying things inside the 21-block arena; `fixture` clears every static (`Hazards.clear`,
`TempBlocks.clear`, `Victims.clear`, `ContinuousBeam.clear`, `ElementServers` statics); delete `run/world` and
`run/config/bending-server.toml` before a full run; the success line `All N required tests passed`; CI 45 min.

### 14.2 The real-client check list (manual, per lane, Enhanced's procedure)

A dedicated server + **two clients** (one English, one Hebrew), the second client **watching** what others see, **3+
render rounds** per lane, screenshots named `<lane>/<ability>-<view>-<lang>.png` kept in `docs/screenshots/qa/` (not
shipped), with F2:

1. Each ability, third person from the second client: wind-up, release, recovery frozen with `/bending_anim target <p>`
   + `play <gesture> <tick>` at 3 ticks (start, peak, end); walking during the cast; the Sneak variant.
2. First person: the hands at the same three ticks; the overlay (slit, spirit, blood, paralysis, bubble, heat, redirect
   path); camera shake and FOV with and without *Reduced motion*.
3. Effects at 1, 8, 24 and 48 blocks; with *Effect density* 0.25; vanilla Particles "Minimal"; at night and in rain.
4. Idle stance (key held), the ambient master effect (set ES with `/bending mastery`), the run character, landing.
5. Looks: every look front/back/side, with the armour/octopus/glider over it, the look picker's faces.
6. Screens: every tab in both languages, narrow window (854×480) and wide; the Element Choice with 1–4 keys; Hebrew
   right-to-left checks (key names, numbers, units).
7. HUD: every status line, the Sneak swap, cooldown sweeps, the water gauge in each source, the moon on a full-moon
   night (`/time set 18000` on phase 0: `/bending debug moon 0`), spirit mode, severed, paralysed; with Enhanced
   installed too (a player with a power and an element: both panels, no overlap).
8. Sounds: every event audible at 1, 8, 24 blocks; loops start/stop/follow; the music themes and stems; subtitles in both
   languages.
9. Cross-loader: the same checks with the **Fabric client on the NeoForge server and the NeoForge client on the Fabric
   server**; a world saved on one loader opened on the other keeps every bender (the `SavedData`).
10. Four-bender fight for 2 minutes: F3 frame time and entity/particle counts (the budget of `ANIMATION.md` 13).

### 14.3 CI

`.github/workflows/build.yml` as Enhanced's: `./gradlew build` (runs `portLint` for both loaders), `:neoforge:runGameTestServer`,
`:fabric:runGameTest`, `tools/parity.py build/gametest/*.json` (fails on a loader-only failure), upload the universal jar,
release on `main`. `wiki.yml` as Enhanced's for `docs/wiki/`.
