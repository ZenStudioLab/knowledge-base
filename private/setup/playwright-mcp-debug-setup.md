---
title: "Playwright MCP Debug Setup"
description: "Setup Playwright MCP server with Chrome profile for instant LLM visual feedback"
tags: [playwright, mcp, debugging, chrome, setup, llm, workflow]
created: 2026-04-04
type: setup
---

# Playwright MCP Debug Setup

## Problem

LLM changes code → needs to verify → but has no visibility into the running app

## Solution

Playwright MCP + Chrome with your profile = instant visual feedback

## Setup

### MCP Config

Add to `~/.config/opencode/config.json`:

```json
{
  "mcp": {
    "playwright": {
      "type": "local",
      "command": ["npx", "-y", "@playwright/mcp@latest", "--extension"],
      "enabled": true
    }
  }
}
```

### Chrome Profile

```bash
export CHROME_USER_DATA_DIR=~/.config/google-chrome/Default
```

## Feedback Loop

1. LLM edits code
2. Hot reload fires
3. MCP sees changes instantly
4. LLM screenshots/inspects
5. Confirms fix works

## MCP Tools

| Tool | Purpose |
|------|---------|
| browser_navigate | Go to URL |
| browser_snapshot | Accessibility tree |
| browser_screenshot | Visual capture |
| browser_click/type | Interact |
| browser_console_messages | Check errors |
| browser_evaluate | Run JS in page |
