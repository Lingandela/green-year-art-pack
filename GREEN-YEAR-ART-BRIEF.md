# GREEN YEAR — Art & Systems Pack
**For the Grok Bot team.** Screenshot catalog + how the art is built + where to change it.

Date: 2026-09-01  
Build: procedural 16-color VGA farm game. Web preview + self-contained `GREEN-YEAR.HTML` floppy (~263 KB / 1.44 MB).

Open `INDEX.html` in a browser to flip through every shot. This document is the code map.

---

## 1. What this game is (art contract)

GREEN YEAR is a **floppy-era farm sim** (Harvest Moon / early Story of Seasons feel) with one modern trick: a **rotatable voxel close-up** when you inspect a plant.

Non-negotiables:

| Rule | Why |
|---|---|
| **Exactly 16 colors** | VGA farm palette. Index is law. No RGB mixing except dither between two indices. |
| **No bitmaps on disk** | Every sprite is drawn with `fillRect` / 16×16 generated tiles. Keeps the file under 1.44 MB. |
| **Integer nearest-neighbor scale** | Internal buffer ~320×180, scaled 2–4×. No bilinear. |
| **Overworld and inspect share the palette + plant shapes** | Inspect is higher detail of the same plant, not a different art style. |
| **Procedural + seeded** | `mulberry32` from the farm seed. Same seed = same plants. |

If you replace art, stay inside these rules unless the product owner drops the floppy constraint.

---

## 2. Palette (do not add a 17th color)

File: `src/game/palette.ts` — array `PAL[0..15]`.

| # | Name | RGB | Hex | Used for |
|---|---|---|---|---|
| 0 | ink | 26,18,16 | `#1a1210` | outline, night, HUD fill |
| 1 | dark soil | 60,36,24 | `#3c2418` | shadow soil, night wood |
| 2 | dirt | 107,62,36 | `#6b3e24` | path, tilled, rug center |
| 3 | light dirt | 138,90,50 | `#8a5a32` | hoe blade, highlight dirt |
| 4 | dark grass | 44,72,24 | `#2c4818` | night grass, dark leaf |
| 5 | grass | 74,122,40 | `#4a7a28` | day grass, stems |
| 6 | light grass | 124,176,72 | `#7cb048` | leaf highlight, energy bar |
| 7 | straw | 200,184,120 | `#c8b878` | hat brim, thatch |
| 8 | wood | 90,58,32 | `#5a3a20` | posts, trunks, pants |
| 9 | light wood | 140,92,52 | `#8c5c34` | rails, crate |
| 10 | cream | 208,200,176 | `#d0c8b0` | skin, Glim fruit, pillow |
| 11 | paper | 236,230,208 | `#ece6d0` | HUD text, moon fruit, highlights |
| 12 | sky | 58,90,140 | `#3a5a8c` | day sky, Vesper fruit, can |
| 13 | light sky | 122,156,204 | `#7a9ccc` | sky hi, water, moonlace leaf |
| 14 | rust | 196,76,40 | `#c44c28` | roof, shirt, Marrow fruit |
| 15 | gold | 232,200,72 | `#e8c848` | hat, Heirloom, sparkle |

Night is **not a new palette**. `shadeIndex()` in `palette.ts` remaps each index darker. Dusk sky uses rust (14) + gold (15).

**To recolor the farm:** edit `PAL`. Every draw call uses indices 0–15, so one edit retints the whole game.

---

## 3. How a frame is drawn

```
Game.frame (src/game/game.ts)
  resize → internal buffer pw × ph (pixel canvas)
  draw:
    title  → drawTitle (cottage silhouette + bitmap font)
    farm   → drawSkyPix + drawScene + particles + drawFarmHudPix
    inspect→ drawInspect (voxel projector)
  blit pixel canvas onto the big canvas at integer scale
  DOM overlay (src/components/game-overlay.tsx + src/game/hud.css)
```

Two cameras:

- **Yard:** world tiles, `TILE = 16`. Camera follows the player (`want ≈ 22` tiles across).
- **Home:** not a camera. The 8×6 room is laid out with `HOME_TILE = 26` so the cottage fills the frame.

