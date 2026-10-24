# Bending — ארכיטקטורה טכנית

מזהה המוד **`bending`**, קבוצה `com.kolodkingal.bending`, Minecraft **1.21.1**, **קובץ JAR אוניברסלי אחד ל-NeoForge 21.1 ול-
Fabric** (Fabric API מותר רק ב-Fabric), Java 21, בלי ספריות זמן ריצה, כל מודל הוא `ModelPart`, וכל אנימציה מונעת
מקוד. נבנה על היסודות של Enhanced: המסמך הזה אומר, מערכת אחרי מערכת, מה **מועתק** (אותו קוד, חבילה חדשה),
מה **מותאם** (אותה צורה, תוכן ייעודי לאלמנט) ומה **חדש**.

## 0. נקודת ההתחלה הכנה: מה יש היום ב-Enhanced

ב-`main` של Enhanced (3.9.0, קומיט 888609c) יש את **שכבת הפלטפורמה הניטרלית ללואדר** של שלב ההעברה P0 שלו: `platform/`
(`Platform`, `Net`, `Attach`, `Registrars`, `EventHub` + מעבד `@Subscribe`, `ClientRegistrar`, `ClientServices`,
`Keys`, `LoaderHooks`, `ItemAccess`, `Parts`, כ-80 מחלקות האירועים המשותפות), **מימוש NeoForge** שלם
(`platform/neoforge/*`, הגשר `GameBindings`/`ClientBindings`, `NeoTestHooks`), משימת Gradle בשם `portLint` שאוסרת
שימוש ב-API של NeoForge מחוץ ל-`platform/neoforge/**`, מנגנון הגדרות TOML משלו של המוד, וה**לחיצת יד** של Fabric
(`HelloPayload`). אין בו **מימוש Fabric, אין בניית Fabric Loom, אין קובץ JAR אוניברסלי**, באף ענף; ה-"PLAN2" שהקוד
מצטט לא נמצא במאגר, ושום דבר לא מתעד איך מוסיפים גרסאות Minecraft מאוחרות יותר, מעבר לשמות מתאמי `r1` ולדוח
השוויון (parity) לכל `<mc>-<loader>`.

אז כדי שקובץ ה-JAR האוניברסלי של Bending יתקיים צריך **לבנות את חצי ה-Fabric** (סעיף 2.3), וזה נעשה או בנתיב P1 של
Enhanced עצמו קודם (ו-Bending מעתיק אותו), או ב-**שלד P0** של Bending (ו-Enhanced מעתיק אותו בחזרה). `PLAN.md` מציב אותו
ב-**גרסה 1.0** של Bending, נתיב **L0-FABRIC**, כשקובץ JAR של NeoForge בלבד הוא אבן הדרך החלופית, כדי שאדמה לעולם לא
תיחסם בגללו. **שאלה פתוחה ליובל:** לשתף את PLAN2 (ואת `SCRATCH/port`) אם הם קיימים; התוכנית שלמטה נכתבה כך שתתאים
לשכבת הפלטפורמה כפי שהיא עכשיו.

## 1. דו-קיום עם Enhanced (שני המודים באותו משחק)

