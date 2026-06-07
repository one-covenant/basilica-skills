# Basilica Skills

Agent skills for using Basilica from AI coding tools.

## Install

Install the Basilica CLI, then install skills:

```bash
basilica skills install
```

## Repository Format

Each directory under `skills/` containing a `SKILL.md` file is an installable
skill:

```text
basilica-skills/
  README.md
  AGENTS.md
  skills/
    use-basilica/
      SKILL.md
```

Root files are repository metadata. The Basilica CLI installer ignores root
files and installs only curated directories under `skills/`.

## Skills

- `use-basilica`: route-first Basilica operations guide for the CLI and Python SDK.
