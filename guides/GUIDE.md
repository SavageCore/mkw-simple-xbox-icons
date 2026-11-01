# Mario Kart World: Icon Swap Guide

This guide covers how to visually swap the Nintendo Switch button icons (A/B and X/Y) to match the physical layout of an Xbox controller.

## 1. Extracting the Game Data

1. Open Eglantine, right-click **Mario Kart World** in the game list.
2. Extract RomFS and navigate to `romfs/Font/` in the dumped folder.
3. Copy out `PicFont_00.Nin_Ounce_NVN2.bffnt.zs`. This single font file holds the face-button glyphs - the 218 Layout archives were all checked and none contain ABXY art (`Layout/BtnIconFooter_00` is only the shared footer frame).

## 2. Opening the Font in Switch Toolbox

> **Linux users:** see [Running Switch Toolbox on Linux](SWITCH-TOOLBOX-LINUX.md).

1. Launch **Switch Toolbox**.
2. `File` > `Open` the `.bffnt.zs` file (Toolbox reads zstd directly, no need to decompress first).
3. Expand the font entries to the glyph sheet texture (1024x1024) and preview it. The bottom button row reads, left to right: **A B X Y L R ZL ZR SL**.

## 3. Swapping the Icons

Swap the **image data** and keep everything else (names, codepoints) intact, so game logic is unaffected:

- **A** ↔ **B**
- **X** ↔ **Y**

1. **Export** the sheet and keep the export as a backup.
2. In any image editor, exchange the A glyph with B, and X with Y.
3. **Replace** the sheet in Toolbox with the edited image.
4. **Verify**: click each entry and check the preview. The A slot should now visually show a **B**, and so on.

## 4. Saving

Save from Toolbox. If the output does not match the original `.zs` framing, save uncompressed and recompress it yourself:

```sh
zstd PicFont_swapped.bffnt -o PicFont_00.Nin_Ounce_NVN2.bffnt.zs
zstd -d -c PicFont_00.Nin_Ounce_NVN2.bffnt.zs | cmp - PicFont_swapped.bffnt
```

(The second command must print nothing: that confirms a clean round-trip.)

## 5. Mod Deployment (Eglantine)

Title ID: `0400C3F00006E000`

Place the modified file under:

```
Simple Xbox Icons/romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs
```

Full emulator path:

```
load/0400C3F00006E000/Simple Xbox Icons/romfs/Font/PicFont_00.Nin_Ounce_NVN2.bffnt.zs
```