Y-sort: house, stall, trees, plots, player are pushed into a sprite list and sorted by world `y` (feet).

---

## 4. File map (what to open when you want to change X)

| You want to change | File | Function / data |
|---|---|---|
| Colors | `src/game/palette.ts` | `PAL`, `shadeIndex`, `skyIndex` |
| Ground tiles (grass, dirt, path, floor, wall) | `src/game/pix.ts` | `tiles()` |
| Farmer, house, stall, trees, plots, HUD strip | `src/game/draw/farm.ts` | `drawPlayerAt`, `drawHouse`, `drawStall`, `drawTree`, `drawFarmPlant`, `drawFarmHudPix` |
| House / stall / plot **positions** | `src/game/world.ts` | `MAP`, `HOME`, `TREES`, `plotPos` |
| Crop shapes, leaf/fruit/stem colors, names | `src/game/crops.ts` | `CROPS`, `CROSSES` |
| Inspect voxel models | `src/game/voxels.ts` | `buildPlantVoxels`, `paintFruit` |
| Inspect frame (soil mound, name plate, genes) | `src/game/draw/inspect.ts` | `drawInspect` |
| Bitmap 6×6 font | `src/game/font.ts` | `GLYPHS` |
| Overlay (title, shop, tools, d-pad) | `src/game/hud.css`, `src/components/game-overlay.tsx` | `.gy-*` |
| Particles | `src/game/juice.ts` | `burst` — palette pixels only |
| Day length / lighting | `src/game/game.ts` `DAY_LEN = 110`, `palette.timeOfDay` | dusk at 0.62, night at 0.82 |
| Floppy bundle | `scripts/write-disk.mjs` | inlines JS + `hud.css` into `public/GREEN-YEAR.HTML` |

---

## 5. World layout (tile units, origin top-left)

Map is **24 × 16** tiles.

```
y=1   north fence + trees
y=3.4 cottage (footprint ~2.4..6.4 x, 1.7..4.95 y)
y=5   PATH  (x 4→20)  door at (4.4, 5.05)   stall at (19.4, 4.15)
y=5.75 porch / spawn-in-front-of-door
y=6.8  player spawn (6.2, 6.8)
y=7.5  plot grid 4×3, origin (7.5, 7.5), step (2.6, 2.5)
y=14   south fence
```

Interior (`HOME`): 8×6. Bed (2.2, 2.2), table (6.15, 2.35), rug (4.0, 3.5), door (4.0, 5.2), spawn (4.0, 3.7).

**Collision:** house body, stall counter, tree trunks (r ≈ 0.42), interior table + bed. Plots are walk-through (Stardew-like).

**To move a building:** change `MAP.*` in `world.ts` **and** the matching draw in `farm.ts` uses those same constants, so the sprite follows. Path tiles are still **hardcoded** in `drawYard` (`ty === 5`, `tx === 4`, etc.) — if you move the house, update the path mask too.

---

## 6. Sprites (how they are made)

There are **no PNG character sheets**. Everything is rectangles.

### Farmer (`drawPlayerAt`)
~16 px tall, origin at feet.

- Straw hat (15 + brim 7)
- Face (10) or back-of-head (8)
- Rust shirt (14) + cream button
- Wood pants (8), 4-frame walk bob
- Tool: hoe (stick + dirt blade) or can (blue + spout). Hand has no extra sprite.
- 4 dirs: 0 down, 1 left, 2 right, 3 up

Swing: `actor.swing` 1→0 over ~0.22s. Hoe arcs; can tilts and drips index 12/13.

**To make a better farmer:** rewrite `drawPlayerAt` / `drawHeldTool`. Keep ~16 px or the house/stall will be covered again (we already shrank a 32 px giant).

### Cottage (`drawHouse`)
48×24 wall, stepped rust roof, chimney puff, south door, two windows (lit at night or even days).

