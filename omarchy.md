# Voquill Setup on Omarchy (Hyprland/Wayland)

## Prerequisites

```bash
# System dependencies
sudo pacman -S webkit2gtk-4.1 xdotool vulkan-headers vulkan-icd-loader vulkan-tools shaderc

# libxdo compatibility (Arch ships libxdo.so.4, Voquill expects .3)
sudo ln -s /usr/lib/libxdo.so.4 /usr/lib/libxdo.so.3
```

## Build from Source

```bash
cd ~/voquill

# Install JS dependencies
npx pnpm install

# Build all monorepo packages (required before Tauri app works)
npx pnpm run build

# Build the Tauri app (CPU transcription)
cd apps/desktop/src-tauri
cargo build --no-default-features

# Build the GPU transcription sidecar (Vulkan for NVIDIA)
cd ~/voquill/packages/rust_transcription
cargo build --bin rust-transcription-gpu --features gpu-vulkan

# Build the pill overlay
cd ~/voquill/packages/rust_gtk_pill
cargo build

# Copy pill binary to resources
cp ~/voquill/packages/rust_gtk_pill/target/debug/voquill-gtk-pill \
   ~/voquill/apps/desktop/src-tauri/target/debug/resources/voquill-gtk-pill
```

## Desktop Integration

### Launcher script

Create `~/.local/share/omarchy/bin/voquill-launch`:

```bash
#!/usr/bin/env bash

eval "$(mise activate bash 2>/dev/null)"

# Check if the actual Voquill binary is running
if pgrep -x Voquill >/dev/null 2>&1; then
    notify-send "Voquill" "Already running" -t 2000
    exit 0
fi

cd "$HOME/voquill/apps/desktop" || exit 1

# Start Vite dev server if not running
if ! lsof -ti:1420 >/dev/null 2>&1; then
    VITE_FLAVOR=emulators npx vite --port 1420 &>/dev/null &
    sleep 2
fi

notify-send "Voquill" "Voice dictation started (Ctrl+Space)" -t 3000

# Run Voquill in foreground so the systemd scope stays alive
# (Omarchy launches apps via uwsm/systemd-run --scope)
export GDK_SCALE=1
export GDK_DPI_SCALE=1
export GDK_BACKEND=x11
export WEBKIT_DISABLE_DMABUF_RENDERER=1
"$HOME/voquill/apps/desktop/src-tauri/target/debug/Voquill" >/dev/null 2>&1
```

**Note:** The `mise activate` line is required because the desktop menu launcher doesn't source your shell profile, so `npx`/`node` (installed via mise) won't be in PATH without it.

```bash
chmod +x ~/.local/share/omarchy/bin/voquill-launch
```

### App icon

```bash
# Extract icon from repo
mkdir -p ~/.local/share/icons/hicolor/128x128/apps
mkdir -p ~/.local/share/icons/hicolor/32x32/apps
cp ~/voquill/apps/desktop/src-tauri/icons/icon.png \
   ~/.local/share/icons/hicolor/128x128/apps/voquill.png
convert ~/.local/share/icons/hicolor/128x128/apps/voquill.png \
   -resize 32x32 ~/.local/share/icons/hicolor/32x32/apps/voquill.png
```

### Desktop entry

Create `~/.local/share/applications/voquill.desktop`:

```ini
[Desktop Entry]
Name=Voquill
Comment=AI Voice Dictation
Exec=/home/YOUR_USER/.local/share/omarchy/bin/voquill-launch
Icon=/home/YOUR_USER/.local/share/icons/hicolor/128x128/apps/voquill.png
Terminal=false
Type=Application
Categories=Utility;Audio;
Keywords=voice;dictation;speech;transcription;whisper;
StartupNotify=false
```

### Hyprland hotkey

Add to `~/.config/hypr/hyprland.conf`:

```
source = ~/.config/hypr/voquill-hotkeys.conf
```

Voquill auto-generates `~/.config/hypr/voquill-hotkeys.conf` with the Ctrl+Space binding.

## Enable GPU Transcription

After first launch, set GPU as the default device in the database:

```bash
sqlite3 ~/.config/com.voquill.desktop/voquill.db \
  "UPDATE user_preferences SET gpu_enumeration_enabled = 1, transcription_device = 'gpu:0';"
```

Then restart the app. The GPU (NVIDIA GeForce RTX 3060 via Vulkan) will be pre-selected in Settings > AI Transcription > Local > Processing device.

## Troubleshooting

### "migration was previously applied but has been modified"
Never modify existing SQL migration files in `src-tauri/src/db/migrations/`. If you need to change a default, do it in the TypeScript code or create a new migration.

### App won't start from app menu
- The launcher script must include `eval "$(mise activate bash 2>/dev/null)"` before any `npx`/`node` commands, because the desktop menu doesn't source your shell profile.
- Omarchy launches apps via `uwsm_app-daemon` / `systemd-run --scope`. The Voquill binary must run in the **foreground** (not backgrounded with `&`) in the launcher script, otherwise the scope ends and kills it.

### Blank/white window
Run `npx pnpm run build` from the repo root to build all monorepo packages. WebKit2GTK also needs `WEBKIT_DISABLE_DMABUF_RENDERER=1`.

### Pill overlay is an opaque box
The pill must run on native Wayland (not X11). The `env_remove("GDK_BACKEND")` in `pill_process.rs` strips the parent's `GDK_BACKEND=x11` so the pill uses gtk-layer-shell.

### "Text file busy" errors when rebuilding
Kill all Voquill and pill processes before rebuilding:
```bash
pkill -9 -x Voquill; pkill -9 -f voquill-gtk-pill
```

## Key Notes

- **WebKit2GTK** requires `GDK_BACKEND=x11` to avoid Wayland protocol crashes
- **Pill overlay** runs natively on Wayland via gtk-layer-shell for proper transparency/rounded corners (GDK_BACKEND is stripped from its environment)
- **Monitor tracking** uses `hyprctl cursorpos` to get Wayland-native cursor coordinates since X11 and Wayland have different coordinate spaces
- **Text injection** uses ydotool on Wayland for paste keystrokes
- Voquill runs in the foreground of its systemd scope; use **Ctrl+Space** to toggle dictation
