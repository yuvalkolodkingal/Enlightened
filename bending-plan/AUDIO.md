# Bending — Audio

Every sound and every piece of music is **synthesised for the mod**: no audio from the shows, no copyrighted music, no
samples. The pipeline, format rules and conventions are Enhanced's (section 1); the events are listed per element with a
synthesis recipe each and both subtitles (sections 2–6); the music is section 7.

## 1. Pipeline and rules

### 1.1 The toolkit (committed this time)

Enhanced's sounds were made with a numpy toolkit (`sfx.py`: oscillators, noise, filters, modal synthesis, a synthetic
reverb) and per-release recipe scripts (`thanos_sounds.py`...), **none of which are in the Enhanced repository**. Bending
**commits its toolkit** under `tools/sound/`:

* `tools/sound/sfx.py`: the primitives: `sine`, `saw`, `square`, `tri`, `noise(white|pink|brown)`, `glide(f0, f1, t)`,
  `env(a, d, s, r)` and `envexp`, `lowpass/highpass/bandpass(cutoff, q)` (biquads), `modal(freqs, decays, amps)` (struck
  objects: stone, metal, glass), `karplus(freq, decay)` (plucked strings), `formant(vowel)` (breath and voice-like
  tones), `chorus`, `reverb(size, damp, wet)` (a Schroeder/FDN synthetic hall), `limiter(−1 dBFS)`, `loop_seam(x)` (a
  crossfade of the last 100 ms into the first so a loop is seamless), `write_ogg(path, x)` (mono, 44.1 kHz, Vorbis q 5
  through `soundfile`/`pyogg`), and `analyze(path)` (peak, LUFS-ish loudness, energy above 12 kHz, length).
* `tools/sound/recipes/<element>.py`: one function per event, named as the event, returning the signal; the tables below
  are these functions' specifications.
* `tools/sound/render.py`: renders every recipe into `src/main/resources/assets/bending/sounds/sfx/<event>.ogg` (music to
  `sounds/music/`), prints the analysis, and fails on a peak over −1 dBFS, a length under 0.5 s, a loop under 2 s or an
  ambience under 15 s, a non-mono or non-44.1 kHz file.
