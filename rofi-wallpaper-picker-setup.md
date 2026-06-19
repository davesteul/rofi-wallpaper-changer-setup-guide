# Rofi Wallpaper Picker — Setup Guide

A visual wallpaper switcher using rofi's dmenu mode with real square thumbnails, bound to a Hyprland keybind. No GTK app, no caching delay, opens instantly.

## How it works

1. Wallpapers are pre-cropped into square thumbnails (so every cell in the grid fills evenly, regardless of the original image's aspect ratio).
2. A script feeds rofi a list of `thumbnail icon -> original file path` pairs using rofi's native `\0icon\x1f` dmenu syntax.
3. Selecting a thumbnail applies the **original full-resolution file** as the wallpaper via `awww` (Arch's official swww replacement).

---

## 1. Prerequisites

```bash
sudo pacman -S rofi awww imagemagick
```

- `rofi` — the picker UI
- `awww` — wallpaper backend (Arch's official repo replacement for `swww`)
- `imagemagick` — used once to generate square thumbnails

Make sure `awww-daemon` is running. Add to your Hyprland autostart (`hyprland.lua`):

```lua
hl.exec_cmd("awww-daemon")
```

---

## 2. Folder structure

```bash
mkdir -p ~/wallpapers
mkdir -p ~/.config/hypr/scripts
```

Drop your `.jpg` / `.jpeg` / `.png` wallpapers directly into `~/wallpapers` (not subfolders — the script only scans the top level).

**Important:** keep phone/portrait wallpapers out of this folder, or run the sorter script below first. Mixed aspect ratios make square thumbnails look inconsistent.

### Optional: separate out portrait/phone wallpapers

```bash
nano ~/.config/hypr/scripts/sort-phone-wallpapers.sh
```

```bash
#!/bin/bash

WALLPAPER_DIR="$HOME/wallpapers"
PHONE_DIR="$HOME/wallpapers-phone"

mkdir -p "$PHONE_DIR"

identify -format "%w %h %f\n" "$WALLPAPER_DIR"/*.{jpg,jpeg,png,JPG,JPEG,PNG} 2>/dev/null | \
while read -r width height filename; do
    if [ -z "$width" ] || [ -z "$height" ]; then
        continue
    fi

    # If height >= width, it's portrait/square -> likely a phone wallpaper
    if [ "$height" -ge "$width" ]; then
        echo "Moving (portrait): $filename"
        mv "$WALLPAPER_DIR/$filename" "$PHONE_DIR/"
    fi
done

echo "Done. Phone wallpapers moved to $PHONE_DIR"
```

```bash
chmod +x ~/.config/hypr/scripts/sort-phone-wallpapers.sh
~/.config/hypr/scripts/sort-phone-wallpapers.sh
```

Review `~/wallpapers-phone` and delete it once confirmed (`rm -rf ~/wallpapers-phone`).

---

## 3. Generate square thumbnails

```bash
nano ~/.config/hypr/scripts/make-square-thumbs.sh
```

```bash
#!/bin/bash

WALLPAPER_DIR="$HOME/wallpapers"
THUMB_DIR="$HOME/wallpapers/.thumbs"

mkdir -p "$THUMB_DIR"

find "$WALLPAPER_DIR" -maxdepth 1 -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \) -print0 | \
while IFS= read -r -d '' file; do
    filename=$(basename "$file")
    thumb="$THUMB_DIR/$filename"

    if [ -f "$thumb" ]; then
        continue
    fi

    magick "$file" -resize 400x400^ -gravity center -extent 400x400 "$thumb"
    echo "Created thumbnail: $filename"
done

echo "Done."
```

```bash
chmod +x ~/.config/hypr/scripts/make-square-thumbs.sh
~/.config/hypr/scripts/make-square-thumbs.sh
```

Run this again any time you add new wallpapers — it skips files that already have a thumbnail, so it's safe to re-run.

---

## 4. The picker script

```bash
nano ~/.config/hypr/scripts/wallpaper-menu.sh
```

```bash
#!/bin/bash

WALLPAPER_DIR="$HOME/wallpapers"
THUMB_DIR="$HOME/wallpapers/.thumbs"
THEME="$HOME/.config/hypr/scripts/wallpaper-menu.rasi"

selected_path=$(find "$WALLPAPER_DIR" -maxdepth 1 -type f \( -iname "*.jpg" -o -iname "*.jpeg" -o -iname "*.png" \) -print0 | \
    shuf -z | \
    while IFS= read -r -d '' file; do
        filename=$(basename "$file")
        thumb="$THUMB_DIR/$filename"
        printf '%s\0icon\x1f%s\n' "$file" "$thumb"
    done | \
    rofi -dmenu -p "Wallpaper" -theme "$THEME")

[ -z "$selected_path" ] && exit 0

awww img "$selected_path"
```

```bash
chmod +x ~/.config/hypr/scripts/wallpaper-menu.sh
```

`shuf -z` randomizes the order on every launch. Remove that one line if you'd rather see wallpapers in consistent (alphabetical) order.

---

## 5. The rofi theme

```bash
nano ~/.config/hypr/scripts/wallpaper-menu.rasi
```

```css
configuration {
    show-icons: true;
}

* {
    background-color: #64727D;
    text-color: #ffffff;
}

window {
    width: 900px;
    height: 900px;
    background-color: #64727D;
}

mainbox {
    background-color: #64727D;
    children: [listview];
}

listview {
    columns: 4;
    lines: 4;
    flow: horizontal;
    fixed-columns: true;
    fixed-height: true;
    spacing: 10px;
    padding: 10px;
    background-color: #64727D;
}

element {
    orientation: vertical;
    background-color: #64727D;
    border: 2px;
    border-color: #64727D;
}

element selected {
    border: 2px;
    border-color: #5294e2;
}

element-icon {
    size: 200px;
}

element-text {
    font: "monospace 1px";
    text-color: transparent;
    expand: false;
    horizontal-align: 0.5;
}
```

**To change the background color** (e.g. to match a different bar/theme), swap every `#64727D` for your color. `#5294e2` is the hover/selection highlight color — change separately if desired.

**To change grid size:** `columns` × `lines` controls the grid (4×4 = 16 wallpapers visible at once, scroll for more). Window `width`/`height` need to roughly match: `width ≈ (icon_size × columns) + (spacing × (columns-1)) + (padding × 2)`, same formula for height using `lines` instead of `columns`.

---

## 6. Bind it in Hyprland

In `~/.config/hypr/hyprland.lua`:

```lua
hl.bind("SUPER + W", hl.dsp.exec_cmd("~/.config/hypr/scripts/wallpaper-menu.sh"))
```

Reload:
```bash
hyprctl reload
```

---

## Notes / things that did NOT work (so you don't retry them)

- **waypaper** — works, but rescans/recaches the entire wallpaper folder on every single launch (no persistent cache), causing a multi-second toolbar flash before zen mode kicks in. Not worth it with a large wallpaper collection.
- **rofi filebrowser mode** (`rofi -show filebrowser`) — does NOT reliably render real image thumbnails despite documentation suggesting it uses XDG thumbnailers (`tumbler`) automatically. Confirmed tumbler itself works fine via direct D-Bus calls, but rofi's filebrowser mode never actually queries it. Generic file-type icons only.
- **`element-icon { size: 100%; }`** — not a supported value in rofi's rasi format; breaks rendering entirely (white screen).
- **`element { height: Npx; }`** — rofi parses this without error (confirmed via `-dump-theme`) but does not actually honor it for layout — row height is still driven entirely by `element-icon size`, which is why thumbnails need to be pre-cropped to square rather than relying on rofi to letterbox/fit non-square images into rectangular cells.
- **Importing an existing rofi theme** (e.g. `@import "Arc-Dark.rasi"`) — caused stubborn grey/white background boxes around thumbnails that were very difficult to override, due to theme-specific state selectors (`element normal.normal`, etc.) taking priority over generic `element` rules. Building the theme from scratch avoided this.

---

## Quick troubleshooting

| Symptom | Fix |
|---|---|
| Clicking a wallpaper opens it in an image viewer instead of setting it | You're missing `-filebrowser-command 'echo'` — not applicable here since this guide uses dmenu mode, not filebrowser mode |
| Thumbnails are generic icons, not real images | Confirm `~/wallpapers/.thumbs/` actually contains generated thumbnail files — rerun `make-square-thumbs.sh` |
| Grid scrolls sideways instead of up/down | Add `flow: horizontal;` to `listview` (counterintuitively, this is what makes it fill rows left-to-right and scroll vertically) |
| Only some rows render even though `lines` is set higher | Recalculate `window height` using the formula above — rofi clips rows that don't fit rather than shrinking them |
| Background still grey/white despite `background-color` overrides | Make sure you are NOT importing an external theme; add a global `* { background-color: ...; }` rule before more specific selectors |
