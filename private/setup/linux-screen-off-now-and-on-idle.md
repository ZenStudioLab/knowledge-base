---
title: "Linux: Turn the screen off now and after 5 minutes idle (GNOME laptop, Cinnamon workbox)"
created: "2026-10-30"
updated: "2026-10-30"
status: "active"
visibility: "private"
tags: [linux, ubuntu, gnome, mutter, linux-mint, cinnamon, dpms, xset, hdmi, i915, systemd]
source: ""
project: ""
publish_candidate: false
---

# Linux: Turn the Screen Off Now and After 5 Minutes Idle

## Summary

Goal: a `screenoff` command that turns the display off right away, and the display turning off by itself after 5 minutes without keyboard or mouse input. Two machines: the laptop (ThinkPad Z16 Gen 2, Ubuntu, GNOME on Wayland) and the workbox (mini PC, Linux Mint 22.1, Cinnamon 6.4 on X11, Samsung `LS27C36x` monitor over HDMI). "laptop" and "workbox" are the SSH host aliases.

Status (2026-10-30):

- **Laptop:** `screenoff` and the 5-minute watcher service are installed, and the service is enabled and running. Neither has been seen to blank the laptop screen yet.
- **Workbox:** `screenoff` runs and the system marks the monitor off, but the screen wakes again within about a second. The cause is found (the monitor drops its HDMI connection in standby). The proposed fix, forcing the HDMI port to "connected", is not yet tested. See Open Items.

## Laptop (GNOME, Wayland)

GNOME can already do the 5-minute auto-off by itself. On this laptop the setting was `0` (off) on 2026-10-29:

```bash
gsettings get org.gnome.desktop.session idle-delay   # 0 = never
gsettings set org.gnome.desktop.session idle-delay 300
```

The setup below is the custom alternative that was installed instead.

### 1. `screenoff` command

`~/.local/bin/screenoff`, made executable:

```bash
#!/usr/bin/env bash
# Turn the display off after 2s so the key release that ran this doesn't wake it.
sleep 2
busctl --user set-property org.gnome.Mutter.DisplayConfig \
  /org/gnome/Mutter/DisplayConfig \
  org.gnome.Mutter.DisplayConfig PowerSaveMode i 1
```

Moving the mouse or pressing a key wakes the screen. If it doesn't wake, switch to a text console with `Ctrl+Alt+F3`, log in, run the same `busctl` command with `i 0`, and go back with `Ctrl+Alt+F2`.

### 2. 5-minute idle watcher

`~/.local/bin/idle-screen-off.sh`, made executable. It reads GNOME's idle timer every 10 seconds:

```bash
#!/usr/bin/env bash
THRESHOLD=300000   # 5 min in ms
done_off=0
while true; do
  idle=$(busctl --user call org.gnome.Mutter.IdleMonitor \
    /org/gnome/Mutter/IdleMonitor/Core \
    org.gnome.Mutter.IdleMonitor GetIdletime | awk '{print $2}')
  if [ "$idle" -ge "$THRESHOLD" ] && [ "$done_off" -eq 0 ]; then
    busctl --user set-property org.gnome.Mutter.DisplayConfig \
      /org/gnome/Mutter/DisplayConfig \
      org.gnome.Mutter.DisplayConfig PowerSaveMode i 1
    done_off=1
  elif [ "$idle" -lt "$THRESHOLD" ]; then
    done_off=0
  fi
  sleep 10
done
```

For a quick test, stop the service, set `THRESHOLD=10000`, run the script, and don't touch anything for 20 seconds. Set it back to `300000` afterwards.

### 3. Start it at login

`~/.config/systemd/user/idle-screen-off.service`:

```ini
[Unit]
Description=Turn off screen after 5 min idle

[Service]
ExecStart=%h/.local/bin/idle-screen-off.sh
Restart=always

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now idle-screen-off.service
systemctl --user status idle-screen-off.service
# stop and remove from login:
systemctl --user disable --now idle-screen-off.service
```

`~/.bashrc` also has `idleoff` and `idleoff-stop` aliases from before the service existed. Don't use `idleoff` while the service runs, or two watchers run at once.

## Workbox (Linux Mint 22.1, Cinnamon, X11)

The GNOME scripts don't work here. Cinnamon has no `org.gnome.Mutter.*` D-Bus services.

### 5-minute auto-off

Built in. The Power Management panel controls these, and both were already `300` on 2026-10-29:

```bash
gsettings get org.cinnamon.settings-daemon.plugins.power sleep-display-ac
gsettings get org.cinnamon.settings-daemon.plugins.power sleep-display-battery
```

The monitor problem below affects this auto-off as much as `screenoff`.

### `screenoff` command

`~/.local/bin/screenoff`, made executable:

```bash
#!/usr/bin/env bash
# Turn the display off after 2s so the key release that ran this does not wake it.
sleep 2
DISPLAY=:0 xset dpms force off
```

`DISPLAY=:0` lets it work from an SSH session too: `ssh workbox ~/.local/bin/screenoff`.

