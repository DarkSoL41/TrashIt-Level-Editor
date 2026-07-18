# TRASH IT — Level Format Technical Report

**Version:** matches Level Editor v0.5
**Scope:** binary level file formats used by the DOS game *Trash It*, reverse-engineered from the game's executable (`G.EXE`, DOS4GW LE) and from direct in-game testing.
**Author:** DarkSoL (Discord: darksol41), with AI assistance from Claude (Anthropic)

This document describes how *Trash It* level files are structured, what has been confirmed through testing, what remains unclear, the engine limitations discovered along the way, and the methodology used to get there. It is meant as a reference for anyone continuing this reverse-engineering work or building tools on top of it.

---

## 1. Overview

Each level in *Trash It* is stored as a set of files sharing the same base name (the level key, e.g. `9L`, `3C`, `7S`) spread across two folders: `LEVELS` and `SPR`. The letter suffix of the level key generally encodes the **theme** (see §5), and the leading digit(s) an index within that theme.

A full level is made up of the following files:

| Extension | Contents | Status |
|---|---|---|
| `.G2`  | Brick/tile bitmap definitions (the raw pixel blocks used to build the level) | Understood, editable |
| `.G2R` | Per-block metadata paired with `.G2` (must stay in sync — same block count) | Understood, editable |
| `.I`   | Tilemap / level layout (which `.G2` block goes where) | Understood, editable |
| `.WAM` | Level header: declared width/height bounds and other header fields | Understood (partially — see §6) |
| `.PAL` | Level color palette (VGA 6-bit-per-channel, 256 colors) | Understood, editable |
| `.COL` | Per-block durability/collision data | Understood, editable |
| `.OB`  | Object list (enemies, hazards, pickups, mechanisms, teleports, etc.) | Understood — see §4, main focus of this report |
| `.SPR` | Sprite sheets (shared across levels of the same theme, referenced by name, e.g. `JACKS.SPR`, `BON.SPR`) | Understood (frame format decoded) |
| `.SCN` | Background image for the level | Understood, editable |
| `.OBT` | Auxiliary object-related table (purpose not fully mapped) | Not analyzed in depth |
| `.STP` | Large auxiliary file (65536 bytes, fixed size) — likely a per-tile or per-pixel lookup table | Not analyzed in depth |
| `.SDE` | Auxiliary level data | Not analyzed in depth |

The editor built alongside this report can open, view, and edit the first set (`.G2`, `.G2R`, `.I`, `.WAM`, `.PAL`, `.COL`, `.OB`, `.SPR`, `.SCN`) and export modified binaries that are byte-compatible with the game.

---

## 2. Sprite format (`.SPR`)

All `.SPR` files share one format, decoded as follows (all integers little-endian):

```
u32 total          — total file size in bytes (self-referential)
u16 nfr             — number of frames in this sprite sheet
u16 <reserved>       — padding to align the offset table to a 4-byte boundary (observed value: 8; purpose not otherwise used)
u32[nfr] offsets     — one offset per frame, relative to byte 8 (i.e. frame N starts at file byte 8 + offsets[N])

For each frame, at its start offset:
  i16 w, h            — frame width/height in pixels
  i16 hx, hy          — hotspot/anchor offset (typically negative, placing the anchor near the frame's bottom-center)
  then h rows of run-length-encoded pixel data:
    u8 skip           — unused/reserved per-row byte (observed but not required for correct rendering)
    u8 rl              — row length in bytes, including this header (i.e. rl-2 pixel bytes follow)
    (rl-2) x u8 pixels — palette indices; index 0 is treated as transparent
```

Each pixel row is horizontally centered within the frame's declared width (`xstart = (w - pixelCount) / 2`) — this was necessary to get pixel-perfect rendering matching the game.

Colors come from the level's `.PAL` file: 256 entries of 3 bytes each (R, G, B), each channel in the 0–63 VGA range. To get standard 0–255 RGB, scale by `255/63`.

---

## 3. Level object list (`.OB`)

### 3.1 Header and record structure

```
u16 cnt              — number of object records that follow
records...           — cnt records, each of variable length depending on its type
```

Each record starts with:

```
i16 type             — object type ID (see §4 for the catalogue)
i16 x, y              — position in level pixel coordinates
i16[...] tail         — type-specific extra fields, length depends on type
```

**There is no universal record length or end-of-record marker.** Each object type has its own fixed record length (in 16-bit words, including the `type`/`x`/`y` header), and the parser must know this length table to walk the list correctly. Getting a single type's length wrong desynchronizes the parser for every object that follows it in the file — this was the source of several bugs described in §6.

