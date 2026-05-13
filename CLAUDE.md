# quarkmind Workspace

**Name:** quarkmind
**Project repo:** /Users/mdproctor/claude/quarkmind
**Workspace type:** public

## Session Start

Run `add-dir /Users/mdproctor/claude/quarkmind` before any other work.

## Artifact Locations

| Skill | Writes to |
|-------|-----------|
| brainstorming (specs) | `specs/` |
| writing-plans (plans) | `plans/` |
| handover | `HANDOFF.md` |
| idea-log | `IDEAS.md` |
| design-snapshot | `snapshots/` |
| java-update-design / update-primary-doc | `design/JOURNAL.md` (created by `epic`) |
| adr | `adr/` |
| write-blog | `blog/` |

## Structure

- `HANDOFF.md` — session handover (single file, overwritten each session)
- `IDEAS.md` — idea log (single file)
- `specs/` — brainstorming / design specs (superpowers output)
- `plans/` — implementation plans (superpowers output)
- `snapshots/` — design snapshots with INDEX.md (auto-pruned, max 10)
- `adr/` — architecture decision records with INDEX.md
- `blog/` — project diary entries with INDEX.md

## Rules

- All methodology artifacts go here, not in the project repo
- Promotion to project repo is always explicit — never automatic
- Workspace branches mirror project branches — switch both together

## Routing

Per-artifact routing destinations.

| Artifact   | Destination | Notes |
|------------|-------------|-------|
| adr        | project     | lands in `docs/adr/` |
| blog       | mdproctor.github.io | via blog-routing.yaml |
| design     | project     | lands in `docs/DESIGN.md` |
| snapshots  | workspace   | |
| specs      | project     | lands in `docs/superpowers/specs/` |
| plans      | workspace   | |

Valid destinations: `project` · `workspace` · `alternative ~/path/to/repo/`

## Context Management

If the conversation is getting very long or you notice context pressure,
proactively suggest writing a handover before continuing.

@/Users/mdproctor/claude/quarkmind/CLAUDE.md