| דבר | Enhanced | Bending | כלל |
|---|---|---|---|
| מזהה מוד / מרחב שמות | `enhanced` | `bending` | כל מזהה, מרקם, צליל, תג, מפתח שפה, הישג וסוג נזק הוא `bending:*` |
| חבילת Java | `com.kolodkingal.enhanced` | `com.kolodkingal.bending` | אין שם מחלקה משותף ב-classpath; שכבת הפלטפורמה היא **עותק**, לא תלות (שני עותקים של `EventHub` חיים יחד: כל מוד רושם מאזינים משלו ב-hubs משלו; כל אחד מגשרי NeoForge מוסיף מאזינים מקוריים משלו) |
| מיקסינים | `enhanced.mixins.json`, חבילה `…enhanced.mixin` | `bending.mixins.json`, חבילה `…bending.mixin` | אותן נקודות הזרקה ב-`PlayerModel.setupAnim` וב-`PlayerRenderer.setupRotations` הן **מצטברות** (`@Inject` ב-HEAD/TAIL, בלי ביטולים), ולכן שני המנגנונים רצים; ה-`HeroPoses.apply` של Bending רץ אחרי זה של Enhanced ונוגע רק בשחקן ש**יש לו אלמנט**, וזה של Enhanced רק בשחקן עם כוח; שחקן עם שניהם מקבל את התנוחה של Enhanced ועליה את זו של Bending (מקובל; האפשרות `bending-client [animation] yieldToEnhanced` על true גורמת ל-Bending לדלג על שחקן שיש לו תנועה פעילה של Enhanced) |
| ערוץ הודעות (payload) | `enhanced:*`, גרסה "14" | `bending:*`, גרסה "1" (`ModNetwork.PROTOCOL_VERSION`), ערוץ אופציונלי `bending:music` "1" | מזהי `CustomPacketPayload.Type` נפרדים; לכל מוד לחיצת יד משלו |
| נתוני צירוף (attachments) | `enhanced:power`, … | `bending:bender`, `bending:victim`, `bending:temp_blocks`, `bending:world` | מפתחות נפרדים; נשמרים זה לצד זה על השחקן ועל העולם (level) |
| קובצי הגדרות | `enhanced-server.toml`, `enhanced-client.toml` | `bending-server.toml`, `bending-client.toml` (+ `<world>/serverconfig/bending-server.toml`) | מנגנון `Configs` המועתק כותב את הקבצים שלו; מסך ההגדרות מציג את שלו |
| מקשי ברירת מחדל | R G V B N H Y Z J (+ צירופי Ctrl, Alt/Del במצב בנייה) | **X C Left-Alt K**, **O** | אין התנגשות בברירות המחדל (1.5 ב-`DESIGN.md`); הקשרים `IN_GAME` |
| HUD | לוח למעלה משמאל | לוח למטה משמאל (פינה ניתנת להגדרה) | אין חפיפה כברירת מחדל |
| לשונית יצירתית | `enhanced` | `bending` | |
| רישום צלילים | `enhanced:*` | `bending:*` | |
| חיבורים רכים | — | `BendingCompat.enhanced()` (סעיף 13) | Bending שואל את `Shelters`/`BossFights`/`Metal.isMetalHero` של Enhanced דרך מחלקות אופציונליות בלי רפלקציה, שנטענות רק אם `ModList`/`FabricLoader` אומרים ש-Enhanced קיים |

## 2. פלטפורמה ובנייה

### 2.1 מועתק מ-Enhanced כמות שהוא (`com.kolodkingal.bending.platform.*`)

`Platform`, `Loader`, `LoaderHooks`, `ItemAccess`, `Parts`, `EntityAttributesCompat`, `CommonTags`; `net/Net`,
`NetService`, `PacketContext`; `attach/Attach`, `AttachKey`, `AttachService`, `SyncTo`; `reg/Registrars`, `Registrar`,
`RegEntry`, `ItemRegistrar`/`ItemEntry`, `BlockRegistrar`/`BlockEntry`, `ComponentRegistrar`, `RegService`; `ext/ItemExt`,
`EntityExt`, `BlockEntityExt`; `event/*` (כל אירוע משותף, `EventHub`, `Events`, `Subscribe`, `Subscriber`, `Priority`,
`Side`, `Cancellable`, `TriState`) ו-`event/client/*`; `client/ClientRegistrar`, `ClientServices`, `Keys`, `KeyContext`,
`KeyMod`, `HudHeights`, `BlockModels`; **כל `platform/neoforge/*`** (`NeoPlatform`, `NeoSetup`, `NeoNet`, `NeoAttach`,
`NeoPersistentData`, `NeoRegistrars`, `NeoHooks`, `NeoItemAccess`, `client/NeoClientRegistrar`, `client/NeoClientServices`,
`bridge/NeoForgeBridge`, `Binding`, `GameBindings`, `ClientBindings`, `ClientConvert`, `ListenerOrderRecorder`,
`ListenerOrderCommand`); **מעבד ההערות** (`processor/` → `SubscribeProcessor` שמייצר
`GeneratedSubscriptions`, כש-`tools/listener-order.txt` נרשם פעם אחת מהמאזינים של Bending עצמו); **מנגנון
ההגדרות** `config/core/*` (`ConfigSpec`, `ConfigFile`, `Configs`, `ConfigWatcher`, `Toml`, `ConfigSyncPayload`) ו-
`config/neoforge/ConfigNeoForge`; **מתאם ה-GameTest** `platform/test/*` (`EnhancedTest` → `BendingTest`, `TestHooks`,
`TestIndex`, `TestFilter`, `ParityReporter`, `EnhancedTests` → `BendingTests`) ו-`platform/neoforge/test/*`; משימת
`portLint` (הכללים `neoforge`, `subscribe`, `hidden`, `impl`, ועוד כלל חדש **`fabric`**: `\bnet\.fabricmc\.` מחוץ ל-
`platform/fabric/**`); קובצי `META-INF/services` של שלושת ה-SPI. רק מחרוזות מזהה המוד ושתי נקודות הכניסה
של `@Mod` משתנות.

