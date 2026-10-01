# sd2snes bootleg core (fpga_bootleg)

Dedicated core for the copy-protected unlicensed LoROM bootlegs listed in
fullsnes "SNES Cart Unlicensed Variants" (sttng/gb-stuff snes_bootleg.md).
It is `sd2snes_base` plus `bootleg.v`; everything else (MSU-1, DMA, cheats,
in-game hooks, SFX fetcher) is unchanged.

| variant (chipfeat[2:0]) | games | hardware |
|---|---|---|
| 1 BITSWAP  | Aladdin 2000, Digimon Adventure, KOF2000, Pocket Monster, Pokemon Gold Silver, Pokemon Stadium, Soul Edge Vs Samurai, X-Men vs SF, Squirrel | latch in banks A23=1/A18-16=000 (80,88,...), read = bits 0,6,7,1,2,3,4,5 |
| 2 CONSTANT | Soul Blade, Hercules, Dragon Ball Z Final Bout | 80-BF:8000-FFFF = 55,0F,AA,F0; C0-FF open bus |
| 3 ALU      | Tekken 2, SF EX Plus Alpha | 80-BF:8000-87FF set/clear/count/shift unit, result at 81xx |
| 4 PORT6    | A Bug's Life, Bananas de Pijamas (both verified in emulation) | 00-3F/80-BF:6000-6FFF; function undocumented, core returns one answer set that passes both games' checks (61=2 while 60xx armed, 63=4, 65=F, 67=0, 6F=3) |
| 5 BITSWAP40 | Marvel Super Heroes vs Street Fighter (verified in emulation) | same latch and bit order as BITSWAP, decoded at 40-4F:8000-FFFF (game writes 4x:xxx2, reads 4x:xxx0) |
| 6 KOF98     | King of Fighters '98 (verified in emulation) | BITSWAP as type 1, plus a bank register at C0-CF:8000-FFFF (game: C0:8788 = 82/00); while bit 7 is set, ROM accesses see A19..A16 replaced by its low nibble. Remap applied in main.v before address.v |

## Layout
- `verilog/sd2snes_bootleg/` – the core (drop next to `sd2snes_base`, `CORE = bootleg`
  -> `fpga_bootleg.bit` / `fpga_bootleg.bi3`, copy to `/sd2snes/` on the SD card).
- `src/` – changed firmware files (full copies). `patches/firmware.diff` is the same
  change as a patch against the uploaded tree (`patch -p1`), verified to apply cleanly.
- `patches/core_vs_base.diff` – what differs from `sd2snes_base` (review aid).
- `sim/tb_bootleg.v` – self-checking testbench vs. a MAME-derived model
  (`iverilog -g2012 sim/tb_bootleg.v verilog/sd2snes_bootleg/bootleg.v && vvp a.out`).
- `src/utils/bootleg_fp.py` – prints table rows with the 64 KB fingerprint filled in.

## How a game gets here
`load_identify()` enables `bootleg_scan` for normal SNES game loads; `smc_id()` then
CRC32s the headerless image only if its size is 1, 2 or 3 MB. A match sets
`fpga_conf = FPGA_BOOTLEG`, `mapper_id = 1`, `fpga_dspfeat = variant`, clears any
chip the copied header claimed, and sizes the ROM from the file. `fpga_dspfeat`
already goes to the FPGA as CMD 0xEF; the new core's `mcu_cmd.v` decodes it.

## Known limits
- All 64 KB fingerprints are filled in: other 1/2/3 MB games cost one 64 KB read.
- Not run on hardware yet. mk2 firmware size not checked (tight 128 KB flash).
- No savestates on this core (not in savestate.c's core list).

## Corrections vs. the fullsnes list
Per nocash (nesdev forum t=15510, 2017): A Bug's Life and Bananas de Pijamas use a
"port 6xxx" protection (their bitswap code is dead), and SF EX Plus Alpha uses the
Tekken 2 ALU. A Bug's Life was traced in an emulator: with bitswap it hits BRK at
01:85C5 on frame 19; with PORT6 it boots and plays (8000 frames, no further port reads
apart from stray pointer reads at A1:61C0, which the core leaves alone).

## Overdumps
Some circulating Picachu, Pokemon Stadium, Tekken 2 (8 MB) and DBZ Final Bout (4 MB) files are overdumps: every 32 KB
bank stored twice, mirrored to 8 MB. They are not recognised; convert them with
  python3 src/utils/bootleg_fp.py --fix OUTDIR *.sfc
which writes the clean 2 MB images (their CRCs then match the table).

## Emulator check (sim/lakesnes)
Boot + ~6000 frames with the core models: Aladdin 2000, KOF2000, Picachu, Pokemon G&S,
Pokemon Stadium, X-Men vs SF, Soul Edge vs Samurai, Squirrel, Marvel vs SF, KOF98, Hercules, DBZ Final Bout, Tekken 2, SF EX Plus Alpha (ALU; black screen with
bitswap), Bug's Life, Bananas all reach gameplay/character select.
Unclear in the emulator: Digimon (bitswap boot check passes, title screen OK, black
after Start with or without protection) and Soul Blade (fights start, graphics
corrupted; also with C0-FF mapped as ROM). Need hardware tests.
Hercules uses the same constant pattern and matches its crack frame-for-frame for 9000
frames, so Soul Blade's corruption is unlikely to come from the pattern itself.

## Dragon Ball Z - Final Bout: incomplete dump (audio only) and full restoration
The known dump (CRC 5BBA4EB3) has four blank 32 KB banks, ROM $038000-$057FFF.  They are
sound data only: the upload table at $05:8000 (76 entries) points into them from entry #32
on.  A blank entry is a valid empty upload, so the game runs but goes silent (e.g. from
character select on).  Nothing else references the hole (the protection reads at banks
$87-$8A overlap its offsets but return the pattern on the real cart).

Source of the data: DVS copied Street Fighter II (Japan) (CRC 5556C5C9) ROM banks $0A-$0F
verbatim into DBZ banks $05-$0A and rebased the table by -5 banks.  Verified: DBZ's intact
65,308 bytes equal SF II at +$28000, the table matches with every bank byte -5, all 76
entries are valid chains in SF II ending exactly on DBZ's table boundaries, and the copy
ends at the bank boundary.  So the hole is SF II $060000-$07FFFF:

  python3 src/utils/repair_dbz_sound.py DBZ_DUMP SF2_DUMP DBZ_restored.sfc   -> CRC DD7AFCB9

(Accepts the 2 MB dump or the 4 MB doubled overdump; refuses any other input.)
Emulation: identical video, no hang, music where the original is silent.  The image is still
the protected cart (the bootleg core provides the protection); both CRCs are in the table.
An earlier partial repair built from SF EX (CRC 51EEB811) is superseded: its entry #60 was
wrong and #61-#66 were missing.  It is still recognised so an existing copy keeps loading.
