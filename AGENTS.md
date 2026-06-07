# AGENTS.md

This repository contains public, user-installable Basilica agent skills.

## Structure

- Each installable skill is a directory under `skills/`.
- Each skill directory must contain `SKILL.md`.
- Root files such as `README.md` and `AGENTS.md` are metadata.
- Do not nest skills under `.claude/skills`, `plugins/`, or tool-specific paths.

## Skill Authoring Rules

- Keep skills user-facing and product-facing.
- Do not include internal developer-only workflow notes.
- Prefer current `basilica` CLI commands and flags over stale examples.
- Call out cost-bearing actions clearly.
- Include cleanup guidance for commands that create resources.
- Link to official Basilica docs for longer workflows instead of duplicating them.

## Current Skills

- `use-basilica`: route-first Basilica operations guide for the CLI and Python SDK.