### 2.2 נקודות הכניסה

* NeoForge: `Bending` (`@Mod("bending")`: `NeoSetup.init(modBus)`, `NeoRegistrars.register(...)` לכל registrar לפי
  הסדר, `ModAttachments.init()`, `ModNetwork.register()`, `Configs.register(SERVER, BendingConfig.SPEC)`, `Configs.register(CLIENT,
  BendingClientConfig.SPEC)`, `ConfigNeoForge.install(modBus)`) ו-`client/BendingClient` (`@Mod(value="bending",
  dist=CLIENT)`: `ClientSetup.register(NeoClientRegistrar.install(container, modBus)); ClientRegistrar.get().configScreen(ConfigScreen::new)`).
* Fabric: `platform/fabric/BendingFabric implements ModInitializer` ו-`platform/fabric/client/BendingFabricClient
  implements ClientModInitializer`, שעושים אותו דבר דרך מימוש Fabric (2.3), ובנוסף `BendingFabricGameTest`
  למתאם המבחנים.
* `fabric.mod.json` (נקודות כניסה `main`, `client`, `fabric-gametest`; `depends: fabricloader ≥ 0.16, fabric-api *,
  minecraft 1.21.1`; `mixins: bending.mixins.json`) ו-`neoforge.mods.toml` (כמו התבנית של Enhanced) **שניהם** ב-
  `META-INF`/שורש של ה-JAR; כל לואדר מתעלם מהקובץ של השני.

### 2.3 חצי ה-Fabric (חדש; נתיב L0-FABRIC של 1.0)

