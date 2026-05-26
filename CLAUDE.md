# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A **Claude Code plugin** (`jira-sdd-flow-plugin`) for Spec-Driven Development over Jira and Confluence. It contains only plugin assets — markdown files, JSON manifests, optional hook scripts. There is no application code, no build step, no test suite, and no linter. "Working in this repo" means authoring plugin components, not running tooling.

## Plugin layout that matters

The plugin manifest lives at `.claude-plugin/plugin.json`. Bump `version` (semver) on user-visible changes.

Component directories are loaded by Claude Code based on convention:

- `skills/<name>/SKILL.md` — model-invoked behaviours. Frontmatter `name` and `description` are mandatory; `description` is what the model matches against to decide whether to activate the skill, so be precise about trigger conditions.
- `commands/<name>.md` — slash commands. Filename becomes `/<name>`. Body is the prompt template.
- `agents/<name>.md` — subagent definitions. Frontmatter takes `name`, `description`, and an optional comma-separated `tools` list.
- `hooks/hooks.json` — event hooks (`PreToolUse`, `PostToolUse`, `Notification`, etc.) with `matcher` regex and shell commands.
- `.mcp.json` — MCP servers shipped with the plugin (same shape as a user-level `.mcp.json`).

Empty component dirs are kept with `.gitkeep`. Delete the marker when the first real file lands in that dir.

## Editing JSON files

`.claude-plugin/plugin.json`, `hooks/hooks.json`, and `.mcp.json` are loaded by Claude Code at plugin activation. A syntax error breaks the whole plugin. After editing any of them, validate:

```bash
python3 -m json.tool < <file>
```

## Incremental authoring

The owner is building this plugin one skill at a time. Default to:

- Adding **one** skill, command, or agent per request unless explicitly told otherwise.
- Not pre-creating placeholder files for SDD phases ("intent", "requirements", "spec", "implement") that haven't been requested yet.
- Not modifying `plugin.json` keywords or version unless asked.

The README's "SDD workflow (target)" section lists the intended phases; treat that as a roadmap, not a checklist to bulk-implement.

## Hard rules

### 1. Plugin must stay user/system/organization agnostic

This plugin is intended to be reusable beyond any single team or tenant. Nothing in `skills/`, `commands/`, `agents/`, `hooks/`, or `.mcp.json` may hardcode an organisation, server, project, or space.

- No Jira/Confluence/GitLab hostnames, no project keys, no space keys, no board IDs, no group names, no branded copy in skill or command bodies.
- All such values come from environment variables. Reference them as `${JIRA_BASE_URL}`, `${JIRA_PROJECT_KEY}`, `${CONFLUENCE_BASE_URL}`, `${CONFLUENCE_SPACE_KEY}` etc. in prose, and pass them through `.mcp.json` server `env` blocks rather than literal strings.
- Examples in docs use synthetic placeholders only: `https://example.atlassian.net`, `PROJ-001`, `DOCS`, `participant_001`.
- No assumed locale defaults (UK English, EUR, DD-MM-YYYY etc) baked into plugin content. Those belong to the operator at use time, not the plugin.
- No references to the author's employer, charities, brands, or tenant-specific terminology.

If a skill genuinely requires an org-specific concept (e.g. "Atlassian Cloud only"), state the constraint in the skill description but still parameterise the values.

### 2. README is the source of truth — keep it in sync

Every change that alters what the plugin contains, requires, or exposes must update `README.md` in the same commit/MR. Do not defer.

Triggers:
- New or removed skill/command/agent/hook/MCP server.
- Manifest field change (name, version, keywords, description).
- New required env var, install step, or configuration knob.
- Layout change.
- Renames.

The README's Configuration section lists every env var the plugin reads. Adding a new one to a skill without listing it there is a bug.

## When adding MCP servers

If a skill depends on an MCP server (Atlassian, GitLab, etc.), declare it in `.mcp.json` rather than assuming the user already has it configured at the user or project level. Plugin-bundled servers travel with the install. Their credentials and endpoints must come from env vars via the `env` field, never literal values.
