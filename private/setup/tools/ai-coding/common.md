---
title: "AI Coding Tools: Common"
created: "2026-04-06"
updated: "2026-04-07"
status: "active"
visibility: "private"
tags: [ai-tools, mcp, setup]
type: setup
---

# AI Coding Tools: Common

Shared plugins, MCP servers, and config patterns used across multiple AI coding tools.

## Shared Skills

| Skill | Purpose | Source | Savings |
|-------|---------|--------|---------|
| caveman | Token compression (~75% output reduction) | [JuliusBrussee/caveman](https://github.com/JuliusBrussee/caveman) | ~65% avg |

## Shared MCP Servers

| Server | Purpose | Used By |
|--------|---------|---------|
| — | — | — |

## Shared Environment Variables

```bash
# API keys — set in ~/.zshrc or ~/.bashrc
ANTHROPIC_API_KEY=
OPENAI_API_KEY=
GITHUB_TOKEN=
```

## Cross-Tool Tips

- Keep API keys in a single `.env` file and source it from your shell profile
- Use `$EDITOR` consistently so all tools open the same editor for file edits
- Use caveman mode ("talk like caveman", "/caveman") to cut output tokens ~75% while keeping technical accuracy

## Related

- [Claude](./providers/claude/index.md)
- [Copilot](./providers/copilot/index.md)
- [Codex](./providers/codex/index.md)
- [Opencode](./providers/opencode/index.md)