* `tools/sound/placeholder.ogg`: the 0.2 s placeholder S0 ships for every event until the sound lane renders it (its
  md5 is pinned by `SoundAssetTests.PLACEHOLDER_MD5`, as Enhanced's).

### 1.2 Format and registration (Enhanced's rules, enforced by `SoundAssetTests` and `DataTests`)

* **Ogg Vorbis, mono, 44,100 Hz**, longer than 0.5 s; loops ≥ 2 s and seamless; ambience ≥ 15 s; peak ≤ −1 dBFS.
* Registered in `registry/ModSounds` (`Registrars.create(Registries.SOUND_EVENT, "bending")`, `register(name)` →
  `SoundEvent.createVariableRangeEvent`), one region per lane (`// ===== earth (1.0) =====`), appended only.
* `sounds.json`: `"<event>": { "subtitle": "subtitles.bending.<event>", "sounds": [ { "name": "bending:sfx/<event>" } ] }`,
  `"stream": true` for music and loops over 10 s; one block per lane inserted before the next lane's block.
* Subtitles: `subtitles.bending.<event>` in `en_us.json` and `he_il.json`, Hebrew with Hebrew letters, no transliteration.
* **Takes.** Short one-shots that repeat (hits, stomps, knives) get **2–3 takes** (`sounds: [a, b, c]`, random) so they do
  not machine-gun; the recipe renders the takes with a different seed and ±4% pitch.
* **Loops** play through the copied `PowerLoopSounds` (`Kind(sound, volume, pitch, predicate on BenderData)`), fading in
  0.2 and out 0.15 per tick, following the player's eyes; entity-bound loops (tornado, spout, fireball) through a
  per-entity `AbstractTickableSoundInstance` (Enhanced's `Loop` generalised to `EntitySoundLoop`).
* **Mastering**: effects normalised to about −16 LUFS-ish by `analyze`; impacts a little louder (−14), loops quieter
  (−20), ambience (−24); music stems 6–8 dB under the effects (Enhanced's rule).
* **Distance**: default attenuation (16 blocks) for small sounds; big ones (lightning strike, boulder impact, shield burst,
  tornado) are played with volume 1.5–2.0 (vanilla scales range by volume) and the lightning adds a separate
  `thunder_distant` played at the listener for 24–64 blocks.

### 1.3 Voice

No words anywhere (the shows' lines are theirs). Grunts and breaths are **synthesised** (`formant` tones with noise, no
samples): the airbender's breath, the firebender's shout at a big move (a short formant "ha!" at 180 Hz with a noise
burst), the blood-bending victim's gasp, the spirit's hollow wind.

---

## 2. Shared events (release 1.0)

| Event | Length | Recipe | Subtitle EN | Subtitle HE |
|---|---|---|---|---|
| `awaken_earth` | 2.0 s | a deep stomp (`modal` 60/90 Hz, 0.3 s) → a rising rumble (brown noise, LP 200→600 Hz swell 1.2 s) → a stone "lock" (modal 320/510/780, 0.4 s) + hall reverb | You become an Earthbender | הפכתם לכשפי אדמה |
| `awaken_air` | 2.0 s | a wide wind swell (pink noise BP 800→2400 Hz, 1 s) → three rising sine chimes (523, 659, 784 Hz, 0.3 s each) → a soft gust tail | You become an Airbender | הפכתם לכשפי אוויר |
| `awaken_water` | 2.0 s | a liquid swell (`formant` "oo" 220 Hz with LP noise ripple 6 Hz) → a shimmer (sine cluster 1046–1568 Hz, detuned) → a slow splash tail | You become a Waterbender | הפכתם לכשפי מים |
| `awaken_fire` | 2.0 s | an inhale (HP noise 1→2 kHz, 0.5 s) → an ignition "fwoom" (saw 110 Hz + crackle bursts, LP 3 kHz, 0.6 s) → a brass-like swell (saw stack 220/330/440, 0.8 s) | You become a Firebender | הפכתם לכשפי אש |
| `mastery_up` | 0.8 s | a two-note chime (sine 880 → 1320 Hz with a soft attack) + a tiny sparkle (3 short sines 2–4 kHz) | Mastery rises | רמת השליטה עולה |
| `points` | 0.5 s | a soft gold "ting" (modal 1760/2640 Hz, 0.4 s) | Skill points earned | נקודות כישור |
| `unlock` | 1.0 s | the element's awaken chord at a third of the length, quieter | Ability unlocked | יכולת נפתחה |
| `scroll_open` | 0.6 s | paper rustle (pink noise HP 2 kHz, 3 short envelopes) | Scroll unrolls | מגילה נפתחת |
| `look_change` | 0.6 s | a cloth whoosh (noise BP 1 kHz sweep 0.3 s) + the element's small chime | Look changes | החלפת מראה |
| `hud_ready` | 0.15 s | a soft click (sine 1200 Hz, 20 ms) — client UI, no subtitle | — | — |
| `refused` | 0.3 s | a dull double tap (modal 180/270, 2× 60 ms) | Cannot bend here | אי אפשר לכשף פה |
| `hit_stagger` | 0.3 s | a body thump (sine 70 Hz + noise burst 40 ms) | Hit | פגיעה |
| `land_soft` | 0.4 s | a padded thud (sine 55 Hz, 0.2 s) + a dust hiss (noise LP 1 kHz, 0.3 s) | Lands | נחיתה |
| `temp_revert` | 0.5 s | a soft stone slide reversed (noise BP 300 Hz swell 0.4 s) + a click; 3 takes | Ground settles | הקרקע מתיישבת |

---

## 3. Earth (release 1.0)

| Event | Length | Recipe | EN | HE |
|---|---|---|---|---|
| `earth_stomp` | 0.5 s | a deep thud (sine 50→40 Hz glide, 0.25 s) + gravel crunch (brown noise BP 400–1200 Hz, 0.15 s, granular 8 grains); 3 takes | Earthbender stomps | כשף אדמה רוקע |
| `earth_rock_lift` | 0.6 s | a tearing crunch (noise BP 300–2 kHz, 0.2 s) → a rising rumble (sine 60→90 Hz, 0.4 s) + pebble clicks (6 modal taps 2–4 kHz) | Rock tears free | סלע נתלש |
| `earth_rock_throw` | 0.5 s | a low whoosh (brown noise LP 800 Hz, envelope 0.35 s with grit: amplitude-modulated 40 Hz) | Rock flies | סלע עף |
| `earth_rock_hit` | 0.5 s | a cracking thump (modal stone 180/290/450 Hz with a 5 ms broadband crack, 0.35 s) + debris clicks; 3 takes | Rock shatters | סלע מתנפץ |
| `earth_rock_hit_flesh` | 0.4 s | the thump without the crack, duller (LP 600 Hz) + a body thud | Rock strikes | סלע פוגע |
| `earth_boulder_throw` | 0.8 s | `earth_rock_throw` an octave down + a sub thump 35 Hz | Boulder flies | סלע ענק עף |
| `earth_boulder_hit` | 1.0 s | a sub-bass drop (sine 45→25 Hz, 0.5 s) + a big crack + a rubble cascade (12 modal taps over 0.5 s) + reverb | Boulder smashes | סלע ענק מתרסק |
| `earth_hold_loop` | 3.0 s loop | a quiet rumble (brown noise LP 120 Hz, slow 0.3 Hz tremolo) + occasional grit ticks | Rock hovers | סלע מרחף |
| `earth_rock_down` | 0.5 s | a soft stone settle (modal 180/290, no crack) + a dust hiss | Rock settles | סלע מתיישב |
| `earth_armour_form` | 1.2 s | stomp → stone sliding (noise BP 200–900 Hz rising 0.7 s, granular) → a heavy "clunk" seal (modal 140/220/360 Hz, 0.3 s) | Earth armour forms | שריון אדמה נוצר |
| `earth_armour_step` | 0.25 s | a stony tread (modal 220/330, 60 ms + grit); 3 takes, volume 0.4 | Stone footstep | צעד אבן |
| `earth_armour_hit` | 0.3 s | a chip (modal 1200/1800 Hz, 40 ms) + a dull knock (sine 90 Hz) | Armour takes a hit | השריון נפגע |
| `earth_armour_crack` | 0.4 s | a sharp split (broadband 3 ms + modal 800/1300 Hz decaying 0.3 s) | Armour cracks | השריון נסדק |
| `earth_armour_break` | 1.0 s | a rubble collapse (14 modal taps falling in pitch 600→150 Hz over 0.6 s) + a dust cloud (noise LP 1 kHz swell) | Earth armour crumbles | שריון האדמה מתפורר |
| `earth_shed_blast` | 0.9 s | `earth_armour_break` + a crack and a wide shard hiss (noise HP 2 kHz, 0.4 s, stereo-ish via two detuned copies) | Armour bursts | השריון מתנפץ |
| `earth_rumble_loop` | 2.0 s loop | brown noise LP 150 Hz with a 2 Hz wobble | Earth rumbles | האדמה רועמת |
| `earth_clunk` | 0.2 s | one stone block clunk (modal 160/260/410 Hz, 0.15 s); 3 takes | Stone shifts | אבן זזה |
| `earth_pillar_rise` | (loop + clunks) | `earth_rumble_loop` pitched up 1.2 while rising + `earth_clunk` per block | Earth pillar rises | עמוד אדמה מתרומם |
| `earth_pillar_sink` | 0.8 s | the rumble descending (sine 80→45 Hz) + two clunks | Earth pillar sinks | עמוד האדמה שוקע |
| `earth_launch` | 0.6 s | a stomp + a sharp stone crack-whoosh (crack 3 ms → noise BP 400→1500 Hz 0.3 s upward) | Earth launch | זינוק אדמה |
| `earth_land_soft` | 0.5 s | `land_soft` + a stone dip (modal 120 Hz) | Earthbender lands | כשף אדמה נוחת |
| `earth_wall_rise` | 0.5 s | stomp → a fast rising stone slide (noise BP 300→1200 Hz 0.3 s) + four clunks 60 ms apart | Earth wall rises | קיר אדמה מתרומם |
| `earth_wall_hit` | 0.3 s | a knock with grit (modal 200/330 Hz + noise 30 ms); pitch by the hit's damage (0.8–1.2) | Wall takes a hit | הקיר נפגע |
| `earth_wall_sink` | 0.9 s | a slow grinding fall (noise BP 200–600 Hz, 0.7 s, falling) + a settle | Earth wall sinks | קיר האדמה שוקע |
| `earth_scrape_loop` | 2.0 s loop | a heavy stone scrape (noise BP 150–500 Hz with 12 Hz granular stutter) | Wall scrapes | הקיר נגרר |
| `earth_wall_push` | 0.5 s | a final thud + the scrape ending | Wall pushes | הקיר דוחף |
| `sand_lift` | 0.5 s | a dry hiss rising (white noise HP 3 kHz → BP 1.5 kHz, 0.5 s) | Sand lifts | חול מתרומם |
| `sand_stream` | 0.6 s | a grainy rush (pink noise BP 1–4 kHz, 0.5 s, amplitude granular 60 Hz) | Sand streams | זרם חול |
| `sand_hit` | 0.3 s | a sandy pelt (noise HP 2 kHz, 80 ms, 4 overlapping) + a soft thud; 3 takes | Sand pelts | חול מצליף |
| `sand_wall_rise` | 0.6 s | pouring sand reversed: a swelling hiss (noise BP 1–3 kHz, 0.5 s crescendo) | Sand wall rises | קיר חול מתרומם |
| `sand_wall_fall` | 0.8 s | the slump: noise BP 800 Hz falling 0.6 s + a soft thud | Sand wall collapses | קיר החול קורס |
| `quicksand_cast` | 0.6 s | a low sandy groan dropping (sine 90→50 Hz + grainy noise, 0.6 s) | Quicksand forms | חול טובעני נוצר |
| `quicksand_sink_loop` | 2.5 s loop | a wet, grainy churn (brown noise LP 400 Hz with slow 0.5 Hz bubbling tremolo) volume 0.5 | Sinking in quicksand | שוקעים בחול טובעני |
| `quicksand_free` | 0.4 s | a sucking pop (sine glide 200→600 Hz 80 ms + a noise pop) | Freed from quicksand | שוחררו מהחול הטובעני |
| `sand_cloud` | 0.8 s | a broad hiss swell (pink noise BP 1–3 kHz, 0.8 s) | Sand cloud | ענן חול |
| `sand_cloud_loop` | 3.0 s loop | a soft swirling hiss, 0.3 Hz pan-like tremolo, volume 0.3 | Sand swirls | חול מסתחרר |
| `lava_path_cast` | 0.9 s | stomp + a molten rumble swelling (brown noise LP 300 Hz + sine 55 Hz, 0.8 s) with crackling (random 2 ms clicks) | Lava path flows | שביל לבה זורם |
| `lava_path_loop` | 3.0 s loop | bubbling hiss (noise LP 800 Hz with random 3–6 Hz bubbles: sine pops 150–300 Hz) volume 0.5 | Lava bubbles | לבה מבעבעת |
| `lava_cool` | 0.4 s | a sizzling tick (noise HP 4 kHz, 0.3 s decay) + a faint stone click | Lava cools | הלבה מתקררת |
| `lava_shuriken_throw` | 0.5 s | a hot whoosh (noise BP 600 Hz→2 kHz, 0.4 s) with a spinning flutter (AM 25 Hz) | Lava shuriken flies | שוריקן לבה עף |
| `lava_shuriken_spin_loop` | 2.0 s loop | a thin flutter (noise BP 2 kHz, AM 25 Hz) volume 0.3 | Shuriken spins | השוריקן מסתובב |
| `lava_shuriken_hit` | 0.5 s | a splat-sizzle (noise burst LP 1 kHz 50 ms → HP 4 kHz sizzle 0.4 s) | Lava shuriken strikes | שוריקן לבה פוגע |
| `metal_knives_form` | 0.4 s | three quick steel "shinks" (modal 2400/3600/5200 Hz, 60 ms each, 100 ms apart) | Steel knives form | סכיני פלדה נוצרים |
| `metal_knife_throw` | 0.3 s | a thin whistle (sine 3200→2200 Hz glide, 0.25 s) + a short air hiss; 3 takes | Steel knife flies | סכין פלדה עפה |
| `metal_knife_hit` | 0.3 s | a "tink" (modal 3000/4500 Hz, 80 ms) into stone; a dull "thuk" (LP 800 Hz) into flesh; 3 takes each | Steel knife strikes | סכין פלדה פוגעת |
| `metal_knife_stick` | 0.5 s | a quivering twang (sine 180 Hz with 12 Hz FM decaying, 0.45 s) | Knife quivers | הסכין רוטטת |
| `metal_recall` | 0.5 s | a reversed shink glide (modal 5200→2400 Hz) + a catch click | Knives return | הסכינים חוזרות |
| `metal_pull_loop` | 2.5 s loop | a scraping grinding hum (saw 70 Hz LP 400 + noise BP 300–900 Hz granular) + a filings hiss (HP 5 kHz sparkle), pitch rises with charge (0.9–1.2) | Metal pulled | מתכת נשלפת |
| `metal_pull_end` | 0.2 s | a clink (modal 2800/4200 Hz, 0.15 s) | Metal gathered | המתכת נאספה |
| `metal_wall_rise` | 0.6 s | plates sliding (noise BP 1–2 kHz 0.3 s) and locking: four clanks (modal 420/630/1100 Hz, 80 ms, 70 ms apart) | Steel wall rises | קיר פלדה מתרומם |
| `metal_wall_hit` | 0.5 s | a resonant plate ring (modal 380/760/1140 Hz, 0.45 s) | Steel wall rings | קיר הפלדה מהדהד |
| `metal_armour_on` | 1.0 s | five plate claps (modal 300/450/700 Hz, 60 ms, 120 ms apart) + a final heavier lock (modal 180/270, 0.3 s) | Steel armour locks on | שריון פלדה ננעל |
| `metal_armour_off` | 0.9 s | five reversed claps (each 60 ms clap with a 40 ms reversed tail) descending | Steel armour peels off | שריון הפלדה מתקלף |
| `metal_armour_hit` | 0.4 s | a dented clang (modal 500/800 Hz with a pitch dip, 0.35 s) | Steel armour dents | שריון הפלדה נמעך |
| `metal_armour_break` | 1.0 s | a cascade of falling plates (8 claps falling in pitch over 0.7 s) + a clatter tail | Steel armour falls apart | שריון הפלדה מתפרק |
| `metal_wreck` | 0.6 s | a crunching, groaning twist (saw 90→60 Hz with heavy FM 7 Hz + metallic noise BP 800–3 kHz, 0.5 s) + a final creak | Metal wrecked | מתכת נהרסת |
| `metal_mend` | 0.6 s | a soft reversed crunch (the twist reversed, LP 2 kHz, quieter) + a chime (sine 1760 Hz) | Metal mends | המתכת מתוקנת |
| `metal_grip` | 0.3 s | a clamp clank (modal 350/520 Hz, 0.2 s) | Metal gripped | מתכת נאחזת |
| `metal_fling` | 0.5 s | a swung-metal whoosh (noise BP 500 Hz→1.5 kHz, 0.3 s) + a clank | Metal flung | מתכת מוטלת |

---

## 4. Air (release 1.1)

| Event | Length | Recipe | EN | HE |
|---|---|---|---|---|
| `air_gather` | 0.5 s | a soft inhale-like rising whoosh (pink noise BP 600→1800 Hz, 0.45 s) | Air gathers | אוויר נאסף |
| `air_blast` | 0.6 s | a hard breathy "whumpf": a 20 ms burst (noise LP 400 Hz) → a 0.4 s decaying rush (BP 500–3 kHz falling) | Air blast | הדף אוויר |
| `air_hit` | 0.3 s | a hollow thump (sine 110 Hz with a resonant LP, 0.25 s) | Thrown by air | נזרקו מהאוויר |
| `air_jump` | 0.4 s | a short down-whoosh (noise BP 1.5 kHz→400 Hz, 0.25 s) + a puff | Air jump | קפיצת אוויר |
| `air_shield_up` | 0.8 s | a rising swirl (noise BP sweeping 400→2000 Hz with a 1.5 Hz rotation tremolo) into a sustained hush | Air shield rises | מגן אוויר מתרומם |
| `air_shield_loop` | 3.0 s loop | a soft airy rotating hum (noise BP 900 Hz, 1.5 Hz tremolo, + a sine 220 Hz very low), pitch 1.0→1.1 as strength falls, volume 0.35 | Air shield hums | מגן האוויר מהמהם |
| `air_shield_hit` | 0.3 s | a muffled "whup" (noise LP 600 Hz, 60 ms) + a flutter (AM 20 Hz tail 0.2 s) | Air shield deflects | מגן האוויר מסיט |
| `air_shield_down` | 0.6 s | the hush falling away (noise BP 1500→300 Hz, 0.5 s fade) | Air shield lowers | מגן האוויר יורד |
| `air_shield_burst` | 0.9 s | a deep "boom-whoosh": sine 50 Hz thump + a wide rush (two detuned noise copies BP 400–4 kHz, 0.7 s) | Air shield bursts | מגן האוויר מתפוצץ |
| `glider_open` | 0.4 s | a cloth snap (noise HP 1 kHz, 30 ms) + a flap (noise LP 800 Hz, 0.2 s) | Glider opens | הגלשן נפתח |
| `glider_close` | 0.4 s | two soft flaps and a wooden click (modal 900 Hz) | Glider closes | הגלשן נסגר |
| `glider_loop` | 3.0 s loop | wind over cloth (pink noise BP 300–1500 Hz with a 3 Hz flutter), volume and pitch with speed (0.3–0.7, 0.9–1.2) | Gliding | גלישה |
| `air_blade_cast` | 0.4 s | a slicing whoosh (noise BP 2 kHz→6 kHz, 0.25 s) with a high "shing" (sine 5200 Hz, 0.1 s) | Air blade | להב אוויר |
| `air_blade_fly_loop` | 2.0 s loop | a thin whistle (sine 2800 Hz with 6 Hz vibrato + noise HP 4 kHz), volume 0.3 | Air blade whistles | להב האוויר שורק |
| `air_blade_hit` | 0.3 s | a crisp cut (noise HP 3 kHz, 40 ms) + a thump | Air blade cuts | להב האוויר חותך |
| `air_slice` | 0.7 s | a spinning whoosh rising then falling (noise BP 800→3000→800 Hz, 0.6 s, AM 8 Hz) | Air slice | חיתוך אוויר |
| `spirit_sit` | 0.6 s | a soft breath out (`formant` "hh" noise LP 2 kHz, 0.4 s) + cloth settling | Sits to meditate | מתיישבים למדיטציה |
| `spirit_leave` | 1.2 s | a rising shimmer (sine cluster 880–1760 Hz detuned, 1 s) with a bell-like inhale (sine 1320 Hz swelling) | Spirit leaves the body | הרוח יוצאת מהגוף |
| `spirit_loop` | 15 s loop (stream) | a quiet hollow wind (pink noise BP 200–800 Hz) with distant chimes (random sine 1–3 kHz, 1 per 3 s, reverb), volume 0.3; the spirit's client only | Spirit world | עולם הרוחות |
| `spirit_return` | 0.9 s | the shimmer reversed + a soft "whum" (sine 110 Hz, 0.3 s) | Spirit returns | הרוח חוזרת |
| `spirit_snap` | 0.5 s | a sharp intake (noise HP 1 kHz reversed 0.2 s) + a crack (3 ms) + a short ring | Snapped back | הרוח נמשכת בחזרה |
| `spirit_meditate_loop` | 6.0 s loop | a slow breathing hush (noise LP 1 kHz with a 0.25 Hz envelope), volume 0.2 | Meditating | מדיטציה |
| `air_bubble_on` | 0.5 s | a glassy "bloop" reversed (sine 400→900 Hz, 0.15 s) into a hush | Air bubble forms | בועת אוויר נוצרת |
| `air_bubble_loop` | 4.0 s loop | a soft underwater hum (sine 150 Hz + noise LP 500 Hz) with slow breathing (0.3 Hz), volume 0.3 | Breathing in a bubble | נושמים בבועה |
| `air_bubble_off` | 0.3 s | a pop (sine 600→200 Hz, 50 ms + noise click) | Air bubble pops | בועת האוויר מתפוצצת |
| `air_cushion` | 0.4 s | a soft "fwoomp" (noise LP 300 Hz, 0.3 s) | Air cushion | כרית אוויר |
| `air_scooter_mount` | 0.5 s | a spin-up whoosh (noise BP 400→2000 Hz with AM 6→20 Hz, 0.4 s) | Air scooter | קטנוע אוויר |
| `air_scooter_loop` | 2.0 s loop | a continuous airy whir (noise BP 1.2 kHz, AM 18 Hz) pitch 0.8–1.3 and volume 0.3–0.5 with speed | Riding the air scooter | רוכבים על קטנוע האוויר |
| `air_scooter_dismount` | 0.4 s | a whoosh down (noise BP 2000→400 Hz, 0.3 s) | Air scooter ends | קטנוע האוויר נגמר |
| `air_scooter_burst` | 0.6 s | `air_shield_burst` shortened | Air ball bursts | כדור האוויר מתפוצץ |
| `air_spout_loop` | 2.5 s loop | a rushing updraft hum (noise BP 500–2500 Hz rising tremolo 0.7 Hz), volume 0.5 | Air spout | עמוד אוויר |
| `air_spout_end` | 0.6 s | the rush fading with a soft puff | Air spout fades | עמוד האוויר נעלם |
| `tornado_cast` | 0.7 s | a spinning whoosh building (noise BP 300→1500 Hz, AM 4→12 Hz) into a roar onset | Tornado forms | טורנדו נוצר |
| `tornado_loop` | 4.0 s loop (stream) | a deep wide rotating roar (brown noise LP 400 Hz + pink BP 600–2400 Hz, 1.2 Hz rotation tremolo) with a whistling top (sine 1800 Hz vibrato 5 Hz, quiet), volume 0.7 | Tornado roars | הטורנדו שואג |
| `tornado_end` | 0.8 s | the roar unwinding (tremolo slowing 1.2→0.3 Hz, fade 0.8 s) | Tornado fades | הטורנדו נעלם |
| `air_wheel_loop` | 2.0 s loop | a fast whirring rush (noise BP 1.5 kHz, AM 30 Hz), volume 0.5 | Air wheel | גלגל אוויר |
| `air_wheel_hit` | 0.3 s | a thump + a short rush | Air wheel strikes | גלגל האוויר פוגע |
| `tethers_cut` | 2.0 s | a slow rising chord (sines 196/294/392 Hz swelling 1.5 s) with a **snapping string** (`karplus` 110 Hz plucked hard + a 3 ms crack at 1.2 s) and a wind opening after | Earthly tethers cut | הקשרים הארציים ניתקו |
| `flight_takeoff` | 0.5 s | a soft lift whoosh (noise BP 600→1500 Hz, 0.4 s) | Takes flight | מתרוממים לתעופה |
| `flight_loop` | 3.0 s loop | a gentle wind rush (pink noise BP 400–1800 Hz), volume 0.2–0.5 with speed | Flying | תעופה |
| `suffocate_start` | 0.4 s | a sharp inhale reversed (noise HP 1.5 kHz, 0.3 s reversed envelope) | Air stolen | האוויר נלקח |
| `suffocate_loop` | 3.0 s loop | a thin sucking wind (noise BP 2–5 kHz with a slow inward swell 0.5 Hz) with a heartbeat under it (sine 55 Hz, two beats per 1.1 s), volume 0.5 | Suffocating | חניקה |
| `suffocate_victim_loop` | 3.0 s loop | a choking breath (`formant` "uh" 160 Hz with noise, gasps every 1 s) + a louder faster heartbeat (1.4 Hz) — the victim's client | Cannot breathe | אי אפשר לנשום |
| `suffocate_end` | 0.5 s | a gasp of released air (noise BP 1 kHz burst 0.3 s with a vowel formant) | Breath returns | הנשימה חוזרת |
| `air_vacuum` | 1.0 s | a deep implosive "thoomp" (sine 60→30 Hz, 0.15 s + noise LP 200 Hz) → 0.3 s of near-silence (a −30 dB hum) → a soft rush back | Air vacuum | ואקום אוויר |
| `air_vacuum_loop` | 3.0 s loop | a hollow pressure hum (sine 80 Hz + filtered noise notch at 1 kHz), volume 0.3 | Airless | בלי אוויר |
| `air_master_loop` | 15 s loop (stream) | a faint breeze (pink noise BP 300–1200 Hz, slow 0.2 Hz swells), volume 0.15; ambient of a master | Breeze | רוח קלה |

---

## 5. Water, ice and blood (release 1.2)

| Event | Length | Recipe | EN | HE |
|---|---|---|---|---|
| `water_draw` | 0.3 s | a liquid "shloop" rising (sine 300→700 Hz with LP noise ripple 8 Hz, 0.25 s) | Water drawn | מים נשאבים |
| `water_shot` | 0.5 s | a liquid whoosh (noise BP 800→2500 Hz, 0.35 s) with a wet onset (sine 250 Hz click + bubble pops) | Water shot | יריית מים |
| `water_hit` | 0.5 s | a splash (noise HP 1.5 kHz burst 0.1 s + bubbly tail LP 2 kHz 0.3 s) and a slap (sine 180 Hz, 40 ms); 3 takes | Water strikes | מים פוגעים |
| `water_splash_block` | 0.4 s | a splash without the slap | Splash | שכשוך |
| `water_vs_fire` | 0.6 s | a hiss (white noise HP 3 kHz, 0.4 s decaying) and a steam sigh (`formant` "hh" LP 1.5 kHz) | Water quenches fire | מים מכבים אש |
| `water_whip_swing` | 0.4 s | a wet whoosh (noise BP 600–2 kHz with LP ripple 10 Hz, 0.3 s) | Water whip swings | שוט מים מתנופף |
| `water_whip_crack` | 0.3 s | a liquid crack (3 ms broadband + a wet slap 180 Hz + splash tail) | Water whip cracks | שוט המים מצליף |
| `water_whip_loop` | 2.0 s loop | a soft sloshing (noise LP 800 Hz with 2 Hz bubbling tremolo), volume 0.25 | Water whip | שוט מים |
| `heal_start` | 0.6 s | a soft liquid shimmer rising (sine cluster 1046–2093 Hz detuned, LP ripple 6 Hz, 0.5 s) | Healing begins | הריפוי מתחיל |
| `heal_loop` | 4.0 s loop | a gentle bubbling (sine pops 400–900 Hz, 2–4 per s) over a warm pad (saw stack 220/330 Hz LP 600 Hz), volume 0.4 | Healing water | מים מרפאים |
| `heal_tick` | 0.3 s | a soft chime (sine 1568 Hz, 0.25 s, soft attack); fire: pitch 0.9 | A heart heals | לב מתרפא |
| `heal_end` | 0.5 s | the shimmer fading | Healing ends | הריפוי נגמר |
| `cleanse` | 0.6 s | a splash + a bright "pling" (sine 2093 Hz, 0.2 s) + a sigh (formant "ah" 200 Hz, 0.3 s) | Cleansed | טיהור |
| `octopus_form` | 1.0 s | a surging swell of water (noise LP 1.2 kHz rising 0.7 s with 4 Hz ripple) + a sloshing settle | Water octopus forms | תמנון מים נוצר |
| `octopus_loop` | 3.0 s loop | soft lapping and sloshing (noise LP 900 Hz, 1.5 Hz irregular tremolo), volume 0.35, pitch 0.9 | Water octopus | תמנון מים |
| `octopus_slap` | 0.3 s | a wet slap (sine 160 Hz 30 ms + splash HP 2 kHz 0.15 s); 3 takes | Tentacle slaps | זרוע מצליפה |
| `octopus_strike` | 0.4 s | two quick wet thwacks 90 ms apart | Tentacles strike | זרועות פוגעות |
| `octopus_lash` | 0.6 s | a wide liquid whip-crack (crack + 3 overlapping splashes) | Tentacles lash | הזרועות מצליפות |
| `octopus_drop` | 0.8 s | a collapse into a splash (noise falling LP 2 kHz→400 Hz 0.5 s + a big splash) | Water octopus collapses | תמנון המים קורס |
| `octopus_steam` | 0.5 s | a hiss (HP 3 kHz, 0.4 s) | Tentacle evaporates | זרוע מתאדה |
| `octopus_freeze` | 0.5 s | a crackling freeze (random 40 clicks 2–6 kHz over 0.4 s with a sine 1200 Hz sheen) | Tentacles freeze | הזרועות קופאות |
| `spout_rise` | 1.0 s | a rushing surge rising in pitch (noise BP 300→1500 Hz, 0.9 s) with a whirl (AM 3 Hz) | Water spout rises | עמוד מים מתרומם |
| `spout_loop` | 3.0 s loop (stream) | a roaring churning water column (brown noise LP 600 Hz + pink BP 800–3 kHz with 2.5 Hz whirl tremolo + bubble pops), volume 0.7, pitch 0.9–1.1 by height | Riding the water spout | רוכבים על עמוד המים |
| `spout_ram` | 0.5 s | a heavy splash-thud (sine 60 Hz thump + big splash) | Water spout rams | עמוד המים דוחף |
| `spout_collapse` | 1.0 s | the column dropping (noise pitch falling 1500→200 Hz 0.6 s) into a broad splash | Water spout collapses | עמוד המים קורס |
| `spout_freeze` | 0.8 s | the crackling freeze + a deep creak (saw 45 Hz FM 3 Hz, 0.5 s) | Water spout freezes | עמוד המים קופא |
| `wave_ride` | 0.8 s | a long rolling wave rising (noise LP 1 kHz swell 0.6 s) with foam hiss (HP 3 kHz) | Wave rises | גל מתרומם |
| `wave_loop` | 3.0 s loop | a rolling surf (noise LP 800 Hz, 0.8 Hz swell) + foam, volume 0.6 | Riding the wave | רוכבים על הגל |
| `wave_end` | 0.8 s | the wave breaking (a splash crash + foam fade) | Wave breaks | הגל נשבר |
| `water_arms_on` | 0.7 s | a surge running down (noise BP 1 kHz→300 Hz 0.5 s) with a wet flex (two liquid slaps) | Water arms form | זרועות מים נוצרות |
| `water_arms_loop` | 2.5 s loop | a quiet continuous slosh (noise LP 700 Hz, 1 Hz tremolo), volume 0.25 | Water arms | זרועות מים |
| `water_arm_strike` | 0.3 s | a wet snap-thwack (crack 2 ms + slap + small splash); 3 takes | Water arm strikes | זרוע מים פוגעת |
| `water_arm_grab` | 0.5 s | a wet grip (slap + a sucking sine 300→150 Hz) and a drag (noise LP 500 Hz 0.3 s) | Water arm grabs | זרוע מים אוחזת |
| `water_arm_climb` | 0.5 s | a liquid pull (noise BP 600 Hz rising 0.3 s) with a creak | Water arm pulls | זרוע מים מושכת |
| `water_arms_off` | 0.6 s | a shake-off splash (3 splashes 80 ms apart + a puddle) | Water arms fall | זרועות המים נופלות |
| `ice_knives_form` | 0.4 s | three crystalline "tinks" (modal 3200/4800/6400 Hz, 50 ms, 90 ms apart) with a frost hiss (HP 6 kHz, 0.3 s) | Ice knives form | סכיני קרח נוצרות |
| `ice_knife_throw` | 0.3 s | a thin icy whistle (sine 3600→2800 Hz + HP noise, 0.25 s); 3 takes | Ice knife flies | סכין קרח עפה |
| `ice_knife_shatter` | 0.4 s | a glassy shatter (12 modal taps 2–8 kHz over 0.25 s, decaying) ; 3 takes | Ice shatters | קרח מתנפץ |
| `ice_spikes` | 0.8 s | a cracking eruption: 10 ice cracks (broadband 2 ms + modal 1.5–4 kHz) rising in pitch over 0.6 s + a frost hiss | Ice spikes erupt | חודי קרח מתפרצים |
| `freeze_cast` | 0.8 s | a sharp frost hiss sweeping outward (noise HP 4 kHz→BP 2 kHz, 0.7 s) with crackles (random clicks 3–7 kHz, 20 per s) and a sine sheen 1400 Hz | Water freezes | מים קופאים |
| `freeze_feet` | 0.5 s | a crunching encasement (noise BP 800–2 kHz granular 0.3 s + two ice cracks) | Frozen in place | קפואים במקום |
| `ice_shatter` | 0.5 s | `ice_knife_shatter` bigger (20 taps, 0.4 s) | Ice breaks | הקרח נשבר |
| `snow_throw` | 0.3 s | a soft fluffy whoosh (noise LP 1.5 kHz, 0.25 s, soft envelope) ×4 overlapping 60 ms apart | Snow thrown | שלג נזרק |
| `snow_hit` | 0.2 s | a soft pat (noise LP 800 Hz, 80 ms) ; 3 takes | Snow hits | שלג פוגע |
| `ice_wall_rise` | 0.6 s | a fast crystalline growth: crackles rising (30 clicks 2–8 kHz over 0.4 s, rising density) into a resonant ring (modal 900/1350 Hz, 0.3 s) | Ice wall rises | קיר קרח מתרומם |
| `ice_wall_break` | 0.6 s | the shatter + a low crack (sine 120 Hz) | Ice wall shatters | קיר הקרח מתנפץ |
| `ice_melt_loop` | 3.0 s loop | a hissing drip (noise HP 3 kHz quiet + drips: sine 1200→800 Hz 60 ms, 2 per s), volume 0.3 | Ice melts | קרח נמס |
| `ice_dome` | 0.9 s | `ice_wall_rise` longer with a hollow boom at the close (sine 70 Hz + LP noise, 0.3 s) | Ice dome closes | כיפת קרח נסגרת |
| `blood_grab` | 0.8 s | a low wet straining groan (saw 65 Hz FM 4 Hz LP 300 Hz + wet noise LP 600 Hz, 0.7 s) with a heartbeat entering (sine 50 Hz, two beats) | Blood bending takes hold | כשפות הדם אוחזות |
| `blood_loop` | 3.3 s loop (stream) | a slow heartbeat (sine 50 Hz, 2 beats per 1.1 s) + a tense string drone (`karplus` 110 Hz bowed-like: sustained saw LP 500 Hz with 0.2 Hz swell) + a faint liquid creak (noise BP 200 Hz granular), volume 0.6 | Blood bending | כשפות דם |
| `blood_victim_loop` | 2.2 s loop | a louder, faster heartbeat (1.4 Hz) + a strained breath — the victim's client | Blood bent | נשלטים בכשפות דם |
| `blood_move` | 0.4 s | a creaking strain (saw 80 Hz FM 6 Hz, 0.3 s) ; 3 takes | Puppet moves | הבובה זזה |
| `blood_fling` | 0.5 s | a wet snap (crack + liquid slap) and a whoosh (noise BP 600→2 kHz, 0.35 s) | Flung by blood bending | מוטלים בכשפות דם |
| `blood_release` | 0.6 s | a breath out (formant "hh", 0.4 s) and the heartbeat stopping (one last beat) | Blood bending releases | כשפות הדם משתחררות |
| `blood_sever` | 1.5 s | a long inhale (noise HP 1 kHz, 0.5 s) → 0.3 s of near-silence → a deep soft thud (sine 45 Hz, 0.2 s) and a sigh of released air (formant "ah" falling 220→160 Hz, 0.5 s) | Bending taken away | הכשפות נשללות |
| `severed_victim` | 2.0 s | a high ring fading (sine 4186 Hz with slow 0.5 Hz amplitude wobble, 1.8 s fade) — the victim's client | Your bending is gone | הכשפות שלכם נעלמו |
| `blood_refused` | 0.6 s | a glassy chime (modal 2093/3136 Hz, 0.4 s) and a snapping string (`karplus` 220 Hz pluck + crack) | A stronger waterbender | כשף מים חזק יותר |
| `water_master_loop` | 15 s loop (stream) | a faint water lapping (noise LP 600 Hz, 0.4 Hz swells) with a rare drip, volume 0.15 | Water murmurs | מים ממלמלים |
| `moon_rise` | 1.5 s | a soft silver swell (sine cluster 1568/2349/3136 Hz detuned, 1.2 s, reverb) — when the full moon's night begins, for waterbenders | Full moon | ירח מלא |

---

## 6. Fire and lightning (release 1.3)

| Event | Length | Recipe | EN | HE |
|---|---|---|---|---|
| `fire_ignite` | 0.2 s | a sharp fire catch (noise burst BP 1–4 kHz 30 ms + a crackle 0.15 s) with a breath (HP noise 20 ms); 3 takes | Fire ignites | אש נדלקת |
| `fireball_throw` | 0.5 s | a punchy whoosh (noise BP 400→2500 Hz, 0.35 s) with a crackling body (random clicks 1–4 kHz, 30 per s) and a low "fwoom" (saw 90 Hz LP 300 Hz, 0.2 s); blue: pitch 1.08 + a hiss layer | Fireball | כדור אש |
| `fireball_fly_loop` | 2.0 s loop | a soft roaring flutter (noise BP 600–2 kHz, AM 14 Hz) volume 0.3 | Fireball flies | כדור האש עף |
| `fireball_burst` | 0.6 s | a fiery "boom-crackle": sine 55 Hz thump 0.15 s + noise burst LP 2 kHz 0.1 s + crackle tail 0.4 s (clicks decaying); great: + a sub drop 40→20 Hz | Fireball bursts | כדור האש מתפוצץ |
| `fire_flick` | 0.2 s | a tiny puff (noise LP 2 kHz, 60 ms) + a click | Small flame | להבה קטנה |
| `fire_vs_water` | 0.5 s | a hiss (HP 3 kHz, 0.4 s) and a steam sigh | Fire quenched | האש כבתה |
| `fire_fists_on` | 0.5 s | an inhale (HP noise 0.2 s) and two fire catches 100 ms apart | Fists ignite | האגרופים נדלקים |
| `fire_fists_loop` | 2.0 s loop | a low crackling flicker (noise BP 800–3 kHz with random 8–20 Hz amplitude crackle), volume 0.3 | Burning fists | אגרופים בוערים |
| `fire_punch_hit` | 0.4 s | a punch thud (sine 90 Hz 40 ms + noise LP 600 Hz) with a fiery "fwoosh" (noise BP 1–3 kHz, 0.3 s); 3 takes | Fire punch | מכת אש |
| `fire_combo` | 0.5 s | a bigger "whoomp" (sine 60 Hz + wider noise, 0.4 s) | Fire combo | שילוב אש |
| `fire_fists_off` | 0.4 s | a flame dying (noise BP 2 kHz fading 0.3 s + a puff) | Fists go out | האגרופים כבים |
| `fire_kick` | 0.5 s | a sweeping fire whoosh (noise BP 500→3000→1000 Hz, 0.4 s) with a sharp crack at the peak | Fire kick | בעיטת אש |
| `fire_kick_hit` | 0.4 s | a flaming thud (`fire_punch_hit` lower, 0.9 pitch) | Fire kick lands | בעיטת האש פוגעת |
| `flamethrower_start` | 0.5 s | an inhale and a double ignition "fwoosh" (two fire catches 60 ms apart into a rising roar onset) | Flamethrower starts | הלהביור מתחיל |
| `flamethrower_loop` | 3.0 s loop (stream) | a roaring jet (brown noise LP 800 Hz + pink BP 500–4 kHz, 10 Hz flutter) with a crackling layer (random clicks 40 per s); volume 0.7; blue: pitch 1.15 + hissing top (HP 5 kHz) | Flamethrower | להביור |
| `flamethrower_end` | 0.4 s | the roar cutting out (fast fade 80 ms) with a puff and three embers (clicks) | Flamethrower stops | הלהביור נפסק |
| `fire_breath_inhale` | 0.6 s | a long dragon's inhale (noise HP 800 Hz→LP 1.5 kHz rising, 0.55 s, with a formant "hh") | Deep breath | נשימה עמוקה |
| `fire_breath_loop` | 3.0 s loop (stream) | a deeper wider roar (`flamethrower_loop` an octave lower in the brown layer) with a vocal rasp (formant "ah" 110 Hz distorted, quiet), volume 0.8 | Fire breath | נשיפת אש |
| `fire_breath_end` | 0.5 s | a cough-puff (noise burst LP 1 kHz 60 ms ×2) and embers | Fire breath ends | נשיפת האש נגמרת |
| `fire_melt` | 0.5 s | hiss + drips | Fire melts ice | האש ממיסה קרח |
| `fire_whip_ignite` | 0.4 s | a rising fiery hiss (noise BP 800→3 kHz, 0.35 s) as the whip uncoils, with crackle | Fire whip ignites | שוט אש נדלק |
| `fire_whip_loop` | 2.5 s loop | a crackling fluttering flame (noise BP 1–3 kHz, AM 12 Hz irregular) with a soft wind, volume 0.4 | Fire whip | שוט אש |
| `fire_whip_strike` | 0.4 s | a whip-crack (3 ms broadband + a 40 ms sine 1200→300 Hz snap) with a fiery whoosh | Fire whip strikes | שוט האש מצליף |
| `fire_whip_hit` | 0.4 s | a crack-thud and a flare (noise BP 2 kHz 0.2 s) | Fire whip lands | שוט האש פוגע |
| `fire_whip_lash` | 0.6 s | a big crack with a long whoosh (0.5 s) | Fire whip lash | הצלפת שוט אש |
| `fire_whip_end` | 0.3 s | a flame snapping out (a short flutter + a puff) | Fire whip goes out | שוט האש כבה |
| `fire_circle_wind` | 0.4 s | two fire catches and a sharp inhale | Fire gathers | אש נאספת |
| `fire_circle` | 1.0 s | a spinning whoosh (noise BP 600–2 kHz, AM 6 Hz rising) erupting at 0.3 s into a wide ring roar (two detuned noise copies LP 2.5 kHz, 0.5 s) with a crackle tail | Fire circle | מעגל אש |
| `fire_circle_legs` | 1.2 s | `fire_circle` with a second lower roar (LP 1 kHz) 0.2 s after | Double fire circle | מעגל אש כפול |
| `fire_circle_snuff` | 0.4 s | a steam hiss | Fire circle cut | מעגל האש נחתך |
| `fire_jet_start` | 0.4 s | two jet ignitions (noise bursts BP 1–3 kHz 50 ms, 40 ms apart) into a thrust onset | Fire jets | סילוני אש |
| `fire_jet_loop` | 2.0 s loop | a roaring thrust (brown noise LP 500 Hz + BP 1–3 kHz, 25 Hz flutter), volume 0.6, pitch 0.9–1.2 with speed | Fire jet flight | תעופה בסילון אש |
| `fire_jet_end` | 0.5 s | the jets cutting (fade 60 ms) with embers falling (6 clicks over 0.4 s) | Fire jets stop | סילוני האש נפסקים |
| `fire_land` | 0.5 s | a fiery thud (`land_soft` + a short flare) | Firebender lands | כשף אש נוחת |
| `fire_heal_start` | 0.6 s | a soft inhale and a warm crackle rising into a hum (saw stack 220/330/440 LP 800 Hz swelling 0.5 s) | Fire healing begins | ריפוי באש מתחיל |
| `fire_heal_loop` | 4.0 s loop | a low warm pad (saw stack LP 700 Hz, slow 0.25 Hz breathing swell) with a soft crackle (quiet random clicks), volume 0.4 | Healing fire | אש מרפאת |
| `fire_heal_end` | 0.5 s | the hum fading | Fire healing ends | הריפוי באש נגמר |
| `fire_burn_out` | 0.6 s | a bright flash-sizzle (noise HP 2 kHz burst 80 ms + sizzle 0.3 s) and a sigh | Poison burned out | הרעל נצרב |
| `lightning_charge` | 1.2 s | a rising crackling electric whine: sine sweep 180→2400 Hz over 1.2 s with crackle bursts (random clicks 2–8 kHz) every 150 ms growing denser, plus soft "zip" sparks (short sine chirps 3→6 kHz, 20 ms) during the circles | Lightning charges | ברק נטען |
| `lightning_hold_loop` | 2.0 s loop | a tense crackling hum (sine 2400 Hz + saw 120 Hz LP 500 Hz + clicks 30 per s), pitch slowly rising 1.0→1.05 over 3 s, volume 0.5 | Lightning held | ברק מוחזק |
| `lightning_strike` | 1.2 s | a deafening crack: a 2 ms click → a broadband burst (white noise, 15 ms, HP 500 Hz) → a sub thump (sine 40 Hz, 0.15 s) → a 0.8 s rumble tail (brown noise LP 200 Hz decaying) with a short ring (sine 5200 Hz, 0.1 s); dual: a second crack 40 ms later; volume 2.0 | Lightning strikes | ברק מכה |
| `lightning_hit` | 0.5 s | a sizzling impact (noise HP 3 kHz burst 60 ms → sizzle 0.3 s) with a "zap" (sine 1800→600 Hz, 50 ms) and a short ring | Struck by lightning | נפגעו מברק |
| `lightning_arc` | 0.3 s | a short crackling zap (clicks 20 ms + sine chirp 4→1 kHz) | Lightning arc | קשת ברק |
| `lightning_cancel` | 0.4 s | sparks fizzling (clicks thinning over 0.3 s + a falling sine 2400→400 Hz) | Lightning fizzles | הברק מתפזר |
| `thunder_distant` | 1.5 s | a rumble (brown noise LP 120 Hz, slow 0.6 s attack, 0.9 s decay) — played at listeners 24–64 blocks away 0.6 s after a strike | Distant thunder | רעם רחוק |
| `paralysed_loop` | 2.0 s loop | a buzzing crackle (square 120 Hz LP 1 kHz + clicks 25 per s), volume 0.4 | Paralysed | משותקים |
| `redirect_stance` | 0.6 s | a slow calm inhale (noise HP 1 kHz LP 3 kHz, 0.5 s) with a faint electric tension (sine 2400 Hz at −30 dB) | Redirection stance | עמידת הטיה |
| `redirect_loop` | 2.0 s loop | a very quiet high hum (sine 2400 Hz + sine 2405 Hz beating), volume 0.15 | Ready to redirect | מוכנים להטות |
| `redirect_pass` | 0.6 s | the strike's crack (short) → a rising electric whoosh through the body (noise BP 400→4000 Hz, 0.25 s, with a sine sweep 300→3000) → a 40 ms pause → the crack again | Lightning redirected | הברק הוטה |
| `redirect_fail` | 1.0 s | the strike crack + a sizzling convulsion (clicks 60 per s, 0.5 s, with a formant "uh" grunt) + a body fall thud | Redirection fails | ההטיה נכשלה |
| `redirect_ground` | 0.9 s | the pass's whoosh → a deep earth thud (sine 45 Hz, 0.2 s) with a crackle spreading (clicks fading over 0.4 s with LP falling) and a soft gold chime (sine 1568 Hz, 0.3 s) | Lightning grounded | הברק הוארק |
| `fire_master_loop` | 15 s loop (stream) | a faint crackle (quiet random clicks 1–3 kHz, 4 per s) with a warm low hum, volume 0.1 | Embers | גחלים |

**Count:** shared 14, Earth 53, Air 50, Water 60, Fire 55 = **232 sound events** (plus 2–3 takes for ~40 of them:
about 310 files). S0 registers every event of a release with the placeholder; the release's sound lane renders them.

---

## 7. Music

All music is **original, written for the mod**, "in the spirit of" the shows' sound (the show's score leans on
East-Asian and Inuit-inspired instruments: taiko and drum ensembles for the Earth Kingdom, flutes and wind for the Air
Nomads, the tsungi horn and brass for the Fire Nation, strings and water sounds for the Water Tribes) **without any
melody, bass line or chord progression of the shows**. Every instrument is **synthesised** (`sfx.py` instruments:
`taiko` = modal drum (60/95 Hz) with a skin noise burst; `shakuhachi` = a breathy sine with noise and pitch wobble;
`erhu` = a sawtooth with a 6 Hz vibrato and a narrow BP body resonance; `tsungi` = a brassy saw stack with a slow
portamento and a formant "aw"; `bells` = modal; `koto` = `karplus` plucks; `pad` = detuned saw stack LP; `drops` = sine
pops with reverb). Rendered as mono 44.1 kHz Ogg, `stream: true`, and **as stems**, following Enhanced's boss-music
pattern (`HelaMusic`: several stems started together, layers fading in by phase, volume by distance, the Music source,
the client option `music`).

| Theme | Key / tempo / length | Instruments and shape | Stems | When it plays |
|---|---|---|---|---|
| **Earth: "Rooted"** (`music.earth`) | D minor pentatonic, 72 BPM, 64 s loop | taiko ensemble in a 4/4 with a heavy beat 1 and a rolling fill every 8 bars; a low drone (pad, D1); a koto motif of five notes rising stepwise then falling a fifth (original); a stone "clack" percussion (modal 1.2 kHz) on the off-beats | A drums, B drone + clacks, C koto motif | the Earth temple (suggestion 8.4), the Earth awakening (a 12 s sting cut from it: `music.earth.sting`), and **optionally** (`music.combatThemes`, default off) stem A while an earthbender within 24 blocks is in a fight (hit or hurt in the last 8 s) |
| **Air: "Ba Gua"** (`music.air`) | G major, 6/8 at 96 BPM, 72 s loop | a shakuhachi-like flute melody in long arcs (an original 8-bar tune, mostly stepwise with one leap of a sixth), wind chimes (random bell notes of the scale, 1 per 2 s), an airy pad swelling with the 6/8 pulse, a soft frame drum on 1 and 4 | A pad + chimes, B flute, C drum | the Air temple, the Air awakening sting, combat option |
| **Water: "Tide"** (`music.water`) | A minor, 58 BPM, 80 s loop | an erhu-like lead with slow slides (an original 16-bar melody that falls, pauses, rises), water drops as percussion (sine pops with long reverb), a bowed-string pad, a slow ocean swell (filtered noise) under everything; on **full-moon nights** stem D (a high bell counter-melody) fades in | A swell + drops, B pad, C erhu, D moon bells | the Water temple, the Water awakening sting, the full moon (stem D alone, very quiet, for waterbenders outdoors: `music.moonBells`, default on), combat option |
| **Fire: "Sozin's March"**... no: **"Breath of the Dragon"** (`music.fire`) | E minor, 112 BPM, 64 s loop | a tsungi-horn-like brass lead (an original march motif of four bars: two short notes, a long, a fall), driving drums (taiko + a sharp snare-like noise burst), a low ostinato (saw bass on E and B), crackling fire as a texture; Azula's look swaps the lead to a colder, higher brass (`music.fire.blue` stem variant) | A drums + bass, B brass lead, C texture | the Fire temple, the Fire awakening sting, combat option |
| **Bosses (suggestion, 8.5)** | one per boss, 3 stems each, as Enhanced's Hela | | | in the boss fight |
| **Menu / Element Choice** (`music.choice`) | a 30 s quiet piece: all four motifs hinted (two bars each) over a shared pad | | | the Element Choice screen (fades under the choice) |

Music never plays over vanilla music by accident: `SelectMusicEvent` sets the vanilla music to null while a theme plays
(Enhanced's `onSelectMusic` pattern), and themes crossfade over 2 s (`RoadMusicPlayer`'s recipe).

**How it is checked (Enhanced's 15.15.3):** every recipe rendered and inspected as a spectrogram and waveform, peak ≤ −1
dBFS after encoding, loudness and >12 kHz energy measured and logged by `render.py`, loops played across the seam ten
times, downbeats on the first sample of each loop, stems trimmed 6–8 dB under the effects; subtitles checked by
`SoundAssetTests` and `DataTests`; every sound heard in a real client at 1, 8 and 24 blocks, in the four elements' fights,
with the music on and off.
