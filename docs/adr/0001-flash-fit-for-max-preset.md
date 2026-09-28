# Flash fit for the Max preset under 118 KiB

The monolithic Max preset (every resident feature enabled at once) overflowed the PY32F071's 118 KiB internal application Flash (120 832 B). We decided to make it fit by surgical, behavior-preserving savings only — no feature removal, no UI degradation, no EEPROM/PY25Q16/calibration format change — landing Max at 118 172 B with 2 660 B free.

## Considered Options

- **Link-time optimization (`-flto`)**: evaluated on `research/lto-feasibility`, worth ~3.5–5 KiB, but **not applied**. The current `ENABLE_LTO` wiring is inert (flags on the empty executable target only) and enabling it globally risks inlining across the `.mb_ramfunc` RAM-stub boundary guarded by `cmake/check_mb_ramfunc.cmake`. Kept as a future lever with the isolation recipe recorded on that branch (`-fno-lto` on `mb_flash.c` + `noclone`).
- **RLE / bit-packing decompression of fonts and bitmaps**: benchmarked on `research/font-bitmap-compression` and **rejected**. Per-glyph indexing tables cost more Flash than the compression saves (net −641 B), and streaming decode would need ~2 KiB of RAM we don't have.
- **Procedural bold derivation (`gFontSmallBold = normal | normal << 1`)**: **rejected**. Matches only 8.5 % of glyphs and fills the counters of `0 e a s B 8` at 6×8 px — a visible UI regression. Instead, `ENABLE_SMALL_BOLD` is off for Max only (564 B saved, clean fallback to `gFontSmall` in `UI_PrintStringSmallBold`).
- **Merging the BEAM/AirCopy FSK codecs and audio/UI paths**: **rejected** except for the transmit tail. The scout audit showed decoders, tone sequences and progress bars have incompatible wire formats, timing contracts and geometry; only the 4-call PA-off transmit suffix was shared (`AIRCOPY_TransmitBufferNow`, −12 B).
- **Signed-division eradication**: **applied**. Cortex-M0+ has no hardware divider, so every signed `/`/`%` pulls `__divsi3` (~468 B). A dozen provably non-negative sites (menu timers, S-meters, scan/spectrum math, TX-offset index, `map()`, `randInt`) were converted to unsigned; vendor `POSITION_VAL(__CLZ(__RBIT()))` became `__builtin_ctz` (constant-foldable); `-lm` was dropped (libm was already 0 B). Measured: Fusion −240 B, Max −248 B.
- **Offloading Breakout to an overlay app**: closed as out of scope — unnecessary once Max fit natively.

## Consequences

- New code must keep divisions unsigned where operands are provably non-negative, keep `PRINTF_DISABLE_SUPPORT_FLOAT`, and never link float/`libm` (a single `%f` or `double` costs 2–7 KiB on this core).
- Max intentionally renders small-bold text in regular weight; all other presets keep `ENABLE_SMALL_BOLD`.
- Residual `_divsi3` (~468 B across ~14 sites with genuinely signed operands) and LTO remain the two documented future levers if headroom shrinks again.

## Validation

`./compile-firmware.sh All` (GCC 13.3.rel1, `-Os`, `nano.specs`): Fusion 106 712 B, Transfer 111 088 B, FieldOps 112 776 B, Labs 111 684 B, **Max 118 172 B / 120 832 B (2 660 B free)**. RAM unchanged (Max 12 448/16 384 B). `.mb_ramfunc` isolation check passes on every preset.
