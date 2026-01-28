# Raspberry Pi Touchscreen Setup Guide

A complete guide for setting up a Raspberry Pi with automatic GUI boot and touch-friendly on-screen keyboard — no physical keyboard required after initial setup.

## Overview

This setup provides:

- Desktop that auto-starts at boot
- Touchscreen functions as mouse input
- Floating on-screen keyboard button (iPhone AssistiveTouch style)
- Power-cycle safe — no manual commands needed
- Works headlessly via SSH when needed

**Target use case:** Touchscreen-only Pi installations where you may not always have a keyboard attached.

---

## Requirements

- Raspberry Pi (3/4/5 recommended)
- Compatible touchscreen display
- SSH access or temporary keyboard for initial setup

---

## Installation Steps

### 1. Install Raspberry Pi OS with Desktop

Use the standard **Raspberry Pi OS with Desktop** image (not Lite). This avoids dependency issues with manual desktop installation.

### 2. Switch from Wayland to X11

Onboard does not work properly under Wayland. You must switch to X11:

```bash
sudo raspi-config
```

Navigate to: **Advanced Options → Wayland → X11**

### 3. Enable Auto-Login

Navigate to: **System Options → Boot / Auto Login → Desktop Autologin**

This eliminates password prompts that would be difficult without a keyboard.

Reboot after changing:

```bash
sudo reboot
```

---

## On-Screen Keyboard Setup

### Install Onboard and Accessibility Services

```bash
sudo apt update
sudo apt install onboard at-spi2-core
```

The `at-spi2-core` package enables the accessibility bus, which is required for auto-show functionality.

### Configure Onboard

Start Onboard:

```bash
onboard &
```

Open Onboard **Preferences** and configure:

**General tab:**
- ✅ Enable "Show floating icon when Onboard is hidden"
- ✅ Enable "Auto-show when editing text" (keyboard appears automatically in text fields)

**Window tab:**
- ✅ Enable "Dock to screen edge"
- Set edge to "Bottom"
- ✅ Enable "Expand to landscape/portrait"
- ❌ Disable "Window decoration" (removes title bar with minimize/maximize/close buttons)
- ✅ Enable "Force to top" (keeps keyboard above other windows)

### Position the Floating Icon

By default the floating icon appears in the top-left. To move it to the top-right corner:

```bash
dconf write /org/onboard/icon-palette/landscape/x 95.0
dconf write /org/onboard/icon-palette/landscape/y 5.0
dconf write /org/onboard/icon-palette/portrait/x 95.0
dconf write /org/onboard/icon-palette/portrait/y 5.0
```

Values are percentages (0-100) of screen position. Adjust as needed.

Restart Onboard to apply:

```bash
pkill onboard
onboard &
```

### Enable Auto-Start on Boot

Create the autostart entry:

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/onboard.desktop
```

Paste the following:

```ini
[Desktop Entry]
Type=Application
Exec=onboard
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Onboard
```

Save (Ctrl+X, Y, Enter) and reboot.

---

## How It Works

After setup, every boot will:

1. Desktop starts automatically (X11 session)
2. Onboard launches with a small floating icon
3. Tap a text field → keyboard slides up from bottom
4. Tap the X on keyboard → keyboard hides, floating icon remains
5. Tap floating icon → keyboard reappears

---

## Troubleshooting

### Keyboard doesn't type or dock properly

You're likely running Wayland instead of X11. Check with:

```bash
echo $XDG_SESSION_TYPE
```

If it says `wayland`, switch to X11 via `raspi-config` (see Installation Steps).

### Floating icon position doesn't change

Settings are stored in dconf. View current settings with:

```bash
dconf dump /org/onboard/
```

Reset all settings if needed:

```bash
dconf reset -f /org/onboard/
```

### Warnings about XInput or accessibility bus

These warnings are normal when running via SSH (tty session). They should not appear when Onboard runs in the actual desktop session.

---

## Resource Usage

| Metric | Typical Value |
|--------|---------------|
| Idle RAM | ~300–400 MB |
| CPU | Nearly idle |

This is acceptable on Pi 3/4/5.

---

## Additional Touch-Friendly Tweaks

```bash
sudo raspi-config
```

Consider:
- Disable screen blanking (keeps display always on)
- Increase UI scaling for small screens (5" or 7")

---

## Why Onboard?

Other on-screen keyboards (Matchbox-keyboard, Florence, etc.) have limitations:
- No floating toggle button
- Unpredictable auto-popup behavior
- Poor touch ergonomics

Onboard is the only option that provides the "assistive floating button" UX similar to iOS AssistiveTouch.

---

## Summary Checklist

- [x] Raspberry Pi OS with Desktop installed
- [x] X11 display server (not Wayland)
- [x] Desktop auto-login enabled
- [x] Onboard installed with at-spi2-core
- [x] Floating icon enabled and positioned
- [x] Keyboard docked to bottom edge
- [x] Auto-show on text field focus
- [x] Onboard auto-starts on boot

---

## Optional: Display Calibration

If you need help with touch calibration or display orientation, note your:
- Screen size (5", 7", etc.)
- Orientation (portrait or landscape)
- Touchscreen brand/model

These details allow for precise calibration configuration.
