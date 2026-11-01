# Mario Kart World: Icon Swap Guide

This guide covers how to visually swap the Nintendo Switch 2 button icons (A/B and X/Y) to match the physical layout of an Xbox controller.

Game dump used here: `/home/savagecore/Roms/Switch 2/MKW/base/romfs/`

## 0. Target file (confirmed)

MKW face-button prompts are **font glyphs**, not layout textures. All 218 `.blarc` archives were scanned and contain no ABXY textures.

- File: `romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs` (zstd-compressed BFFNT, `Ounce` = Switch 2 codename, `NVN2` = API)
- Decompressed size: 4,202,856 bytes
- Glyph sheet: single 1024x1024 `R8G8B8A8_UNORM_SRGB` texture
  - BNTX container at file offset `0x1000` (length `0x401068`)
  - Raw blocklinear pixels at file offset `0x2000` (length 4,194,304)
  - TileMode 0 (Default), BlockHeightLog2 4, platform `Ounc` (Orin/Ounce deswizzle path)
- Grid: 9x9 cells, 105 px pitch, 104x104 glyph area + white baseline bar
- Top icon row (display orientation), left to right: **A B X Y L R ZL ZR SL**

### CMAP (private use E0A0-E0CF -> glyph index)

Relevant entries (full table in `Font/` dump notes):

| Codepoint | Glyph | Button |
|-----------|-------|--------|
| U+E0CB | 35 | Y |
| U+E0CC | 36 | X |
| U+E0CD | 37 | B |
| U+E0CE | 38 | A |
| U+E0C9 | 33 | R |
| U+E0CA | 34 | L |
| U+E0C5 | 29 | ZL |
| U+E0C6 | 30 | ZR |
| U+E0C4 | 28 | SL |
| U+E0BB | 21 | SR |

(U+E0A0-E0A2 all map to glyph 0, U+E0B7-E0BA / E0BD-E0BE are unmapped `FFFF`.)

### Swap logic

Keep the original Nintendo codepoints so game logic stays intact - only exchange the glyph pixels:

- **A** (glyph 38) <-> **B** (glyph 37)
- **X** (glyph 36) <-> **Y** (glyph 35)

Display order changes from `A B X Y ...` to `B A Y X ...` (see `previews/verify_check.png`).

The abandoned `Layout/BtnIconFooter_00` lead is documented here only to save future work: it is a shared footer layout, not the button art. Its `timg/__Combined.bntx` exposes only `DeleteCan_00^t` and references `PicFont_00.fcpx`, which is what pointed at the font file.

## 1. Extracting the Game Data

The dump is already available at `/home/savagecore/Roms/Switch 2/MKW/base/`. For a fresh dump from Eglantine:

1. Open Eglantine, right-click **Mario Kart World** in the game list.
2. Extract RomFS (Eden-family equivalent of Yuzu's Dump RomFS / Ryujinx's Extract Data).
3. Navigate to `romfs/Font/` in the dumped folder.

Decompress (Toolbox cannot read `.zs` directly):

```sh
mkdir -p /tmp/opencode/mkw/font
zstd -d -c "/home/savagecore/Roms/Switch 2/MKW/base/romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs" > /tmp/opencode/mkw/font/PicFont.bffnt
```

## 2. Decoding the Glyph Sheet (headless, no Toolbox GUI)

Wine exposes no AT-SPI tree, so Toolbox's GUI texture preview cannot be driven. `SaveBitMap`/`GetBitmap` also fail headless (they need an OpenGL context and return null). The working path is `TextureData.GetImageData`, which deswizzles on the CPU via `TegraX1Swizzle.deswizzle(..., isOrin: true)`.

Minimal headless exporter (`ExportFont.cs`, compiled with the bottle `csc.exe` against `FirstPlugin.Plg.dll` + `Toolbox.Library.dll` + `Syroot.NintenTools.NSW.Bntx.dll`):

- Slice the BNTX out of the BFFNT (`0x1000`, length `0x402068 - 0x1000`), `BNTX.Load` it.
- Call `texture.GetImageData(0, 0, 0)` -> 4,194,304 linear RGBA bytes.
- Note: the returned rows are bottom-up (OpenGL convention). Flip vertically for viewing.

Cell layout (display orientation, 105 px pitch starting at x=0):

- Columns: A `[1,86]`, B `[106,191]`, X `[211,296]`, Y `[316,401]` (y `[0,88]`, underline bar at y 106-112 is identical in every cell and stays in place).

Swap the four 86x89 glyph blocks A<->B and X<->Y with Pillow/numpy, flip back to raw orientation, and save as `swapped_linear.raw`.

## 3. Re-swizzling and Patching

Re-swizzle with Toolbox's own inverse (`SwizzleFont.cs`):

```csharp
TegraX1Swizzle.swizzle(1024, 1024, 1, 1, 1, 1, 0, 4, tileMode: 0, blockHeightLog2: 4,
    data: swappedLinear, size: 4194304, isOrin: true)
```

(`tileMode`/`blockHeightLog2` are read back from the loaded `Texture` object; both files use 0 / 4.)

Patch the decompressed BFFNT at the verified offset (`0x2000`, confirmed byte-identical to the original `TextureData[0][0]`):

```sh
python3 do_patch.py  # data[:0x2000] + swapped + data[0x2000+4194304:]
```

Verify by re-running the step-2 exporter on the patched file: the top row must read `B A Y X L R ZL ZR SL` (`previews/verify_check.png`).

Recompress with plain zstd (original framing is stock zstd, magic `28 b5 2f fd`):

```sh
zstd -f PicFont_swapped.bffnt -o PicFont_00.Nin_Ounce_NVN2.bffnt.zs
zstd -d -c PicFont_00.Nin_Ounce_NVN2.bffnt.zs | cmp - PicFont_swapped.bffnt && echo ROUNDTRIP_OK
```

## 4. Mod Deployment (Eglantine)

Title ID: `0400C3F00006E000`

Place the modified file under:

```
Simple Xbox Icons/romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs
```

Full emulator path:

```
load/0400C3F00006E000/Simple Xbox Icons/romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs
```

Take an in-game screenshot of a button prompt and save it as `previews/preview.png`.
