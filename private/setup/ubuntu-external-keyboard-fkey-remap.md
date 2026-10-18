---
title: "Ubuntu: Make an external keyboard's F-row match the ThinkPad F-row"
created: "2026-10-18"
updated: "2026-10-18"
status: "in-progress"
visibility: "private"
tags: [ubuntu, linux, keyboard, input-remapper, hid_apple, thinkpad, evtest]
source: ""
project: ""
publish_candidate: false
---

# Ubuntu: Make an External Keyboard's F-Row Match the ThinkPad F-Row

## Summary

Goal: on a ThinkPad Z16 Gen 2 (Ubuntu), make F1-F12 on an external 2.4 GHz keyboard (DAREU) do exactly what the same keys do on the laptop keyboard.

Status when written: steps 1-3 are verified. Step 4 (the remap) was fixed after a daemon-log diagnosis, but the final "press F1, it mutes" test had not been confirmed yet. F9-F12 and persistence across reboots are still open.

## Why a driver setting alone does not work

The DAREU receiver reports USB id `05ac:024f` ("Apple Aluminium Keyboard"), so Ubuntu loads `hid_apple`. That driver turns the F-row into Apple media keys (brightness, scale, keyboard light, prev/play/next, mute, volume). The ThinkPad F-row is a different layout, so the same keycap sends different codes. `hid_apple fnmode` only chooses media keys vs plain F-keys. It cannot translate one layout into the other.

```
ThinkPad laptop                      DAREU receiver (05ac:024f)
event3  AT Translated Set 2 keyboard   event11 / event12  "Smart Smart Wireless Device"
event13 ThinkPad Extra Buttons         both bound to hid_apple
event4  Video Bus
```

## Laptop F-row (target)

Captured with `evtest --grab`, plain press, no Fn. A plain F1 mutes, so the laptop puts media keys first.

| Key | Code sent by the laptop | Meaning |
|---|---|---|
| F1 | `KEY_MUTE` (113) | mute |
| F2 | `KEY_VOLUMEDOWN` (114) | volume down |
| F3 | `KEY_VOLUMEUP` (115) | volume up |
| F4 | `KEY_F20` (190) | mic mute |
| F5 | `KEY_BRIGHTNESSDOWN` (224) | dim screen |
| F6 | `KEY_BRIGHTNESSUP` (225) | brighten screen |
| F7 | `KEY_SWITCHVIDEOMODE` (227) | monitor layout switch |
| F8 | `KEY_WLAN` (238) | airplane mode |
| F9 | nothing visible to Linux | handled by firmware or a non-key path |
| F10 | `KEY_CONNECT` (218) | answer-call key |
| F11 | `KEY_CANCEL` (223) | end-call key |
| F12 | `KEY_BOOKMARKS` (156) | star key |

The model-specific codes above belong to the Z16 Gen 2 (21JXA000CD). Other ThinkPad generations differ, so capture before reusing.

## Steps

### 1. Capture the laptop F-row without triggering the keys

`--grab` gives `evtest` exclusive access, so F7 does not change the monitor layout while testing. Run it with `sudo`, press F1-F12 within the 40 seconds, then paste the output.

```bash
sudo apt install evtest
sudo timeout 40 sh -c 'for e in 3 4 13; do stdbuf -oL evtest --grab /dev/input/event$e & done; wait' | grep "EV_KEY.*value 1$"
```

- The laptop keyboard (`event3`) is dead for those 40 seconds.
- Find the event numbers with `cat /proc/bus/input/devices` first. They change between boots.
- A key that prints nothing cannot be remapped from software (F9 here).

### 2. Make the external keyboard send plain F1-F12

```bash
echo 2 | sudo tee /sys/module/hid_apple/parameters/fnmode
```

`fnmode` values: 0 disabled, 1 media keys first, 2 F-keys first, 3 auto. Check it worked by pressing F5 in a terminal: it prints `~` characters (the F5 escape code) and the screen does not dim. This setting resets on reboot.

