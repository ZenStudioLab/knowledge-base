---
title: "Opencode Setup"
created: "2026-04-06"
updated: "2026-04-06"
status: "active"
visibility: "private"
tags: [ai-tools, opencode]
type: setup
---

# Setup: Opencode

Open-source terminal AI coding agent. Supports multiple model providers (Anthropic, OpenAI, local models).

## Plugins / Packages

| Name | Purpose | Install |
|------|---------|---------|
| Dynamic Context Pruning | Intelligently manages conversation context to optimize token usage | [GitHub](https://github.com/Opencode-DCP/opencode-dynamic-context-pruning) |

## Install

```bash
# via installer script
curl -fsSL https://opencode.ai/install | sh
```

## Config

- Config file: `~/.config/opencode/config.json`
- Supports `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, and others

## Tips & Gotchas

-

## Related Tools

### OpenChamber

Desktop and web interface for OpenCode AI agent.

- **Repository:** [openchamber/openchamber](https://github.com/openchamber/openchamber)
- **Language:** TypeScript
- **License:** MIT
- **Stats:** 2,723 stars, 279 forks
- **Homepage:** https://openchamber.dev/

### OpenCode Monitor

CLI tool for monitoring and analyzing OpenCode AI coding usage.

- **Repository:** [Shlomob/ocmonitor-share](https://github.com/Shlomob/ocmonitor-share)
- **Language:** Python
- **License:** MIT
- **Stats:** 248 stars, 40 forks
- **Homepage:** https://ocmonitor.vercel.app/

### OpenCode Dynamic Context Pruning

Plugin for intelligently managing conversation context to optimize token usage.

- **Repository:** [Opencode-DCP/opencode-dynamic-context-pruning](https://github.com/Opencode-DCP/opencode-dynamic-context-pruning)
- **Language:** TypeScript
- **License:** AGPL-3.0
- **Stats:** 1,833 stars, 97 forks

## Related

- [Common](../../common.md)
- [Claude](../claude/index.md)
- [Copilot](../copilot/index.md)
- [Codex](../codex/index.md)