| מחלקה (`platform/fabric/…`) | מממשת | איך |
|---|---|---|
| `FabricPlatform` | `Platform` | `loader()` FABRIC; `isPhysicalClient` דרך `FabricLoader.getEnvironmentType()`; `configDir` ‏`FabricLoader.getConfigDir()`; `modResource` דרך `ModContainer.findPath`; `currentServer` ממאזין של `ServerLifecycleEvents`; `fakePlayer` ‏`FakePlayer.get(level, profile)` (‏`net.fabricmc.fabric.api.entity.FakePlayer` של Fabric API); `hooks()`, `itemAccess()`, `parts()`, `attributes()` למטה |
| `FabricHooks` | `LoaderHooks` | `blockPlaceCancelled`: אין ב-Fabric אירוע הצבה: שולחים את `BlockEvent.EntityPlaceEvent` של המוד עצמו ב-hub **וגם** את הדפוס הנפוץ של `PlayerBlockBreakEvents`/מודי שטח מוגן (‏`UseBlockCallback` מיועד לאינטראקציה, לא להצבה): קוראים ל-`BlockEvent.EntityPlaceEvent` ב-hub בלבד, ושואלים מודי שטח מוגן שאינם תלויים ב-**Fabric Permissions API** דרך אירועי ה("הגנה") שלהם אם הם קיימים: מגבלה מתועדת: ב-Fabric, מודי שטח מוגן שנתלים על `PlayerBlockBreakEvents.BEFORE` מכובדים בשבירות (אנחנו שולחים אותו); בהצבות אנחנו שולחים רק את אירוע ה-hub. `blockBreakCancelled`/`blockBreakEventCancelled`: `PlayerBlockBreakEvents.BEFORE` + אירוע ה-hub. `projectileImpactCancelled`: אירוע ה-hub בלבד (ל-Fabric אין). `finalizeMobSpawn`: `mob.finalizeSpawn(...)` ישירות. `enderPearlTeleportCancelled`: hub בלבד. `isShield`: `stack.getItem() instanceof ShieldItem || FabricShieldTag`. `drownsInFluidAtEyes`: `entity.isEyeInFluid(FluidTags.WATER) && !canBreatheUnderwater` |
| `FabricItemAccess` | `ItemAccess` | ‏`ItemStorage.SIDED` של Fabric Transfer API עטוף לשיטות הדומות ל-item-handler |
| `FabricParts` | `Parts` | `EnderDragonPart` בלבד (אין ב-Fabric multipart גנרי) |
| `FabricAttributes` | `EntityAttributesCompat` | מהירות שחייה: אין ב-Fabric (מחזיר null ← הקוראים מדלגים); טיסה ביצירתי: המאפיין של המוד עצמו `bending:creative_flight` שנרשם ב-Fabric ומיושם דרך `Abilities.mayfly` במקבילה של `PowerTicker` (‏`BenderTicker.applyPassives` קובע את `mayfly` ישירות כשהמאפיין חסר) |
| `FabricNet` | `NetService` | ‏`PayloadTypeRegistry.playC2S()/playS2C().register(type, codec)` לכל הודעה, ו-`ServerPlayNetworking.registerGlobalReceiver` / `ClientPlayNetworking.registerGlobalReceiver` עם `Context` שעוטף את ההקשר של Fabric; לחיצת היד לגרסה: **`HelloPayload`** רשום ב**שלב ההגדרה (configuration)** (‏`PayloadTypeRegistry.configurationS2C/C2S`, `ServerConfigurationNetworking.registerGlobalReceiver`, ‏`ServerConfigurationNetworking.ConfigurationTask`): השרת שולח את `HelloPayload.local()` שלו, הלקוח עונה, ו-`HelloPayload.check` מחליט (בדיוק המילים ש-NeoForge מציג); ערוצים אופציונליים: `ServerPlayNetworking.canSend(player, type)` עבור `hasChannel`; השולחים דרך `ServerPlayNetworking.send` / `PlayerLookup.tracking/world/around` |
| `FabricAttach` | `AttachService` | **Attachment API** של Fabric API (`AttachmentRegistry.create(id, b -> b.initializer(def).persistent(codec).copyOnDeath().syncWith(syncCodec, AttachmentSyncPredicate.all()/targetOnly()))`): אותה סמנטיקה כמו של NeoForge (`SyncTo.SELF` ← `targetOnly()`); `persistentData` ← נתוני צירוף מסוג `CompoundTag` בשם `bending:persistent_data` |
| `FabricRegistrars` | `RegService` | ‏`Registry.register` ב-`onInitialize` לפי סדר ההצהרה; `entityAttributes` דרך `FabricDefaultAttributeRegistry.register`; `spawnEgg` דרך `SpawnEggItem` (‏Fabric מאפשר סוג נדחה דרך `FabricSpawnEggItem`… משתמשים ב-`SpawnEggItem` של vanilla כשהסוג רשום קודם: Bending רושם סוגי ישויות לפני פריטים) |
| `FabricPersistentData` | | מצב נתוני הצירוף בלבד |
| `client/FabricClientRegistrar` | `ClientRegistrar` | `EntityRendererRegistry.register`, `BlockEntityRendererFactories.register`, `EntityModelLayerRegistry.registerModelLayer`, `LivingEntityFeatureRendererRegistrationCallback.EVENT` (← `addLayers` עם `LayerContext` מעל `EntityRendererProvider.Context`), `ParticleFactoryRegistry`, `KeyBindingHelper.registerKeyBinding`, `HudRenderCallback` (← `guiLayer`; העוגנים ממופים לסדר ציור: ל-Fabric יש callback אחד, לכן השכבות מצוירות לפי סדר הרישום **אחרי** ה-HUD של vanilla; ‏`BELOW DEBUG_OVERLAY` ≈ אחרי הכול חוץ ממסך הדיבאג: מקובל), `ColorProviderRegistry.ITEM`, `BuiltinItemRendererRegistry`, `DimensionRenderingRegistry.registerDimensionEffects`, `ResourceManagerHelper.registerReloadListener`, `ClientCommandRegistrationCallback`, ‏`configScreen` ← נקודת כניסה של **ModMenu** אם קיימת (`modmenu` אופציונלי) |
| `client/FabricClientServices` | `ClientServices` | ‏`newKey` ← `KeyMapping` (ל-Fabric אין הקשרי התנגשות או מקשי שינוי: `KeyContext` נשמר בבדיקה של המוד עצמו ב-`ClientEvents.handleKeys`, ו-`KeyMod` נבדק עם `Screen.hasControlDown()` וכדומה); `matches` = `mapping.matches(key)` && ה-`isActive` של ההקשר; `hudLeftHeight` ← הספירה של המוד עצמו (ל-Fabric אין: Bending קורא את שדות `Gui` של vanilla דרך מיקסין accessor); `renderBatched` ← `BlockRenderDispatcher.renderBatched` |
| `bridge/FabricBridge` | גשר האירועים | האירועים של Fabric הם callbacks, לא אפיק עם עדיפויות: הגשר רושם **callback אחד של Fabric לכל אירוע מקושר** ששולח ל-hub **בכל העדיפויות לפי הסדר** (`EventHub.post(E)`); ביטול ממופה לערך ההחזרה של ה-callback (`InteractionResult.FAIL` / `false`); כ-40 קישורים לאירועים ש-Bending משתמש בהם (סעיף 2.5), כל אחד מתאם קטן: `ServerTickEvents`, `ServerLivingEntityEvents.ALLOW_DAMAGE/AFTER_DEATH`, `PlayerBlockBreakEvents`, `UseBlockCallback`, `UseItemCallback`, `UseEntityCallback`, `AttackEntityCallback`, `ServerPlayerEvents.AFTER_RESPAWN/COPY_FROM`, `ServerEntityEvents.ENTITY_LOAD/UNLOAD`, `ServerPlayConnectionEvents.JOIN/DISCONNECT`, `EntityElytraEvents`, `ServerLifecycleEvents`, בצד הלקוח: `ClientTickEvents`, `WorldRenderEvents.AFTER_PARTICLES/START`, `HudRenderCallback`, `ClientPlayConnectionEvents`, `ClientPreAttackCallback`; אירועים **בלי מקביל ב-Fabric** (שינויי כמות ב-`LivingIncomingDamageEvent` ← `ServerLivingEntityEvents.ALLOW_DAMAGE` + **מיקסין** על `LivingEntity.hurt` לכמות; `ComputeFovModifierEvent` ← מיקסין על `AbstractClientPlayer.getFieldOfViewModifier`; `ViewportEvent.ComputeCameraAngles` ← מיקסין על `Camera.setup`; `RenderHandEvent`/`RenderArmEvent` ← מיקסינים על `ItemInHandRenderer`; `MovementInputUpdateEvent` ← מיקסין על `KeyboardInput.tick`; `ProjectileImpactEvent` ← מיקסין על `Projectile.onHit`; `LivingKnockBackEvent` ← מיקסין על `LivingEntity.knockback`; `MobEffectEvent.Applicable` ← `LivingEntity.canBeAffected`) הם **מיקסינים של Fabric בלבד** ב-`bending.fabric.mixins.json` (`platform/fabric/mixin/`), וכל אחד שולח את האירוע המשותף. זה עיקר העבודה של L0-FABRIC. |
| `FabricTestHooks` | `TestHooks` | נקודת הכניסה `fabric-gametest`: מחלקת מחולל המסומנת ב-`FabricGameTest` ← `BendingTests.enhancedTests()`; ‏`mockPlayerWithConnection` כ-`GameTestHelper.makeMockServerPlayerInLevel` של Fabric ועוד `Connection` על `EmbeddedChannel` (המתכון של NeoForge, בלי תלות בלואדר) |

