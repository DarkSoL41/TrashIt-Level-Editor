# TRASH IT — Level Editor (v0.5)

A fan-made, browser-based level editor for the DOS game **Trash It**, built entirely from a reverse-engineered understanding of the game's own binary level files. Open a level, look at (and edit) its bricks, objects, durability data, sprites, and background, then export files that are byte-compatible with the original game.

No installation, no build step — it's a single self-contained HTML file. Open it in any modern browser and go.

> This is an unofficial fan tool. It is not affiliated with, endorsed by, or supported by the original developers/publisher of *Trash It*.

---

## What this is

*Trash It* stores each level as a handful of binary files (`.G2`, `.G2R`, `.I`, `.WAM`, `.PAL`, `.COL`, `.OB`, `.SPR`, `.SCN`) spread across the game's `LEVELS` and `SPR` folders. This editor:

- **Reads** those files directly in your browser (no server, no upload — everything happens locally on your machine).
- **Renders** the level exactly as the game would draw it: bricks, background, and objects (enemies, pickups, mechanisms, hazards, teleports, the exit, rocket parts, etc.), using the game's own sprites and palette.
- Lets you **inspect** every brick and object: durability, material, type-specific fields, and known/confirmed behavior notes gathered from real in-game testing.
- Lets you **edit**: move, add, or delete bricks and objects; change per-object fields; rename object types and fields with your own labels (saved locally in your browser).
- **Exports** only the files you actually changed, ready to drop back into the game's folders.

For the full technical breakdown of the level file formats, the object type catalogue, and everything discovered about the game engine's quirks and limitations along the way, see **`TECHNICAL_REPORT.md`** in this same package.

---

## Quick start

1. Open `TrashIt_Level_Editor.html` in a modern desktop browser (Chrome, Firefox, or Edge — anything reasonably recent works).
2. Click **"Open level files…"** (or drag & drop) and select the files of one level from your game's `LEVELS` and `SPR` folders. At minimum you need:
   - `XX.G2`, `XX.G2R`, `XX.I`, `XX.WAM`, `XX.PAL`
   - Recommended in addition: `XX.COL`, `XX.OB`, `XX.SPR`

   (`XX` is the level's key, e.g. `9L`, `3C`.) You can also select **all files from all levels at once** — the editor groups them by name automatically and lets you switch between levels with the dropdown at the top.
3. Use the **View** mode to browse the level: hover bricks to see their durability/material info in the sidebar, and check the **Sprites** / **Background** tabs to look at the raw `.SPR`/`.SCN` assets.
4. Switch to **Edit** mode to place, move, or delete bricks and objects. Click an existing brick/object to select and edit it; click an empty spot with a brush selected to place a new one.
5. When you're done, open the **Export** section in the sidebar — it lists exactly which files were changed and gives you a single button to download all of them at once. **Replace all of the listed files together** in the game's level folder; some of them are interdependent (e.g. `.G2`/`.G2R` must stay in sync) and replacing only part of the set can corrupt the level.

That's it — no build tools, no installation, just a file to open.

---

## Language

The editor has a language switch (**EN** / **RU**) in the top-right corner of the header. English is the default; your choice is remembered in the browser for next time.

---

## A note on caution

Some object types have engine-level quirks that can make a level **fail to load entirely** if misused (for example, certain "hidden in a crate" objects need to actually sit on top of a valid crate, and a few object types only work on specific level themes). The editor surfaces what's currently known about these quirks directly in each object's info panel, and defaults new placements to the safer configuration where possible — but always keep a backup of your original level files before overwriting them, and test changes in the actual game.

Editing is generally low-risk if you only change positions and confirmed fields; adding brand-new object instances of a type whose full field layout isn't 100% confirmed carries more risk (the editor flags this in the object's info panel).

---

## Credits

**DarkSoL** (Discord: `darksol41`) — reverse engineering, in-game testing, debugging, and direction of the whole project.
**Claude** (Anthropic) — AI-assisted static analysis, format decoding, statistical verification, and editor implementation, working alongside DarkSoL throughout.

*Trash It* is the property of its original developers/publisher. This editor and its accompanying documentation are unofficial, fan-made tools created for preservation, modding, and learning purposes.

---

## License / disclaimer

No warranty of any kind. Use at your own risk, and always keep backups of your original game files before replacing anything with edited output from this tool.
