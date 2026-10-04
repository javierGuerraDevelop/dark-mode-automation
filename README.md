# dark-mode-automation

Automatically switches the desktop and app theme based on the time of day, and adjusts screen brightness to match.

- **08:00–20:00 (US Eastern)** → light theme, brightness 80%
- **20:00–08:00 (US Eastern)** → dark theme, brightness 30%
- **On login/boot** → checks the current time and applies the correct state
- **On wake from suspend** → systemd catches up the missed timer
- **Ghostty** → theme symlink switched and reloaded via `SIGUSR2` (same as `ctrl+shift+,`), no restart needed

Each switch updates GNOME via `gsettings` (`color-scheme`, `gtk-theme`), points `~/.config/ghostty/current-theme.conf` at `~/.config/ghostty/themes/<mode>.conf` (included by Ghostty's `config-file = current-theme.conf`) and reloads Ghostty, then sets display brightness. Neovim follows the system color scheme on its own via `auto-gnome-theme.nvim`.

## Layout

| File | Installed location |
|---|---|
| `bin/theme-set` | `~/.local/bin/theme-set` |
| `bin/lg-tv` | `~/.local/bin/lg-tv` |
| `systemd/theme-set.service` | `~/.config/systemd/user/theme-set.service` |
| `systemd/theme-set.timer` | `~/.config/systemd/user/theme-set.timer` |

## Install

```bash
sudo pacman -S ddcutil   # HKC monitor brightness (DDC/CI)
install -d ~/.local/bin ~/.config/systemd/user
install -m755 bin/theme-set bin/lg-tv ~/.local/bin/
install -m644 systemd/theme-set.service systemd/theme-set.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable theme-set.service
systemctl --user enable --now theme-set.timer

# One-time TV pairing (accept the prompt on the TV)
lg-tv pair <TV-IP>
```

## Commands

```bash
# Apply the correct theme for the current time now
theme-set auto

# Force a theme manually
theme-set dark
theme-set light

# Reconcile display brightness only (no theme or Ghostty changes)
theme-set --brightness-only auto

# LG TV brightness (over the local network)
lg-tv get
lg-tv set 30
lg-tv pair <TV-IP>       # one-time, or after a TV reset

# Reload a running Ghostty by itself (same as ctrl+shift+,)
pkill -USR2 -x ghostty

# Show the schedule (next run)
systemctl --user list-timers theme-set.timer

# Run the service immediately (same as `theme-set auto`)
systemctl --user start theme-set.service

# Check the last run / errors
systemctl --user status theme-set.service
journalctl --user -u theme-set.service

# Pause / resume the schedule
systemctl --user disable --now theme-set.timer
systemctl --user enable --now theme-set.timer

# Reload after editing the timer
systemctl --user daemon-reload
systemctl --user restart theme-set.timer
```

## Brightness

Brightness values live at the top of `theme-set`:

- `BRIGHTNESS_DARK` / `BRIGHTNESS_LIGHT` — HKC N07 via DDC/CI
- `TV_BRIGHTNESS_DARK` / `TV_BRIGHTNESS_LIGHT` — LG TV via webOS SSAP

Every DDC write is verified with a read-back and retried up to 3 times. Each run logs what it did to the journal: `journalctl --user -u theme-set.service`.

**HKC N07 (DP-2)**

- Requires `ddcutil`; displays that don't respond are skipped.
- Check with `ddcutil getvcp 10`; set with `ddcutil setvcp 10 <value>`.
- If not in the `i2c` group: `sudo usermod -aG i2c "$USER"` and re-login.

**LG TV (HDMI-A-2)**

- Requires a one-time `lg-tv pair <TV-IP>`; the key is stored in `~/.config/theme-set/lg-tv.json` (chmod 600) and reused.
- If the TV is off or unreachable, it is skipped silently and the theme switch still runs.
- `energySaving` must be off on the TV, otherwise backlight changes are ignored.
- Older firmware accepts direct `ssap://settings/setSystemSettings` writes. If the TV is ever updated, Aug 2026 firmware blocks that path; `lg-tv` then falls back automatically to the alert/luna workaround (the same method bscpylgtv/ColorControl use), which still worked as of Sept 2026.
- If the TV's IP changes, edit `~/.config/theme-set/lg-tv.json` or re-run `lg-tv pair <new-ip>` (a DHCP reservation is recommended).
- To reset pairing: delete `~/.config/theme-set/lg-tv.json` and pair again.

## Timezone

The schedule is hardcoded to US Eastern in two places:

- `theme-set.timer`: `OnCalendar=... America/New_York`
- `theme-set`: `TIMEZONE="America/New_York"`

Change both if you want a different timezone (e.g. `America/Los_Angeles`). DST is handled automatically.