**מגבלות בצד Fabric שמתועדות במשחק** (שורה במסך ההגדרות של Fabric): מודי שטח מוגן מכובדים בשבירות
ובאינטראקציות, ובהצבות רק דרך האירוע של Bending עצמו; יכולת מהירות השחייה הפסיבית משתמשת במאפיין ש-Bending רושם.

### 2.4 קובץ ה-JAR האוניברסלי (חדש; L0-FABRIC עם אחראי הבנייה)

פרויקט Gradle **מרובה-מודולים**: `common/` (מתקמפל מול vanilla דרך **VanillaGradle** (או ModDevGradle של NeoForge ב-
"מצב vanilla", ‏`neoForge { version = … }` עם `neoFormVersion`), מיפויי Mojang, ‏`src/main/java` = הכול חוץ מ-
`platform/neoforge` ו-`platform/fabric`; המיקסינים; המעבד מופעל כאן), `neoforge/` (ModDevGradle 2.0.148,
תלוי ב-`common` כ-`compileOnly` + מקורות `platform/neoforge`, ‏`neoforge.mods.toml`, מריץ `runClient`, `runServer`,
`runGameTestServer`), `fabric/` (**Fabric Loom** 1.8+, ‏`mappings loom.officialMojangMappings()` (כך שהקוד הוא מיפוי אחד;
Loom ממפה מחדש את ה-JAR שנבנה ל-intermediary), תלוי ב-`common`, ב-Fabric API וב-`fabric.mod.json`, מריץ
`runClient`, `runServer`, `runGameTest`). **JAR אחד**: משימת `universal` **ממזגת** את שני קובצי ה-JAR של הלואדרים
עם הזזה (relocation) בסגנון **Forgix** (תוסף Gradle בזמן בנייה `io.github.pacifistmc.forgix`, ‏1.2.x; **לא** ספריית
זמן ריצה) שמפיקה את `build/libs/bending-<version>.jar` שמכיל את המחלקות של שני הלואדרים (הקוד המשותף מופיע
פעמיים: פעם בשמות Mojang עבור NeoForge, פעם ממופה ל-intermediary עבור Fabric, מוזז ל-`…bending.neoforge.*`
ול-`…bending.fabric.*`? **לא**: Forgix משאיר את החבילות ופותר את המחלקות המשותפות הכפולות על ידי **שמירת שני
העצים של הלואדרים תחת שורשים מוזזים משלהם וכתיבה מחדש של קובצי המטא-דאטה כך שיצביעו אליהם**; ה-JAR גדול בערך פי 2
מ-JAR של לואדר יחיד, וזה מקובל). ה-CI בונה את שלושתם ובודק את שני הלואדרים: `./gradlew :neoforge:runGameTestServer
:fabric:runGameTest` ומשווה את דוחות השוויון (parity) `build/gametest/1.21.1-neoforge.json` ו-`1.21.1-fabric.json`
(‏`ParityReporter` כבר נותן שמות לקבצים לפי לואדר; השוואה של `tools/parity.py` מפילה את הבנייה על מבחן שעובר רק
בלואדר אחד). **משחק משותף (Cross-play)**: שני הלואדרים מדברים באותן הודעות `bending:*` עם אותם codecs ואותו
`PROTOCOL_VERSION`; **שמירות עוברות** כי כל נתוני צירוף נשמרים תחת מזהה המוד עצמו עם ה-`Codec` של המוד עצמו בשני
הלואדרים (‏`AttachmentType` של NeoForge וה-Attachment API של Fabric שניהם שומרים תחת מזהה המוד ב-NBT של הישות/העולם:
NeoForge תחת `neoforge:attachments`, ו-Fabric תחת `fabric:attachments`: **לא אותו נתיב NBT**: עולם שיועבר
בין לואדרים **יאבד** נתוני צירוף). **הפתרון (חדש, S0 של 1.0): `SaveBridge`**: המוד כותב גם העתק משקף
של כל נתוני צירוף מתמידים ל**מכל הנתונים המתמידים של הישות/העולם תחת מפתח משלו**
(‏`bending:save` ← נתוני הצירוף מסוג `CompoundTag` בשם `bending:persistent_data`, שנשמר בעצמו על ידי מערכת
נתוני הצירוף של כל לואדר... מעגלי). **התכנון הנכון:** Bending שומר את **נתוני השחקן שלו באחסון של העולם עצמו**, לא
בנתוני צירוף: `SavedData` (‏`BendingSavedData` ב-`data/bending_players.dat` של העולם הרגיל (Overworld), עם מפתח UUID של השחקן,
מקודד ב-`Codec`) לכל מה שחייב לשרוד מעבר בין לואדרים (אלמנט, SP, פתיחות, רמת שליטה, סידורי קרב, מראה, זהיר,
טיימר הניתוק, תצורות מוכנות), ומשתמש ב**נתוני צירוף רק לתצוגת זמן הריצה המסונכרנת** (‏`bending:bender` מסונכרן, לא
נשמר, נבנה מחדש מהנתונים השמורים בכניסה/בהחייאה). `SavedData` הוא של vanilla וזהה בשני הלואדרים. נתוני
צד העולם (‏`bending:world`: מצב פעמוני הירח, טיימרי ניתוק של שחקנים לא מחוברים, יומן הבלוקים הזמניים) הם גם
`SavedData` (‏`data/bending_world.dat` לכל ממד). **זה פורמט השמירה של סעיף 5.**

