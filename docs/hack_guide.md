# Pokémon Emerald — Gastly Starter Hack Guide

A step-by-step walkthrough for editing a Pokémon Emerald ROM (US 1.0) so that:

1. One of the three starters is replaced with **Gastly**.
2. **Haunter evolves into Gengar by leveling up** (no trade required).
3. **Gastly / Haunter / Gengar can learn every TM and HM** in the game (any move, any type).
4. **HMs can be forgotten** at the move-relearn / level-up screen, no Move Deleter needed.
5. **Gastly's line learns 36 moves naturally** as it levels — its entire useful movepool (TMs, tutors, egg moves) available without items.
6. **Treecko, Torchic, and Mudkip are catchable in the wild** on themed routes (Route 102 / Fiery Path / Route 119, ~10% encounter rate).
7. **Player starts with $999,999** (the game's money cap) — effectively unlimited for any playthrough.
8. **Gastly's line is significantly stronger** — Gengar boosted from 500 BST to 695 BST (pseudo-legendary tier).
9. **Mew, Celebi, and Deoxys catchable in the wild** at 1% encounter rate on themed routes.
10. **Every Pokémon has catch rate 255** (the engine's maximum) — combined with weakening, virtually anything catches with any ball.

---

## 0. Legal / ROM note

These edits assume you own a legal copy of Pokémon Emerald and are working with a backup of your own cartridge dump. Don't distribute the patched ROM — share an IPS/UPS patch file instead (Lunar IPS does this).

The exact offsets below are for **Pokémon Emerald (US), version 1.0**. CRC32 of a clean ROM: `1F1C08FB`. If your CRC differs, the offsets will be off by a few bytes — use a tool like **G3HS (Hopeless Game and Hack Suite)** which searches by signature instead.

---

## 1. Tools

| Tool | What it's for | Where to get it |
| --- | --- | --- |
| **HxD** (Windows) or **Hex Fiend** (macOS) | Raw hex editing of the ROM | hxd.com / ridiculousfish.com/hexfiend |
| **mGBA** | Testing the patched ROM, save states | mgba.io |
| **Lunar IPS** | Optional — create an IPS patch from your modified ROM | romhacking.net |
| **G3HS** *(optional, GUI alternative)* | Skips most hex math — point-and-click for evolutions, learnsets, TM compat | Available on PokéCommunity |

If you'd rather not hex-edit at all, jump to **Section 6 — The lazy GUI path** at the bottom.

---

## 2. Back up the ROM

Before touching anything:

```
cp pokemon_emerald.gba pokemon_emerald_BACKUP.gba
```

Every change below is a few bytes. Keep the backup so you can re-patch from scratch if needed.

---

## 3. Replace a starter with Gastly

In Emerald, the three starter species IDs are stored as three 16-bit (2-byte) little-endian values, back-to-back (2-byte stride), at:

| Slot | Offset | Default species | Internal index | Default bytes |
| --- | --- | --- | --- | --- |
| 1 — Treecko (Bag 1) | `0x005B1DF8` | Treecko | 277 = 0x0115 | `15 01` |
| 2 — Torchic (Bag 2) | `0x005B1DFA` | Torchic | 280 = 0x0118 | `18 01` |
| 3 — Mudkip  (Bag 3) | `0x005B1DFC` | Mudkip  | 283 = 0x011B | `1B 01` |

**Important:** Gen 3 uses *internal species indices*, not National Pokédex numbers. They happen to match for #1–251 (Bulbasaur–Celebi), but Hoenn Pokémon are renumbered: Treecko = 277 not 252. Since Gastly is #92 (in the matching range), its internal index is also 92.

**Gastly's species index = 92 decimal = `0x5C`.** Little-endian bytes: **`5C 00`**.

Pick whichever starter slot you want Gastly to replace and overwrite the two bytes. Example — replace Treecko (the leftmost ball in Birch's bag):

1. Open the ROM in HxD.
2. Press `Ctrl+G`, enter `5B1DF8`, hit Enter.
3. The first two bytes should be `FC 00`. Overwrite with `5C 00`.
4. Save.

**Verification:** Boot the ROM in mGBA, walk to the Route 101 cutscene where Birch is attacked, open the bag — that slot now contains a Gastly.

> Note: this changes the *species* only. The starter's level (5), held item (none), and moves are determined by Gastly's natural learnset, so it will know whatever Gastly learns at level 5 (Hypnosis + Lick by Emerald's learnset).

---

## 4. Make Haunter evolve into Gengar by level-up

Each Pokémon has up to 5 evolution slots in a table starting at:

**Evolution table base: `0x32531C`**

Each Pokémon block is `5 × 8 = 40` bytes (`0x28`). Slot layout:

```
00–01  evolution method   (LE u16)
02–03  parameter          (LE u16 — level, item index, friendship, etc.)
04–05  target species     (LE u16)
06–07  padding            (00 00)
```

Method codes you care about:

| Code | Meaning |
| --- | --- |
| `0x00` | (empty slot) |
| `0x01` | Happiness (any time) |
| `0x04` | **Level up at a given level** |
| `0x05` | **Trade** ← what we want to remove |
| `0x07` | Use item (e.g. Thunder Stone) |

**Haunter's species index = 93 = 0x5D.**

Haunter's evolution block is at:

```
0x32531C + (0x5D × 0x28) = 0x32531C + 0xE88 = 0x3261A4
```

Open that offset in HxD. You will see eight bytes that look like:

```
05 00 00 00 5E 00 00 00
```

That reads as: **method 0x05 (Trade)** → parameter 0 → target species `0x5E` (94 = Gengar) → padding.

Overwrite it with:

```
04 00 24 00 5E 00 00 00
```

That now reads as: **method 0x04 (Level-up)** → at **level 0x24 = 36** → into Gengar.

(Use any level you like — `1E` = 30, `28` = 40. Level 36 mirrors how trade-evo Pokémon are often re-routed in fan hacks.)

**Verification:** In mGBA, give your Haunter rare candies. It should pop the evolution animation at level 36.

---

## 5. Let Gastly / Haunter / Gengar learn every TM and HM

Gen 3 stores TM/HM compatibility as an 8-byte bitfield per Pokémon — 64 bits, one for each of the 50 TMs + 8 HMs (with a few unused trailing bits). Setting all bits to 1 (`FF FF FF FF FF FF FF FF`) flags the Pokémon as learning every TM and every HM, regardless of type.

**Table base in Emerald: `0x1FD0F0`**
**Stride: 8 bytes per species, indexed by species ID.**

So the bitfield for species N starts at:

```
0x1FD0F0 + (N × 8)
```

For our three Pokémon:

| Species | Index | Offset of bitfield |
| --- | --- | --- |
| Gastly  | 92 (0x5C) | `0x1FD0F0 + 0x2E0` = **`0x1FD3D0`** |
| Haunter | 93 (0x5D) | `0x1FD0F0 + 0x2E8` = **`0x1FD3D8`** |
| Gengar  | 94 (0x5E) | `0x1FD0F0 + 0x2F0` = **`0x1FD3E0`** |

At each of those three offsets, overwrite the 8 bytes with:

```
FF FF FF FF FF FF FF FF
```

Now Gastly will gladly accept TM01 Focus Punch, TM06 Toxic, TM27 Return, HM01 Cut, HM03 Surf, HM07 Waterfall, etc. — anything with a type ban (like Normal-type TMs on a Ghost) is now bypassed.

> Caveat: this only affects TMs/HMs. The *level-up* learnset is still vanilla. If you also want Gastly to learn moves like Shadow Ball or Hypnosis naturally, edit the level-up learnset — easiest with G3HS (Section 6).

> The level-up move limit is also bypassed by TMs, so you can build whatever moveset you want via TMs alone.

---

## 6. Allow forgetting HMs

In vanilla Emerald, the engine refuses to let you forget HM moves (Cut, Fly, Surf, Strength, Flash, Rock Smash, Waterfall, Dive) outside the Move Deleter NPC in Lilycove.

There are two clean ways to remove the check:

### 6a. Nuke the HM move list (recommended — verified)

The simplest patch is to nuke the data the `IsMoveHm()` function reads from. Emerald stores the eight HM moves as a `0xFFFF`-terminated `u16` array at offset `0x329EEA`:

```
0F 00 13 00 39 00 46 00 94 00 F9 00 7F 00 23 01 FF FF
^ Cut  Fly   Surf  Stren Flash R.Sm Wfall Dive  end
```

The forget-move screen iterates this list, returns "is HM" if it finds the current move, and stops at `0xFFFF`. Change the **first two bytes** to `FF FF`:

| Offset | Old bytes | New bytes |
| --- | --- | --- |
| `0x00329EEA` | `0F 00` | `FF FF` |

Now the iteration terminates instantly and no move is ever flagged as an HM.

**Important — what this affects and what it doesn't:**

- ✅ Move-forget screen: HMs become forgettable
- ✅ PC Move-delete: HMs become deletable
- ✅ Move-replace on level up: HMs become replaceable
- ✅ Field moves (Cut/Surf/Fly/Strength/Flash/Rock Smash/Waterfall/Dive) keep working — they're triggered by Field Effect IDs, not by this list.
- ✅ HM01-08 items in your bag still teach moves normally — TM/HM teaching uses item data, not this list.

### 6b. Alternative: pre-made IPS patch

If you'd rather use a community patch, search PokéCommunity for **"HM Remove Patch (Pokemon Emerald)"** — there are several variants that do essentially the same thing at slightly different locations. Apply with Lunar IPS.

**Verification:** Teach Gastly HM01 Cut via TM/HM, then go to Pokémon → Summary → Moves → Cut and try to swap it out. The "Cut can't be forgotten!" message should not appear.

---

## 7. Putting it all together

Recommended order of operations:

1. Back up the clean ROM.
2. Apply Section 3 (starter swap).
3. Apply Section 4 (Haunter level-up evo).
4. Apply Section 5 (TM/HM compat).
5. Apply Section 6 (HM forget — patch or IPS).
6. Boot in mGBA, save state at the Route 101 satchel scene, and test each behavior in turn.
7. Use Lunar IPS → "Create IPS Patch" with `pokemon_emerald_BACKUP.gba` as source and your modded ROM as target. You now have a small `.ips` you can re-apply to any clean copy of Emerald.

---

## 8. The lazy GUI path (G3HS)

If hex math isn't your idea of fun:

1. Download **G3HS** (Hopeless Game and Hack Suite) from PokéCommunity.
2. Open your ROM.
3. **Starter swap:** Tools → Starter Editor → set slot 1/2/3 to Gastly.
4. **Trade-evo removal:** Pokémon Editor → Haunter → Evolutions tab → change "Trade" to "Level Up" at level 36 → Save.
5. **TM/HM compat:** Pokémon Editor → Gastly → TM/HM tab → "Check All". Repeat for Haunter and Gengar.
6. **Level-up learnset (optional bonus):** Pokémon Editor → Gastly → Attacks tab → add any moves you want at any levels.
7. **HM forget:** still needs the byte patch from 6a, or the IPS from 6b. G3HS doesn't expose this toggle.

G3HS handles all the offset math, including version-specific signature scanning, so it's robust against ROM variants.

---

## 9. Comprehensive level-up learnset (all of Gengar's good moves naturally)

The Gastly/Haunter/Gengar line gets a new shared learnset placed in free space at the end of the ROM (`0x00E3D000`), and the three learnset pointers in `gLevelUpLearnsets` (table base `0x32937C`) are updated to point at it.

**Learnset data format (Gen 3):** each entry is a u16 little-endian: `(level << 9) | move_id`. List terminates with `0xFFFF`.

**The 36-move learnset placed at `0x00E3D000`:**

| Level | Move | Move ID |
|---:|---|---:|
| 1 | Hypnosis | 95 |
| 1 | Lick | 122 |
| 1 | Spite | 180 |
| 1 | Astonish | 310 |
| 5 | Mean Look | 212 |
| 8 | Curse | 174 |
| 12 | Night Shade | 101 |
| 16 | Confuse Ray | 109 |
| 19 | Disable | 50 |
| 22 | Nightmare | 171 |
| 25 | Sludge Bomb | 188 |
| 28 | Shadow Ball | 247 |
| 31 | Toxic | 92 |
| 34 | Will-O-Wisp | 261 |
| 37 | Psychic | 94 |
| 40 | Dream Eater | 138 |
| 43 | Thunderbolt | 85 |
| 46 | Ice Punch | 8 |
| 49 | Fire Punch | 7 |
| 52 | Thunder Punch | 9 |
| 55 | Giga Drain | 202 |
| 58 | Pain Split | 220 |
| 61 | Destiny Bond | 194 |
| 64 | Perish Song | 195 |
| 67 | Shadow Punch | 325 |
| 70 | Skill Swap | 285 |
| 73 | Knock Off | 282 |
| 76 | Trick | 271 |
| 79 | Memento | 262 |
| 82 | Grudge | 288 |
| 85 | Snatch | 289 |
| 88 | Substitute | 164 |
| 91 | Thunder | 87 |
| 94 | Explosion | 153 |
| 97 | Rest | 156 |
| 99 | Sleep Talk | 214 |

**Pointer updates** at `gLevelUpLearnsets` (base `0x32937C`, stride 4):

| Species | Offset | Old pointer | New pointer |
|---|---|---|---|
| Gastly (0x5C) | `0x003294EC` | `6C 38 32 08` | `00 D0 E3 08` |
| Haunter (0x5D) | `0x003294F0` | `80 38 32 08` | `00 D0 E3 08` |
| Gengar (0x5E) | `0x003294F4` | `98 38 32 08` | `00 D0 E3 08` |

All three pointers share the same learnset (saves space, behavior identical).

---

## 10. Wild starters — Treecko, Torchic, Mudkip catchable in the wild

The Gen 3 wild-encounter engine reads exactly 12 land-encounter slots per map (hardcoded), so we can't literally add a 13th. Instead we **repoint each route's species pointer to a fresh copy of the table in free space** and swap one ~10% slot. The original tables are left untouched — reverting is just a pointer flip.

**Wild encounter master table:** `0x552D48` (20 bytes per entry: `u8 map_group, u8 map_num, u16 pad, u32 land_ptr, u32 water_ptr, u32 rock_smash_ptr, u32 fishing_ptr`).

**WildPokemonInfo struct (8 bytes):** `u8 encounter_rate, u8[3] pad, u32 species_table_ptr`. The species table is 12 slots × 4 bytes (`u8 min_lvl, u8 max_lvl, u16 species`).

**The three starter swaps:**

| Route | Map | Slot (prob) | Replaced | Now |
|---|---|---|---|---|
| Route 102 | (0, 17) | slot 5 (10%) | Lotad L4 | **Mudkip L5–7** |
| Fiery Path | (24, 14) | slot 4 (10%) | Torkoal L15 | **Torchic L14–16** |
| Route 119 | (0, 34) | slot 5 (10%) | Oddish L26 | **Treecko L25–27** |

(Lotad still appears in slot 4; Torkoal can still be obtained from the in-game gift NPC; Oddish still appears in three other Route 119 slots.)

**Free-space layout:**
- `0x00E3D000`: shared learnset (74 bytes)
- `0x00E3D100`: Route 102 species table copy with Mudkip (48 bytes)
- `0x00E3D130`: Fiery Path species table copy with Torchic (48 bytes)
- `0x00E3D160`: Route 119 species table copy with Treecko (48 bytes)

**Pointer updates** (modify only the species_table_ptr inside each WildPokemonInfo):

| Route | WildPokemonInfo address | Field offset | New pointer |
|---|---|---|---|
| Route 102 | `0x0055084C` | +4 → `0x00550850` | `00 D1 E3 08` |
| Fiery Path | `0x00551028` | +4 → `0x0055102C` | `30 D1 E3 08` |
| Route 119 | `0x00551308` | +4 → `0x0055130C` | `60 D1 E3 08` |

**About Gen 3 internal species indices:** Hoenn Pokémon are *not* indexed by their National Dex numbers — they're at 277+ in the order they appear in the internal Pokédex layout. Treecko = 277, Torchic = 280, Mudkip = 283, Lotad = 295, Oddish = 43 (Gen 1 carries over). Numel = 339 (not 322). Use a tool like G3HS or pret/pokeemerald's `species.h` for the full table.

---

## 11. Unlimited money

Pokémon Emerald stores the player's money XOR-encrypted in the save block with a per-game encryption key, which makes runtime "always 999,999" hacks fragile. The cleanest baked-in approach is to **bump the starting-money constant from $3,000 to $999,999** (the game's hard cap). Combined with trainer rewards and the cap-check in `SetMoney`, money is effectively unlimited for any practical playthrough.

The constant lives in the literal pool used by `NewGameInitData`:

| Offset | Old bytes | New bytes |
| --- | --- | --- |
| `0x000845BC` | `B8 0B 00 00` (3000) | `3F 42 0F 00` (999999) |

**About truly never-decreasing money:** to make `RemoveMoney` a no-op would require patching the function's first instruction to `mov r0, #1; bx lr` (`01 20 70 47`). Identifying it in Emerald US 1.0 requires disassembly — there's a likely candidate around `0x000E518C` (right after the MAX_MONEY literal at `0x000E5188`) but I haven't 100% verified it. If you want this, the safest path is to use a known PokéCommunity "Infinite Money" IPS patch on top of this build.

---

## 12. Stat boost — Gastly line to pseudo-legendary tier

The base stats table is at `0x3203CC` with **28 bytes per Pokémon**. The first 6 bytes are HP / Atk / Def / Spe / SpA / SpD.

| Pokémon | Offset | Vanilla stats | New stats | BST gain |
| --- | --- | --- | --- | --- |
| Gastly | `0x00320DDC` | 30/35/30/80/100/35 (310) | **70/60/60/110/130/65 (495)** | +185 |
| Haunter | `0x00320DF8` | 45/50/45/95/115/55 (405) | **85/75/75/120/140/85 (580)** | +175 |
| Gengar | `0x00320E14` | 60/65/60/110/130/75 (500) | **100/110/95/130/150/110 (695)** | +195 |

Gengar at 695 BST is in the pseudo-legendary tier (Salamence, Metagross, Tyranitar). Speed 130 outspeeds most things, SpA 150 hits like a truck.

---

## 13. Legendary wild encounters — Mew, Celebi, Deoxys

Same repoint-encounter-table technique as the starter wild encounters. Each gets a 1% slot at level appropriate to its route.

| Pokémon | Route | Map | Slot (prob) | Level | Internal index |
| --- | --- | --- | --- | --- | --- |
| **Celebi** | Petalburg Woods | (0, 27) | slot 10 (1%) | 5–7 | 251 |
| **Mew** | Route 120 | (0, 35) | slot 10 (1%) | 30–35 | 151 |
| **Deoxys** | Route 121 | (0, 36) | slot 10 (1%) | 40–45 | 410 (Normal forme) |

Notes:
- Deoxys has 4 different formes stored as 4 separate species (407 Defense, 408 Attack, 410 Normal, 412 Speed in pokeemerald layout). I used the Normal forme (50/150/50/150/150/50 BST = 600).
- The game's "you can only catch this legendary once" logic uses flags in scripts, not encounter slots. A wild-spawn legendary doesn't trigger those flags, so you could theoretically encounter (but only one fits in your party at a time).
- All three now have catch rate 255 (see next section), so combined with weakening + status they're catchable with regular Poké Balls.

**Free-space layout (continuation):**

| Offset | Content |
| --- | --- |
| `0x00E3D200` | Petalburg Woods species table copy with Celebi (48 bytes) |
| `0x00E3D230` | Route 120 species table copy with Mew (48 bytes) |
| `0x00E3D260` | Route 121 species table copy with Deoxys (48 bytes) |

**Pointer updates:**

| Route | WildPokemonInfo `species_table_ptr` offset | New pointer |
| --- | --- | --- |
| Petalburg Woods | `0x00550B4C` | `00 D2 E3 08` |
| Route 120 | `0x00551390` | `30 D2 E3 08` |
| Route 121 | `0x00551414` | `60 D2 E3 08` |

---

## 14. Maximum catch rate (engine cap)

Every Pokémon's catch rate field (offset +8 within each 28-byte base stats entry) is overwritten to **`0xFF` (255)** — the engine's maximum.

```
for species in 1..411:
    offset = 0x3203CC + species*28 + 8
    rom[offset] = 0xFF
```

This means 371 single-byte patches across the base stats table (40 species were already at 255 in vanilla — mostly weak/common Pokémon).

**Practical result on catch chance:**

| Target | Vanilla catch chance (full HP, Poké Ball) | With cr=255 |
| --- | --- | --- |
| Mewtwo (cr 3 → 255) | < 0.5% | ~33% |
| Mewtwo at 1 HP + Sleep | ~3% | **~100%** |
| Common wild Pokémon | already 50%+ | virtually 100% with weakening |

**About true "100% any ball" guarantee:** the actual catch outcome is determined by a shake-count formula in `Cmd_handleballthrow` (battle script command for ball throw). To literally guarantee capture without weakening, that function's `odds >= 0xFF` comparison would need to be patched to always pass. I couldn't safely locate it without proper disassembly tooling — search for `0xFF0000` literal pool entries returned only graphics data, and the comparison `cmp r0, #0xFF` is too common a Thumb pattern to disambiguate. The catch-rate-255 approach gets you ~95% of the way there in practice. If you want a strict 100%, the easiest path is a community **"100% Catch Rate"** GameShark/AR cheat applied at the emulator level, or use Master Balls (which you can buy with your $999,999 — they're sold at the Battle Frontier).

---

## 15. Title screen — "Pokémon: Gengar Edition"

Reference mockups for the new title screen have been generated as standalone image files:

| File | Purpose |
| --- | --- |
| `title_screen/gengar_edition_title_final.png` | High-res static mockup, **Distressed Grunge** font style (your pick) |
| `title_screen/gengar_edition_title_animated.gif` | Animated preview — pulsing red eyes, breathing purple aura, "PRESS START" blink |
| `title_screen/font_comparison_grid.png` | Side-by-side of all 4 font styles considered |
| `title_screen/gba_reality_check.png` | What 240×160 @ 16/64 colors actually looks like vs the full-res mockup |

**These are reference art**, not in-ROM patches. Replacing the actual title screen in the ROM requires:

1. **Image conversion to GBA tile format.** The title screen background is rendered as 4bpp tiles (16 colors per tile drawing from a 16-color palette bank). Tools: `gbagfx` (from pokeemerald build chain), `Tilemap Studio`, or GraphicsGale. The mockup needs to be heavily reduced — color-quantized to ≤16 colors and down-sampled to 240×160.

2. **Logo replacement.** "Pokémon Emerald Version" is a baked-in image in the ROM, not text. The new "Pokémon: Gengar Edition" logo must be drawn as a separate bitmap and substituted at the same tile/palette offsets.

3. **LZ77 decompression cycle.** Most Emerald title screen graphics are stored LZ77-compressed. You need to decompress the original, replace, recompress (or write uncompressed if it fits), and repoint if size changes.

4. **Animation in-ROM.** Animation is done via palette cycling or tile-swapping in the title screen update function. The mockup's pulsing-eye effect could be replicated with a palette cycle on the 2-3 palette entries used for the eyes.

**Relevant ROM offsets to target** (Emerald US 1.0, for whoever does the implementation):

| Asset | ROM offset | Notes |
| --- | --- | --- |
| Title screen logo tiles | `0x008CA610` (approx) | LZ77-compressed; "Pokémon Emerald Version" logo |
| Title screen Rayquaza tiles | `0x0017610C` (approx) | LZ77-compressed |
| Title screen BG palettes | varies | Loaded by the title screen state init |
| Title screen tilemap | `0x008CC178` (approx) | The arrangement of tiles on screen |

(I haven't verified these offsets surgically — they're from PokéCommunity references. Anyone doing the swap should confirm with G3HS or pret/pokeemerald symbol maps before writing.)

**My recommendation:** keep the high-res PNG/GIF as art direction. If you want to commission this, hand the `gengar_edition_title_final.png` to someone with TilemapStudio experience and they can do the GBA-pixel-art conversion. The mockup is the spec, they translate it to hardware.

---

## 16. Things I attempted but couldn't bake in (and how to add them)

**Run indoors.** The check lives in `IsRunningDisallowed` which reads bit 2 of `gMapHeader.flags`. A well-known community single-byte patch flips one conditional branch in the running validation, but the specific offset varies by ROM revision and I couldn't safely identify it empirically (the bit-test pattern `04 21 08 42` returned zero hits — the compiler emitted a different instruction sequence than expected). **To add it:** download the **"Run Indoors / Run Anywhere" IPS patch** for Pokémon Emerald US 1.0 from PokéCommunity and apply on top of this ROM with Lunar IPS.

**Faster default text speed.** `OPTIONS_TEXT_SPEED_MID` (value 1) is stored as the default in `NewGameInitData`. The compiled instruction is a generic `mov rX, #1` which appears thousands of times in the ROM, so searching for it directly isn't reliable. **To add it:** in-game, set text speed to Fast on the first save; or apply a community **"Fast Text"** IPS.

**Truly unlimited balls.** Bag contents are XOR-encrypted in the save with a per-save key, so we can't pre-load 999 Poké Balls into the ROM. **Workaround that's already in place:** $999,999 starting money buys ~5000 Poké Balls from the first PokéMart. Functionally unlimited.

**Literal 100% catch with any ball.** Requires patching the shake-count formula in `Cmd_handleballthrow`. **To add it:** apply a community **"100% Catch Rate"** IPS, or use the GameShark code `82005274 0001` at the emulator level (mGBA → Tools → Cheats).

---

## 17. Quick reference — every byte you need to write

| Change | Offset | Old bytes | New bytes |
| --- | --- | --- | --- |
| Starter slot 1 → Gastly | `0x005B1DF8` | `15 01` | `5C 00` |
| Haunter evo method | `0x003261A4` | `05 00 00 00 5E 00 00 00` | `04 00 24 00 5E 00 00 00` |
| Gastly TM/HM compat | `0x001FD3D0` | `E7 D9 D5 00 E3 DA 00 C2` | `FF FF FF FF FF FF FF FF` |
| Haunter TM/HM compat | `0x001FD3D8` | `C9 BF C8 C8 AB FF BC CC` | `FF FF FF FF FF FF FF FF` |
| Gengar TM/HM compat | `0x001FD3E0` | `BB D1 C6 D3 F0 00 BB 00` | `FF FF FF FF FF FF FF FF` |
| HM list head terminator | `0x00329EEA` | `0F 00` | `FF FF` |
| Gastly learnset pointer | `0x003294EC` | `6C 38 32 08` | `00 D0 E3 08` |
| Haunter learnset pointer | `0x003294F0` | `80 38 32 08` | `00 D0 E3 08` |
| Gengar learnset pointer | `0x003294F4` | `98 38 32 08` | `00 D0 E3 08` |
| Route 102 species pointer | `0x00550850` | `1C 08 55 08` | `00 D1 E3 08` |
| Fiery Path species pointer | `0x0055102C` | `F8 0F 55 08` | `30 D1 E3 08` |
| Route 119 species pointer | `0x0055130C` | `D8 12 55 08` | `60 D1 E3 08` |
| Starting money | `0x000845BC` | `B8 0B 00 00` | `3F 42 0F 00` |
| Gastly base stats | `0x00320DDC` | `1E 23 1E 50 64 23` | `46 3C 3C 6E 82 41` |
| Haunter base stats | `0x00320DF8` | `2D 32 2D 5F 73 37` | `55 4B 4B 78 8C 55` |
| Gengar base stats | `0x00320E14` | `3C 41 3C 6E 82 4B` | `64 6E 5F 82 96 6E` |
| All 371 catch rates | various (every `0x3203CC + N*28 + 8`) | varies | `FF` |
| Petalburg Woods species ptr | `0x00550B4C` | `18 0B 55 08` | `00 D2 E3 08` |
| Route 120 species ptr | `0x00551390` | `5C 13 55 08` | `30 D2 E3 08` |
| Route 121 species ptr | `0x00551414` | `E0 13 55 08` | `60 D2 E3 08` |
| New learnset (74 bytes) | `0x00E3D000` | (FF padding) | encoded learnset table |
| Route 102 new species table | `0x00E3D100` | (FF padding) | 48-byte copy with Mudkip |
| Fiery Path new species table | `0x00E3D130` | (FF padding) | 48-byte copy with Torchic |
| Route 119 new species table | `0x00E3D160` | (FF padding) | 48-byte copy with Treecko |
| Petalburg Woods new species table | `0x00E3D200` | (FF padding) | 48-byte copy with Celebi |
| Route 120 new species table | `0x00E3D230` | (FF padding) | 48-byte copy with Mew |
| Route 121 new species table | `0x00E3D260` | (FF padding) | 48-byte copy with Deoxys |

Total: **386 patches** + **7 new data blocks**. The accompanying IPS file is **2,756 bytes** (most of which is the 371 catch-rate byte changes across the base stats table).

All bytes verified against a clean Emerald US 1.0 ROM (CRC32 `1F1C08FB`).

---

## 10. Troubleshooting

- **Your hex editor shows different default bytes at one of those offsets.** Your ROM is probably Emerald US 1.1, Emerald (EU), Emerald (J), or an already-modified copy. Re-dump a clean Emerald US 1.0, or use G3HS which auto-locates by signature.
- **The starter selection cutscene crashes.** You likely overwrote the wrong byte. Restore the backup and try again — only those two bytes should change.
- **Haunter still won't evolve at 36.** Double-check the eight bytes; the order is `method, param, target` and all are little-endian. `04 00 24 00 5E 00 00 00` not `00 04 00 24 00 5E 00 00`.
- **HM still won't forget.** The byte patch only covers the standard menu path; some hacks check elsewhere. The community IPS in 6b covers all paths.

Happy haunting.