### 3. Write the Input Remapper preset

Input Remapper 2.0.1 expects the preset file to be a plain top-level JSON list of mappings, saved under the device name: `~/.config/input-remapper-2/presets/Smart Smart Wireless Device/dareu.json`.

Save this as an executable script, for example `/tmp/dareu-remap.sh`. It writes the preset and starts it, and asks for your sudo password once.

```bash
#!/usr/bin/env bash
set -euo pipefail

DEVICE="Smart Smart Wireless Device"
PRESET="dareu"
CONF="$HOME/.config/input-remapper-2"
DIR="$CONF/presets/$DEVICE"

mkdir -p "$DIR"

python3 - "$DIR/$PRESET.json" <<'PY'
import json, sys

symbols = [
    "XF86AudioMute",           # F1
    "XF86AudioLowerVolume",    # F2
    "XF86AudioRaiseVolume",    # F3
    "F20",                     # F4 (mic mute)
    "XF86MonBrightnessDown",   # F5
    "XF86MonBrightnessUp",     # F6
    "XF86Display",             # F7 (monitor layout)
    "XF86WLAN",                # F8 (airplane mode)
]
mappings = [
    {
        "input_combination": [{"type": 1, "code": 59 + i}],  # KEY_F1 = 59
        "target_uinput": "keyboard",
        "output_symbol": sym,
        "mapping_type": "key_macro",
    }
    for i, sym in enumerate(symbols)
]
with open(sys.argv[1], "w") as f:
    json.dump(mappings, f, indent=2)
PY

sudo input-remapper-control --config-dir "$CONF" --command start --device "$DEVICE" --preset "$PRESET"
```

Then press F1 on the external keyboard: it should mute. Test F7 (monitor layout) and F8 (airplane mode) last because they change the setup.

## Debugging

Read the daemon log. The unit is `input-remapper-daemon` (alias `input-remapper`). The log is the only place that shows real injection errors:

```bash
sudo journalctl -u input-remapper-daemon --no-pager -n 80
```

## Failed Approaches and Pitfalls

### Heredoc pasted with indentation
The closing `EOF` of a heredoc must be at the very start of a line. Pasted with leading spaces, the shell stays on a `>` prompt forever. Use a script file or a one-line `python3 -c`.

### `{"mappings": [...]}` wrapper
Input Remapper 2.0.1 logs `ERROR: Expected mapping to be a dict: mappings` and loads nothing. The preset file must be a bare list of mapping objects.

### "Done" does not mean injecting
`input-remapper-control --command start` prints `Done` as soon as the request is sent. If the injection fails, the evidence is in the daemon log (`ERROR: Did not grab any device`), and in `fuser -v /dev/input/event11` showing no Input Remapper process holding the device.

### Running the control tool without sudo
`input-remapper-control` as a normal user cannot read `/dev/input` and reports `Device ... is unknown`, and `--list-devices` prints nothing. Use `sudo` and pass `--config-dir "$HOME/.config/input-remapper-2"` so root looks in your home, not `/root`.

### Long commands wrapped by the terminal
A long single command pasted from chat can be split into two lines, so the first half runs alone. Prefer a short script file.

### Git on this drive
The drive reports a different owner, so git says "dubious ownership". Pass `git -c safe.directory=<repo path>` per command rather than changing the global config.

## Open Items

- Confirm the final F1-F8 test after the preset-format fix.
- F10-F12: no confirmed Input Remapper key names for the call, end-call, and star codes yet.
- F9: leave as plain F9; the laptop's F9 sends no visible event.
- Make `fnmode=2` survive reboots: `echo "options hid_apple fnmode=2" | sudo tee /etc/modprobe.d/hid_apple.conf` (then reboot or replug the receiver).
- Autoload the preset at login.

## Related

- [Wezterm: Maximize new windows on open](./wezterm-maximized.md)