On 2026-10-30 the installed copy is a debug version. It also logs to `~/screenoff.debug.log` the monitor state at 0–12 s and the input events seen in that time, and it prints one result line. Swap it back to the four lines above once the monitor problem is fixed.

### Prerequisite: the Intel graphics driver must be loaded

The workbox used to boot with `nomodeset` in `GRUB_CMDLINE_LINUX_DEFAULT`. With that, Linux doesn't load the Intel driver (`i915`) and draws the screen with a basic fallback (`simpledrm`). Signs: resolution 1024×768, the output is called `None-1`, and `/sys/class/drm/card0/device/driver` points to `simple-framebuffer`. The fallback can't power the monitor down, but `xset q` still reports `Monitor is Off`, so it looks like success.

Fix: remove `nomodeset` from `/etc/default/grub`, run `sudo update-grub`, and reboot. On 2026-10-30 the line is `GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=eDP-1:d"`, the driver is `i915`, and the monitor runs on `HDMI-1` at 1920×1080. If the desktop doesn't come up after removing `nomodeset`, SSH still works, so it can be put back the same way.

### The monitor drops its HDMI connection in standby

With the Intel driver loaded, the screen still woke right back up. A trace on 2026-10-30 showed this sequence:

```
+0.0 s  Monitor is Off   hdmi=connected      xset dpms force off
+0.6 s  Monitor is Off   hdmi=disconnected   monitor drops the HDMI link (hot-plug event)
+0.8 s  Monitor is On    hdmi=disconnected   Cinnamon reacts to the "unplug" and wakes the display
+2.3 s  Monitor is On    hdmi=connected      second hot-plug event, display stays on
```

There were no input events at any point, so the keyboard and mouse aren't the cause. The lock screen also reacts to the same unplug event, but it isn't the cause.

Turning off **Auto Source Switch** in the Samsung monitor's menu changed nothing. The trace was the same afterwards.

Proposed fix (not yet tested): make Linux treat the HDMI port as always connected, so the dropped link no longer reaches Cinnamon. It lasts until the next reboot:

```bash
echo on | sudo tee /sys/class/drm/card1-HDMI-A-1/status
# undo:
echo detect | sudo tee /sys/class/drm/card1-HDMI-A-1/status
```

If it works, make it permanent with the kernel option `video=HDMI-A-1:e` in `GRUB_CMDLINE_LINUX_DEFAULT`, then `sudo update-grub`.

### Trace command

Run on the workbox, or through `ssh workbox '…'`. It blanks the screen and prints the monitor state, the HDMI connection state and the hot-plug events for about 6 seconds:

```bash
export DISPLAY=:0
U=$(mktemp); timeout 8 udevadm monitor --kernel --property --subsystem-match=drm > "$U" 2>&1 &
sleep 0.5; xset dpms force off
for i in $(seq 1 30); do
  printf "%s %s hdmi=%s\n" "$(date +%T.%2N)" "$(xset q | grep -o 'Monitor is [A-Za-z]*')" \
    "$(cat /sys/class/drm/card1-HDMI-A-1/status)"
  sleep 0.2
done | uniq -f1 -c
wait; grep -E "^KERNEL\[|HOTPLUG" "$U"; rm -f "$U"
```

Success looks like `Monitor is Off` and `hdmi=connected` for the whole run, with no `HOTPLUG=1` lines.

## Failed Approaches and Pitfalls

### The Enter key wakes the screen
Running the off command straight away fails: releasing Enter counts as input and wakes the display. That's why both `screenoff` scripts wait 2 seconds first.

### A silent command looks broken
The first `screenoff` printed nothing. When the screen was already off, or the user was on an SSH terminal away from the monitor, a working run looked the same as a failed one. Check `DISPLAY=:0 xset q | grep "Monitor is"` before and after.

### `~/.local/bin` missing from `PATH` over SSH
A command run through `ssh workbox '…'` didn't find `screenoff` by name, even in a login shell, although the user's interactive terminal did. Over SSH, use the full path `~/.local/bin/screenoff`.

### `xset` says off, but the monitor is on
`Monitor is Off` is only what X believes. Under `nomodeset` it is always wrong. Check the kernel's view too: `cat /sys/class/drm/card*-HDMI-A-1/dpms`.

### Monitor menu settings
Auto Source Switch off didn't stop the HDMI drop on the `LS27C36x`. Don't count on a monitor menu setting to fix this.

## Open Items

- Workbox: test the forced-`on` HDMI status with the trace command. If it holds, add `video=HDMI-A-1:e` to GRUB and confirm it after a reboot.
- Workbox: swap `screenoff` back to the plain version and delete `~/screenoff.debug.log`.
- Laptop: confirm that `screenoff` and the 5-minute watcher actually blank the screen and that the screen wakes on input.
- Laptop: the watcher fires once per idle period. If the screen wakes without real input, it stays on until the next idle period.

## Related

- [Ubuntu: Make an external keyboard's F-row match the ThinkPad F-row](./ubuntu-external-keyboard-fkey-remap.md)
