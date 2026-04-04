---
title: "Wezterm: Maximize new windows on open"
created: "2026-04-04"
updated: "2026-04-04"
status: "active"
visibility: "private"
tags: [wezterm, terminal]
source: ""
project: ""
publish_candidate: false
---

# Wezterm: Maximize New Windows on Open

## Summary
Configure Wezterm to open new windows maximized by default.

## Solution

```lua
config.initial_cols = 282
config.initial_rows = 80
```

## Failed Approaches

### `wezterm.geometry.Maximized()`
Returns nil - the geometry module wasn't accessible via `wezterm.geometry`.

### `window_frame.state = 'Maximized'`
Not a valid WindowFrameConfig field.

## Context
- Wezterm: 20240203-110809-5046fc22
- Platform: Ubuntu

## Related
- private/snippets/wezterm/