One important discovery: files do **not** reliably end with a special sentinel value. An earlier hypothesis (that the byte pattern `0x7FFF` marks end-of-records) turned out to be a misreading of a corrupted parse caused by a wrong record length (see §6.1) — it was actually just the last data field of a legitimate object record. The current parser walks records purely by length and stops when the declared `cnt` is reached (or the file runs out, which is treated as a parsing error/warning in the editor).

### 3.2 Ambiguous/undetermined record lengths

For a handful of types (`41`, `42`, `43`, `44` at the time of writing) the exact record length has not been independently confirmed — the editor falls back to a best-effort guess derived from statistical analysis of the byte stream (assuming the most common length that keeps the overall object count consistent). Levels using these types are flagged in the editor with an "ambiguous" warning. **Exporting such a level unmodified is always safe** (the original bytes are preserved verbatim); only *editing* objects of these unconfirmed types on such a level carries some risk.

---

## 4. Object type catalogue

Below is the full catalogue of object types identified as of this report. "Confirmed" means the behavior was verified in-game; "Unclear"/"Risky" means the type is understood at the byte level but its in-game behavior is unconfirmed or only partially confirmed.

| Type | Name | Status | Notes |
|---|---|---|---|
| 0 | Swing (stand) | Confirmed | Field 11: 1=wooden (`CSAW.SPR`), 2=metal (`BCSAW.SPR`). Only the stand/support sprite was found — the swinging beam itself doesn't appear to be a separate sprite file. |
| 1 | Swing counterweight | Confirmed | Sprite `LEAD.SPR`. |
| 2 | Respawn point (P1) | Confirmed | Respawn after being crushed + return point for a lost hammer. Sprite: spinning striped cylinder, `FLAG.SPR`. |
| 3 | Respawn point (P2, co-op) | Confirmed | Analog of type 2 for player 2. Appears once per level, always near types 2/4/5, only in co-op themes (B/K/D). |
| 4 | Respawn point (P3, co-op) | Confirmed | Analog of type 2/3 for player 3. |
| 5 | Respawn point (P4, co-op) | Confirmed | Analog of type 2/3/4 for player 4. |
| 6 | Cannon | Confirmed | Field 11: 1=stationary (barrel `CFIR.SPR` + base `CFIX.SPR`), 2=mobile/on a wheel (barrel `CFIR.SPR` + wheel `CWHL.SPR`). |
| 7 | Cannonball | Confirmed | Sprite `CFIR.SPR` (a single round-ball frame). Field 9: 2=large (default), 1=small. |
| 8 | Bag of cannonballs | Confirmed | Composite sprite: white neck + orange body, both from `DIS.SPR`. |
| 9 | Player 1 start | Confirmed | Sprite `JACKS.SPR`. |
| 10 | Player 2 start (co-op) | Confirmed | Analog of type 9. Same theme/clustering pattern as types 3/4/5 — sits right next to type 9. |
| 11 | Player 3 start (co-op) | Confirmed | Analog of type 9/10. |
| 12 | Player 4 start (co-op) | Confirmed | Analog of type 9/10/11. |
| 13 | Exit | Confirmed | Sprite `BELL.SPR` (3 stacked parts bottom-to-top: yellow base → pole → round bell). Field 9 — appearance mode: 1 = present from the start; 2 = invisible, appears after collecting all rocket parts (Launch theme, paired with field 12 = 327); 8 = invisible, appears after the vacuum collects a threshold % of trash (City theme, paired with field 11 = approximate % needed, usually ≥80). |
| 14 | Hammer spawner | Confirmed | Spawns hammers from inside a breakable block. Field 10 = direction (0/1) or count (2+); field 12 = behavior (0 = normal, sprite `TIMMY.SPR`; 1 = tricky; 2 = white, breaks blocks, sprite `TOMMY.SPR`). |
| 15 | Explosive cake/pie | Confirmed | Triggers on touch. Field 9: 2 = no candle (default), 1 = with a candle. Sprite `DYNA.SPR` (separate frames exist for the candle and ball variants). |
| 16 | *(unidentified)* | Risky | Invisible when placed; related to types 14/15/17 by proximity but not identified on its own yet. |
| 17 | Hammer spawner (golden) | Confirmed | Same field scheme as type 14, but with a big golden-capped hammer. Sprite `KTIMMY.SPR`. |
| 18 | Co-op object | Confirmed | Only appears in co-op mode. Sprite `TIMBIN.SPR` (frame 0 used as icon). |
| 22 | Spring/trampoline | Confirmed | Composite sprite: grey base + orange cap, both from `SUCKER.SPR`. |
| 23 | Forklift | Confirmed | Exact matching sprite not pinned down with certainty (`FORK.SPR` didn't visually match well). |
| 24 | Gears (forklift speed upgrade) | Confirmed | Sprite `FORK.SPR`, frame 12 (two meshed gear wheels). Field 9: 1 = visible immediately; 2 = must be hidden inside a crate/barrel (spawner logic like type 14) — placing it hidden without a crate underneath causes the **level to fail to load** (game behavior, not an editor bug). Record length is 11 words, confirmed by round-trip testing across all 147 levels; only 1 real instance of this type exists in the whole game (level `9L`), so the meaning of most tail fields beyond field 9 is unconfirmed. |
| 27 | Teleporter | Confirmed | Sprite `TELLY.SPR`. Pairing logic: field 11 = "outgoing" ID (where this one sends the player), field 10 = "receiving" ID. If teleporter A has field11 = N and teleporter B has field10 = N, stepping into A sends the player to B. |
| 28 | Red button | Confirmed | Sprite `REDBUT.SPR` (red variant). |
| 29 | Green button | Confirmed | Analog of type 28. Sprite `REDBUT.SPR` (green variant). |
| 30 | Blue button | Confirmed | Analog of type 28/29. Sprite `REDBUT.SPR` (blue variant). |
| 31 | UFO (unconfirmed) | Unclear | Possibly a UFO enemy (`UFO.SPR` exists in the game's sprite set), but not confirmed in-game. Complex composite record (length 24), structured as several nested sub-records of the form `9,X,9,Y` plus a `13,Z,6,1` block. Statistics over all 36 instances in the game: fields 1–4 and the structural markers (`9`,`13`,`6`) are always constant; a handful of fields vary, always taking power-of-two values (1/2/4/8/16/32/64/128 — consistent with bitmasks, likely for movement direction/pattern), plus one trailing field that's almost always 0 (occasionally 1 or 3). |
| 32 | Spiky enemy (green) | Confirmed | Sprite `SPK.SPR`. Cannot appear on the first level theme (an engine restriction). Field 9 = behavior (1 = just runs around; 2 = occasionally releases spikes; 4 = visually identical to 2, difference not fully understood). Field 11 = visibility (1 = runs openly from the start; 2 = hidden inside a block, usually a barrel, spawner logic like type 14). |
| 33 | Walking bonus | Confirmed | Sprite `BON.SPR`. Field 9 (rawIdx5) = visibility (1 = runs around openly right away; 2 = hidden in a crate, spawner logic like type 14). Field 11 (rawIdx7) = bonus type inside it: 1 = clock (time bonus), 2 = running hammers (count set by field 12: 0 and 1 both give one hammer, 2 gives two, etc.), 4 = super-vacuum bonus, 8 = trash-collecting bonus. Two instances found (levels `7S`, `8S`) have field11 = 1 (clock) *and* field12 = 10 (not 0) — meaning field 12 is not ignored for the clock bonus either; it may set the bonus's duration in that case. Across all 48 instances in the game, fields 1–5 are always constant (`2,1,0,0,3`) and field 7 is always `5`. |
| 34 | *(unidentified)* | Unclear | Only a single instance exists in the entire game (level `8S`, coordinates 1136,808). Nothing to compare statistically; in-game appearance not yet confirmed. |
| 35 | Rocket part: nose | Confirmed | Launch theme. The very top of the rocket (nose cone with antenna). Sprite `APOLLO.SPR`, frame 8. |
| 36 | Rocket part: section 2 | Confirmed | Section directly below the nose (type 35). Sprite `APOLLO.SPR`, frame 7. |
| 37 | Rocket part: section with clamp | Confirmed | Below type 36, has a side clamp/latch detail. Sprite `APOLLO.SPR`, frame 6. |
| 38 | Rocket part: transition cone | Confirmed | Dark transition cone below type 37. Sprite `APOLLO.SPR`, frame 5. |
| 39 | Rocket part: section with lamps | Confirmed | Panel with side "lamp"/tank details, below type 38. Sprite `APOLLO.SPR`, frame 2. |
| 40 | Rocket part: section with emblem | Confirmed | Panel with a red round emblem, below type 39. Sprite `APOLLO.SPR`, frame 3. |
| 41 | Rocket part: section with lamps 2 | Confirmed | Visual duplicate of type 39, below type 40. Sprite `APOLLO.SPR`, frame 4. |
| 42 | Rocket part: flame skirt | Confirmed | Skirt with red flame detail, below type 41. Sprite `APOLLO.SPR`, frame 1. |
| 43 | Rocket part: base (nozzles) | Confirmed | The very bottom — base with 3 engine nozzles. Sprite `APOLLO.SPR`, frame 0. |
| 44 | Crawler robot | Confirmed | Automatic tracked robot that assembles the rocket parts (Launch theme). Sprite `CRAWL.SPR`, frame 0. Only works on the Launch theme, and **requires a platform block underneath it** — without one, or on another theme, the level fails to load. |
| 45 | Bomb enemy | Confirmed | Sprite `BOM.SPR`. Like type 32, cannot appear on the first level theme (engine restriction) — only works on certain themes. |
| 46 | 1UP (extra life) | Confirmed | Visually a golden Jack holding a hammer (see reference screenshots collected during testing). No matching real in-game sprite file was found among the game's assets; the editor uses a homemade placeholder icon (a gold "1UP" badge) instead. |

Types not listed above either haven't been placed in any real level (so nothing could be tested), or haven't been investigated yet.

The chain **35 → 36 → 37 → 38 → 39 → 40 → 41 → 42 → 43** represents the full rocket, top to bottom; collecting all of these on a Launch-theme level is what triggers the exit's field-13/mode-2 appearance behavior (see type 13 above).

---

## 5. Level themes

Level keys follow the pattern `<index><ThemeLetter>`, e.g. `9L`, `3C`, `7S`. The theme letter groups levels that share the same tileset/sprite pack and, importantly, the same **engine restrictions** on which object types are allowed to function (see §6.2). Observed theme letters include (non-exhaustive, inferred from testing rather than an authoritative source): `A`, `B`, `C`, `D`, `H`, `I`, `J`, `K`, `L` (Launch), `M`, `S` (City), `T`, and combinations used for co-op levels (`B`/`K`/`D`).

---

## 6. Engine limitations and bugs discovered

This section documents behavior of the **game itself** (not the editor) that was discovered through testing, along with editor bugs found and fixed during development.

### 6.1 Wrong record length for type 24 (gears) — historical bug, now fixed

Early versions of the editor used a length of 7 words for type 24 (gears). This was wrong on two separate occasions during development:

1. **First fix attempt (7 → 14 words):** it was noticed that on level `9L` — the only level in the game containing a type-24 object — the object immediately after it was corrupted into a phantom "type 3" object with suspicious near-origin coordinates. Extending the type-24 length to 14 words "fixed" this symptom, and the level appeared to end with a coincidental 2-word trailer `[0, 0x7FFF]`.
2. **Correct fix (14 → 11 words):** further inspection revealed that the "14-word" fix was itself wrong — it silently absorbed the start of the *next legitimate object*, an exit (type 13) at coordinates (164, 690), which made the exit disappear from the parsed object list entirely. The real length is **11 words**. With this fix, the object count matches the file's declared `cnt` exactly (34/34), the previously "missing" exit reappears, and the file's last word is confirmed to simply be the last tail field of that exit record — not a special end-of-file sentinel as previously assumed.

This episode also permanently disproved an earlier general hypothesis that the byte value `0x7FFF` (32767) marks an end-of-records sentinel in `.OB` files; that belief was an artifact of the wrong-length bug, not a real feature of the format. The current code carries a defensive check for `0x7FFF`/`-32768` values purely as a fallback safety net for genuinely malformed/edited files, not because it is known to have that meaning in real game data.

**Lesson for future work:** a wrong record length for any type doesn't just corrupt that one object — it silently reinterprets every subsequent object in the file, which can look like unrelated bugs (phantom objects, missing objects, objects that appear "moved") far away from the actual root cause. Always verify a type's length via round-trip testing across *all* levels containing that type, not just visual inspection of one level.

### 6.2 Theme-restricted object types

Several enemy/mechanism types only function on specific level themes, independent of whether the object is present in the file:

- **Type 32** (spiky enemy) and **type 45** (bomb enemy) cannot appear on the game's first level theme at all — this is an engine-level restriction, not a data problem. Placing them there via the editor does not crash anything, but the enemy simply won't show up in-game.
- **Type 44** (crawler robot) and the whole rocket-part chain (**types 35–43**) only work on the **Launch** theme.

### 6.3 Crash-on-load conditions (not editor bugs — verified game behavior)

- **Type 24** (gears) placed with field 9 = 2 ("hidden in crate") but **not** positioned on top of a valid crate/barrel object causes the level to **fail to load entirely** in the game. This is not a parsing error and not an editor crash — it's how the game itself behaves. Because of this, the editor's default placement value for field 9 on this type was deliberately set to 1 (visible) rather than 2, to avoid this trap for anyone placing a new instance through the toolbar.
- **Type 44** (crawler robot) has the same requirement — it must sit on a platform block, and only works on the Launch theme; violating either condition prevents the level from loading.

### 6.4 Object count is not a reliable proxy for "near a limit"

Level `9L` (which triggered the type-24 investigation above) has only 34 objects — nowhere close to the game's practical maximum; level `3L`, for comparison, has 94 objects, and several other levels exceed 60–80. So symptoms that look like "running out of some engine slot" should not be assumed to be an object-count ceiling without further evidence; in the one case investigated in this project it turned out to be a data-format bug instead (§6.1).

### 6.5 Order-independence of object records

New objects added through the editor are appended to the end of the object list; the game does not appear to depend on any particular ordering of objects within the file (aside from the exact record-length parsing constraint described above). Round-trip and placement-simulation tests confirm that inserting new records after any existing one, including after the last object, produces a structurally valid file that re-parses identically.

---

## 7. Methodology

The findings in this report came from a combination of:

- **Static byte-level analysis** of `.OB`, `.SPR`, and `.PAL` files using a custom Python decoder, run across all 147 levels shipped with the game, to build per-type statistics (which fields vary, what values they take, how many instances of each type exist and on which levels).
- **Round-trip verification**: every proposed record-length fix was validated by re-parsing every level in the game and checking that (a) the number of parsed objects matches the file's declared count exactly, and (b) rebuilding the file from the parsed structure reproduces the original bytes exactly, byte for byte. This was done both with an independent Python re-implementation and by running the editor's actual JavaScript `parseOb`/`buildOb` functions directly in Node.js against real level files, to make sure both implementations agree and neither hides a bug.
- **Sprite decoding and matching**: candidate sprite files were rendered to images (applying the level's palette) and visually compared frame-by-frame against reference screenshots taken from the actual game, to identify which `.SPR` file and frame index corresponds to which object type.
- **Direct in-game testing** performed by the human collaborator (DarkSoL): placing specific field-value combinations for a given object type into a level, loading it in the actual game, and observing the resulting behavior (or crash). This is the only source of ground truth for *behavioral* meaning (as opposed to byte-level structure) and is credited per-type in §4 as "Confirmed".

Where a finding could not be independently cross-checked (usually because only one instance of an object type exists across all 147 levels), this is called out explicitly rather than presented as certain.

---

## 8. Open questions / suggestions for further work

- **Type 16, 31, 34** remain unidentified or unconfirmed; type 31 in particular has a large, structured record that likely encodes multiple sub-points (possibly a patrol path or multi-stage spawn behavior) worth testing in-game.
- **Types 41–44** have record lengths derived from statistical inference rather than an independently confirmed source; if the engine's actual object descriptor table (referenced from `G.EXE`) can be located and read directly, this would remove the remaining uncertainty for these types once and for all.
- **`.OBT`, `.STP`, `.SDE`** files have not been reverse-engineered at all. `.STP` in particular is a fixed 65536-byte file per level, which strongly suggests some kind of per-pixel or per-tile lookup table (possibly a physics/collision heightmap or an animation-state table) — worth investigating if further object behaviors need explaining.
- The exact meaning of most **type 24** tail fields beyond field 9 is unknown, since only one real instance of this object exists in the entire game.
- A dedicated pass matching `field9=2` "hidden in a crate" objects (types 24, 32, 33) against the specific crate/barrel object they were placed on would help confirm exactly which underlying object types are valid "containers" for this mechanic.

---

## 9. Credits

Reverse engineering, in-game testing, and debugging: **DarkSoL** (Discord: `darksol41`).
Static analysis tooling, sprite decoding, statistical verification, and this report: AI assistance from **Claude** (Anthropic), directed and verified by DarkSoL throughout.

*Trash It* is the property of its original developers/publisher. This document and the accompanying level editor are unofficial, fan-made reverse-engineering tools, not affiliated with or endorsed by the original developers.