### 2.5 אירועים ש-Bending משתמש בהם (מה שהגשרים חייבים לקשר)

צד השרת: `ServerTickEvent.Post`, `LevelTickEvent.Post`, `PlayerTickEvent.Post`, `EntityTickEvent`, `LivingIncomingDamageEvent`,
`LivingDamageEvent.Pre/Post`, `LivingDeathEvent`, `LivingKnockBackEvent`, `LivingFallEvent`, `LivingChangeTargetEvent`,
`MobEffectEvent.Applicable`, `EntityJoinLevelEvent`, `EntityLeaveLevelEvent`, `PlayerEvent.PlayerLoggedIn/Out/Respawn/Changed
Dimension/Clone`, `PlayerInteractEvent.RightClickItem/RightClickBlock/EntityInteract`, `AttackEntityEvent`, `BlockEvent.BreakEvent/
EntityPlaceEvent`, `ProjectileImpactEvent`, `EntityStruckByLightningEvent`, `EntityMountEvent`, `LivingEntityUseItemEvent`,
`ItemTossEvent`, `RegisterCommandsEvent`, `ServerStarting/Started/Stopping/StoppedEvent`, `OnDatapackSyncEvent`,
`LevelEvent.Load/Save/Unload`, `EntityMobGriefingEvent`, `ExplosionEvent.Detonate`. צד הלקוח: `ClientTickEvent.Post`,
`InputEvent.Key/MouseButton.Post`, `MovementInputUpdateEvent`, `RenderLevelStageEvent`, `RenderPlayerEvent.Pre/Post`,
`RenderHandEvent`, `RenderArmEvent`, `RenderGuiLayerEvent.Pre`, `ViewportEvent.ComputeCameraAngles/ComputeFogColor/RenderFog`,
`ComputeFovModifierEvent`, `ClientPlayerNetworkEvent.LoggingIn/LoggingOut`, `SelectMusicEvent`, `ScreenEvent.Opening`,
המקביל של `FMLClientSetupEvent`.

