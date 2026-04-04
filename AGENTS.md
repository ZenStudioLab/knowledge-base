# AGENTS.md

## Project Overview

Personal knowledge base repository for storing notes, ideas, tips, snippets, setup guides, runbooks, and selected public writing. Uses Markdown with frontmatter for all content.

## Repository Structure

```
knowledge-base/
├── catalog.md          # Alphabetical index of all content
├── private/            # Internal knowledge (not for publishing)
│   ├── inbox/          # Quick capture queue
│   ├── ideas/          # Ideas and brainstorms
│   ├── tips/           # Tips and tricks
│   ├── snippets/       # Code snippets
│   ├── setup/          # Setup guides
│   ├── ci-cd/          # CI/CD documentation
│   ├── code/           # Code-related notes
│   ├── projects/       # Project documentation
│   ├── runbooks/       # Operational runbooks
│   └── references/     # Reference materials
├── public/             # Publishable content
│   ├── blog/           # Blog posts
│   ├── notes/          # Public notes
│   └── pages/          # Static pages
└── templates/           # Content templates
```

## Catalog

The `catalog.md` file is the **single source of truth** for all content. It provides an alphabetical index that agents can query to understand what exists in the repository.

**Catalog sections:**
- `## Content` - Alphabetical list of all notes with descriptions
- `## Reference Folders` - Directory listing
- `## Templates` - Available templates

## Auto-Update Hook

When creating or deleting content, the catalog **must be updated** to maintain consistency.

### Adding New Content

1. Create the file in the appropriate folder
2. Update `catalog.md`:
   - Add entry to `## Content` section in alphabetical order
   - Include format: `- [Title](./path/to/file.md) — Type`
   - Types: Setup, Tip, Snippet, Runbook, Note, etc.

### Deleting Content

1. Remove entry from `catalog.md` `## Content` section
2. Delete the file from its folder

### Example Catalog Update

```markdown
## Content

- [New Entry](./private/setup/new-entry.md) — Setup
- [Existing Entry](./private/tips/existing.md) — Tip
```

## Content Creation Workflow

### Quick Capture

1. Drop notes into `private/inbox/`
2. Tag with date: `private/inbox/2026-04.md`
3. Later: sort into appropriate folder and update catalog

### Using Templates

Templates are in `/templates/`:
- `tip.md` - Tips and tricks
- `snippet.md` - Code snippets
- `setup.md` - Setup guides
- `runbook.md` - Operational runbooks
- `private-note.md` - Internal notes
- `public-note.md` - Publishable notes
- `idea.md` - Ideas
- `blog-post.md` - Blog content
- `page.md` - Static pages
- `project.md` - Project documentation

### Frontmatter Standard

```yaml
---
title: "Descriptive Title"
description: "Brief description of content"
tags: [tag1, tag2, tag3]
created: 2026-04-04
type: tip  # tip, setup, snippet, runbook, note, idea, blog, page
---
```

## Rules

1. Capture quickly in `private/inbox/`
2. Move reusable knowledge into correct private folders
3. Keep public content inside `public/`
4. Do not publish directly from raw private notes
5. Use templates for consistency
6. Use markdown and frontmatter for every note
7. **Always update catalog.md when creating/deleting content**

## Index Files

Each folder has a `README.md` that lists its contents. Keep these in sync when adding/removing files.

## Common Tasks

| Task | Command/Action |
|------|----------------|
| Add a tip | Create in `private/tips/` + update catalog |
| Add a setup guide | Create in `private/setup/` + update catalog |
| Add a runbook | Create in `private/runbooks/` + update catalog |
| Quick capture | Add to `private/inbox/YYYY-MM.md` |
| Publish content | Move from `private/` to `public/` + update catalog |
