# Project Memory & Skills - Quick Start Guide

## Overview

This document replaces the old "Knowledge Base" CLI guide.

The current Snailer CLI provides project memory via:

- **Skills**: reusable workflows you can install and run (early/alpha)
- **NOTES.md**: lightweight persistent project notes the agent can read/write
- **CodeWiki**: an experimental local code indexer (MVP)

If you are looking for the old commands like `snailer kb ...` or `snailer knowledge-base ...`, note that they are **not available** in the current CLI.

---

## Skills

### Where skills live

Snailer loads skills from:

- User skills: `~/.snailer/skills/`
- Repo-local skills: `./.snailer/skills/`

### List skills

```bash
snailer skills list
```

### Create a new skill template

```bash
snailer skills new my-skill
```

This creates a folder with a `SKILL.toml` manifest and starter files.

### Inspect a skill

```bash
snailer skills inspect <skill_id>
```

### Install a skill

```bash
snailer skills install <git-url-or-local-path>
```

Note: installation is currently a stub in the implementation; if this changes, this doc should be updated alongside the code.

---

## Persistent Notes (NOTES.md)

Snailer provides two built-in tools for persistent project notes:

- `read_notes`
- `write_notes`

These tools operate on a `NOTES.md` file in your project root and are meant to store:

- architecture decisions
- known bugs
- conventions and patterns
- important constraints

### Typical usage

- Start Snailer (interactive):

```bash
snailer
```

- Ask the agent to record a decision:

```text
Please write a short decision note in NOTES.md under "Decisions" about why we chose X.
```

You can also edit `NOTES.md` manually in your editor—Snailer will treat it as regular project context.

---

## CodeWiki (experimental)

CodeWiki is an MVP local indexer that can scan Rust files and summarize symbols.

### CLI (MVP)

```bash
snailer wiki index
snailer wiki insights
snailer wiki export
```

### REPL

In interactive mode, you can also use slash commands like `/codewiki` (exact subcommands may evolve).

---

## Migration Notes

If you have older docs/examples referencing:

- `snailer kb ...`
- `snailer knowledge-base ...`
- `@kb ...` in prompts

Update them to the mechanisms above (Skills, NOTES.md, CodeWiki). If you need a true document store/search knowledge base, track it as a roadmap item and keep docs clearly marked as "planned" until the CLI exists.