### 2.6 גרסאות Minecraft מאוחרות יותר

הקוד של Enhanced קורא למתאם ה-GameTest שלו "מתאם **r1** (‏1.21.1)" וכותב דוחות שוויון (parity) לכל `<mc>-<loader>`,
וזה העקבה היחידה לשיטה: **קו גרסאות אחד לכל מתאם**. Bending הולך לפיה במפורש: `common/` מכוון לגרסה
הנתמכת הנמוכה ביותר; הבדלי API של vanilla בין גרסאות נמצאים מאחורי **חבילת מתאם `mc/`** קטנטנה
(‏`platform/mc/McApi` כש-`r1` הוא המימוש היחיד ב-1.x: `getMoonPhase`, בנאים של אפשרויות חלקיקים, גישות ל-`Camera`,
כל דבר ששינה שם בין 1.21.1 ל-1.21.x), וגרסת Minecraft חדשה מתווספת כ**מטריצת מאפייני Gradle**
(‏`-Pmc=1.21.4` שבוחר את `neo_version`/`fabric_version`/`parchment`), מתאם `r2` חדש במקום שבו
משהו זז, ודוחות השוויון `1.21.4-neoforge.json` / `1.21.4-fabric.json` מושווים מול `1.21.1-*`. קובץ
ה-JAR האוניברסלי הוא לכל גרסת Minecraft (‏JAR אחד לכל גרסת MC, וכל אחד אוניברסלי בין לואדרים), כמו
כל מוד מרובה-לואדרים. **לאשר מול PLAN2 אם הוא קיים.**

## 3. חבילות

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

## 4. רישומים (מה ש-S0 רושם; הנתיבים ממלאים את ההתנהגות)

**פריטים:** `element_scroll`, `water_skin` (פס בסגנון עמידות דרך רכיב נתונים `WATER` בטווח 0–8), `air_glider` (רכיב
`DYE`: ‏`DyeColor`), הצעות: `bending_scroll_earth|air|water|fire`, `training_dummy`.

**בלוקים (מצבים זמניים הם בלוקים אמיתיים עם פנקס `TempBlocks`):** `quicksand` (בלוק דמוי `SoulSand`:
`getBlockSupportShape`, ‏`entityInside` מאט ומטביע), `steel_plate`, `ice_spike` (3 מצבי `RISE`), `temp_lava` (בלוק
לא זורם במראה של `LiquidBlock`: הוא מרחיב את `Block` עם הרינדור של לבה ו-`entityInside` שגורם נזק לבה דרך
`DamageSources.lava`, ‏`lightLevel 15`, לא נוזל: בלי התפשטות), `wrecked_hopper|rail|piston|redstone|iron_door|
furnace|anvil|metal_block` (8 בלוקי `WreckedBlock` שנושאים את המצב המקורי בישות הבלוק שלהם `WreckedBlockEntity`:
`original` + ה-NBT המקורי), הצעות: `training_dummy`, `metal_dummy`. **ישויות בלוק:** `wrecked`, `training_dummy`.

**ישויות** (כולן `MobCategory.MISC`, ‏`noSummon` כשהן פנימיות):

| ישות | גודל | מעקב / מרווח | נשמרת | הערות |
|---|---|---|---|---|
| `rock` (‏`RockEntity extends TelekineticBlockEntity` בדפוס) | 0.9 (סלע ענק 1.9) | 10 / 1 | לא | נושאת `BlockState`, מוחזקת/משוגרת, הנזק לפי מהירות |
| `sand_stream` | 1.0 | 8 / 2 | לא | סכנה בצורת חרוט עם מיקום; 10 טיקים |
| `steel_knife` | 0.25 | 6 / 5 | לא | נתקעת במטרות (עוקבת אחריהן 3 ש׳) |
| `lava_shuriken` | 0.6 | 8 / 2 | לא | קופצת פעם אחת ברמת שליטה M10 |
| `air_blade` | 0.9 רוחב 0.3 גובה | 8 / 2 | לא | עוברת דרך יצור אחד |
| `air_scooter` | 1.0 × 0.9 | 10 / 1 | לא | רכב (`canAddPassenger`), מוּנע בקלט של הרוכב (הלקוח שולח `Input` דרך תנועת הרכב של vanilla) |
| `tornado` | 2.0 × גובה | 12 / 2 | לא | נושא רוכבים (מסלול), נסחף, נשלח מחדש כל 4 טיקים |
| `spirit` | 0.6 × 1.8 | 12 / 1 | לא | `noPhysics`, ישות המצלמה של הבעלים שלה |
| `water_shot` | 0.4 | 8 / 2 | לא | |
| `water_spout` | 2.0 × גובה | 10 / 1 | לא | רכב |
| `ice_knife` | 0.25 | 6 / 5 | לא | |
| `snow_burst` | 0.25 | 6 / 5 | לא | וריאציה של כדור שלג |
| `fireball` | 0.3–1.2 | 8 / 2 | לא | |