### Stall (`drawStall`)
Striped awning 14/11, counter, crates, pixel text `STALL`, merchant (purple-ish 12 coat, rust hat).

### Trees (`drawTree`)
Trunk 8, oval canopy 5/6, 1 px sway.

### Plots
Unlocked: 2×2 dirt/wet/grass tiles + wood frame.  
Locked: two posts.  
Plant overlay: `drawFarmPlant` — stem + shape fruit. Stages 0–4 from `stageOf()`.

### Interior
Wood wall row, floor tiles scaled to 26 px, rust rug, bed (cream pillow + rust blanket), table + candle flicker, south door with dirt threshold. Night window uses sky index.

---

## 7. Crops — overworld vs inspect

Each crop is a row in `CROPS`:

| id | Name | Shape | Leaf | Fruit | Stem | Ripe nights | How you get it |
|---|---|---|---|---|---|---|---|
| glim | Glim | berry | 6 | 10 | 5 | 2 | start (3 seeds) |
| marrow | Marrow | squash | 5 | 14 | 4 | 4 | stall 18g after first sale |
| vesper | Vesper | bell | 4 | 12 | 4 | 5 | stall after 3 species or 40g earned |
| thatch | Thatch | grain | 6 | 15 | 5 | 3 | stall after first sale |
| pip | Pip | root | 5 | 8 | 4 | 4 | stall (hybrid unlock) |
| amber | Amberdrop | berry | 6 | 15 | 5 | 3 | Glim × Marrow |
| moonlace | Moonlace | bell | 13 | 11 | 6 | 4 | Glim × Vesper |
| bloodroot | Bloodroot | root | 4 | 14 | 8 | 5 | Marrow × Vesper |
| strawgold | Strawgold | grain | 6 | 15 | 5 | 3 | Thatch × Glim |
| ironvine | Ironvine | vine | 5 | 14 | 8 | 5 | Thatch × Marrow |
| glassheart | Glassheart | heart | 13 | 11 | 6 | 6 | Pip × Moonlace |
| heirloom | Heirloom | crown | 6 | 15 | 9 | 7 | Glassheart × Ironvine — **endgame** |

**Overworld** (`drawShapeFruit`): 8 tiny 2D silhouettes so the field reads at 16 px.

**Inspect** (`voxels.ts` `paintFruit`): 3D voxel blobs, drag to orbit (`rotY`, `rotX`). Same shape names. Genes tweak size / extra fruit / fruit hue (`fruitColor`: gene color 0→cream, 1→base, 2→rust, 3→gold).

Growth stages (inspect + overworld): `SEED 0 → SPROUT 1 → LEAF 2 → BUD 3 → RIPE 4`.

**To add a crop:** `types.ts` `CROP_IDS`, `crops.ts` `CROPS` + maybe `CROSSES`, a `paintFruit` branch if you need a new shape, shop unlock in `sim.ts`. Stay in 12 if you can — almanac and floppy copy assume 12.

---

## 8. Overlay / HUD (not canvas)

Canvas HUD (left): `DAY N`, gold, energy bar, quest text. Drawn in `drawFarmHudPix`.

DOM overlay (`hud.css`):

- Title: Boot / Continue / Write Disk
- Top-right: Sound, Book, Stall
- Bottom: crop picker, Hoe / Can / Hand, context act, Sleep
- Left: N/W/E/S pad (touch)
- Inspect: Look away + Harvest
- Shop / Almanac panels

Font: Silkscreen on web (Google), floppy falls back to system monospace if Silkscreen isn’t installed.

**To restyle chrome:** `src/game/hud.css`. Disk build **inlines this file**, so keep it self-contained (no Tailwind, no extra font URLs).

---

## 9. Systems the art has to support

These are live, not mockups:

1. **Tools** — Hoe tills (row hoe upgrade), Can waters (wide / field upgrades), Hand plants / harvests (scythe = harvest from yard).
2. **Energy** — 20 + extra plots. Sleep restores. Overlay Sleep is a shortcut; bed is the diegetic one.
3. **Breeding** — two flowering adjacent plots, sleep, chance of a cross seed (`CROSSES`). Same-species sleep can mutate genes.
4. **Shop** — seeds, plot deeds, can/hoe/scythe/scarecrow. Sell bin.
5. **Quest spine** — Till → Plant → Water → Sleep → Harvest → Sell → Buy Marrow → pair → cross → expand → can → Vesper → Moonlace → Pip → Glassheart → Thatch → Ironvine → Heirloom.
6. **Day clock** — `dayT` 0..1 over 110s. Lighting + walk-slow after 0.72.
7. **Floppy** — Write Disk downloads `GREEN-YEAR.HTML`. Must keep fitting 1.44 MB.

---

## 10. Known art gaps (please mark these)

1. **Farmer** — readable at 16 px, still rectangle-VGA. Face is two pixels. Walk cycle is bob + leg swap, not a sheet.
2. **Overworld plants vs inspect** — inspect is the showpiece; field sprites are silhouettes. They share *shape family*, not voxel projection.
3. **Cottage vs stall** — both rust-red. Stall has stripes + STALL label; cottage could use a cooler roof or trim so they don’t rhyme.
4. **Interior** — room is a wood box floating in soil void. No exterior visible through the door. Ceiling is a cream strip.
5. **Locked plots** — two posts, easy to miss in grass.
6. **Trees** — ovals. Fine at distance, mushy up close.
7. **Path** — 16×16 dirt tiles, T-junction coded by tile coords, not a painted road.
8. **Night** — index remap + moon. No interior lamp except the candle.
9. **Overlay vs pixel HUD** — two layers. Mobile pad is HTML, not a pixel d-pad.
10. **Font** — 6×6 caps only on canvas; overlay is Silkscreen. Two type systems.
11. **Merchant** — 12 px NPC, no idle besides 1 px bob.
12. **Scarecrow** — only appears after purchase; empty yard has no landmark between path and plots.

**What we already fixed and should not regress:** farmer scale (~16 px, feet origin), HUD on the left, enter-house not auto-Leave, path T to stall, STALL sign, quest copy says bed/stall not cot/cart.

---

## 11. How to propose a change (for the bot team)

When you suggest art, say **which file + function** and **whether it stays 16-color procedural**.

Good:  
> “`drawHouse` — make the roof index 12 (sky) with 14 trim so it doesn’t match the stall. Keep 48×24 footprint so collision in `world.ts` still matches.”

Bad:  
> “Make it look more like Stardew.” (that’s a PNG atlas + 32+ colors + size budget break)

If you want sprite sheets: they must be **indexed to this 16**, magenta-keyed, tiny, and inlined or the floppy download grows. Prefer rewriting the draw functions.

Disk rebuild after art: `node scripts/write-disk.mjs` → `public/GREEN-YEAR.HTML`. Target **< 1.44 MB** (currently ~263 KB).

---

## 12. Endgame path

```
Glim (start)
  + Marrow (buy)  → Amberdrop
  + Vesper (buy)  → Moonlace
  + Thatch (buy)  → Strawgold
Marrow + Vesper   → Bloodroot
Thatch + Marrow   → Ironvine
Pip (buy) + Moonlace → Glassheart
Glassheart + Ironvine → Heirloom   ← harvest this to close the year
```

Almanac shows `Parent x Parent` once discovered. Inspect is where the player is supposed to fall in love with a new cross — that screen is the art north star.

---

## 13. Questions for the team

1. Keep strict 16-color procedural, or allow a small indexed sprite atlas for farmer + house only?
2. Should overworld plants be tiny voxel projections (cost/CPU) or stay 2D silhouettes?
3. Cottage roof color / silhouette — more “home,” less “red box”?
4. Interior: keep the framed room, or a bigger cutaway with visible yard through the door?
5. Locked plots: deed sign, faded frame, or leave the posts?
6. Overlay: stay HTML (readable, big hit targets) or pixel-chrome to match the canvas?
7. Any crop silhouette that doesn’t read at field distance?

Mark up the PNGs. Point at a file in §4 when you want a change built.