סכנות שהן **לא ישויות** (אובייקטי `Hazard` בצד השרת שמתקתקים על ידי `HazardTicker` מ-`LevelTickEvent.Post`,
הדפוס `SandHazards` של Enhanced): משך החיים של קירות, עמודים, כתמי חולות טובעניים, שבילי לבה, דיסקיות קרח, חרוט
פיצוץ האוויר (טיק אחד), פרוסת האוויר, טבעות מעגל האש, כדור שואב האוויר, אחיזת החנק, אחיזת הדם, טעינת/בליץ
הברק, חלון ההטיה, קרניים (`ContinuousBeam`: להביור, נשימת אש).

**סוגי נזק** (`data/bending/damage_type/*.json`, כולם בתג `bending:bending_damage`; אלה של ברק גם ב-
`minecraft:is_lightning`; אלה של אש ב-`minecraft:is_fire`; קרניים ב-`bypasses_cooldown` + `no_knockback`): `rock`,
`boulder`, `earth_wall_push`, `shed_blast`, `pillar`, `sand`, `quicksand`, `lava_path`, `lava_shuriken`, `steel_knife`,
`metal_fling`, `wreck`; ‏`air_blast`, `air_shield_burst`, `air_blade`, `air_slice`, `tornado`, `scooter_burst`, `suffocation`,
`air_vacuum`; ‏`water_shot`, `water_whip`, `octopus`, `spout_ram`, `water_arm`, `ice_knife`, `ice_spike`, `snow`, `blood`,
`blood_fling`; ‏`fireball`, `fire_punch`, `fire_kick`, `flamethrower`, `fire_breath`, `fire_whip`, `fire_circle`, `lightning`,
`lightning_arc`, `redirected_lightning`. לכל אחד יש שורות `death.attack.bending.<id>` ו-`.player` בשתי השפות.

**אפקטים:** `paralysed` (בלי תנועה/התקפה/כשפות; הלקוח נועל קלט), `severed` (הכשפות נדחות; משך = 
הטיימר), `chilled` (האטה + פי 1.5 אש), `wet` (3 ש׳; פי 1.5 ברק), `zaheer_tether_cut`? (לא: דגל שמור), ו-`stuck`
הדומה ל-`entombed` (חולות טובעניים/רגליים קפואות: בלי קפיצה, מהירות ×0.2).

**חלקיקים:** `energy` (מועתק), `grain`, `droplet`.

**תגים** (`data/bending/tags/`): בלוק `earth`, `sand`, `metal`, `metal_devices`, `plants`, `unbendable`, `water_sources`
(מים, קלחת, שכבות שלג); entity_type ‏`bosses`, `no_breath`, `no_blood`, `fire_mobs`; damage_type ‏`bending_damage`,
`water_damage`, `fire_damage`, `lightning_damage`; פריט `metal_charge` (מטילים/גושונים ← ערך טעינה במפת נתונים? לא:
אין מפות נתונים ב-Fabric: זוג תגים `metal_charge_2` ו-`metal_charge_1`).

**רכיבי נתונים:** `water` (מספר שלם 0–8), `glider_dye` (‏`DyeColor`).

**הישגים** (לשונית `bending:`): להפוך לכל אלמנט, הפתיחה הראשונה, רמת שליטה 10, העץ המלא, דרכו של זהיר, ניתוק,
הטיית ברק, התוספות.

**לשונית יצירתית** `bending` עם שלושת הפריטים.

## 5. נתונים ופורמט שמירה

### 5.1 ‏`BenderSave` (‏`SavedData` של vanilla, ‏`data/bending_players.dat` בעולם הרגיל (Overworld); רשומה אחת לכל UUID של שחקן)

```
Codec<BenderSave> = RecordCodecBuilder:
  element: Element (string; "none")
