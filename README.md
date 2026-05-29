# jira-sdd-flow-plugin

Claude Code plugin for **Spec-Driven Development (SDD)** over Jira and Confluence.

Refines vague ideas into detailed specs, produces **intent → requirements → spec** documentation in Confluence and linked Jira issues, and drives implementation from the resulting spec.

> Status: scaffold. Skills, commands, agents, hooks, and MCP servers are added incrementally.

## What's in the box

### Skills

| Skill                       | Trigger                                                                                                     | What it does                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
|-----------------------------|-------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `setup-jira-sdd-environment` | First install, "set up the plugin", "configure env vars", `/setup-jira-sdd-environment`, or any env-var error from another skill | Walks the user through the env vars this plugin needs (Jira/Confluence URLs, credentials, project key, space key, SDD root page ID). Detects which are already set, prompts only for the missing ones with per-variable guidance, produces a copy-pasteable shell/`.env` block (token line left blank by design), and verifies the Atlassian MCP can actually reach Jira and Confluence with the configured values. Run this first.                                                                                                                                                                                              |
| `grill-jira-ticket`         | "grill this ticket", `/grill-jira-ticket`, a Jira issue key, or an idea to grill                            | Interviews the user about a Jira ticket (or free-form plan) using a decision-tree grill. For an existing ticket: cross-checks against parent and siblings, flags inconsistencies, writes the resolved understanding back, and reconciles related tickets with comments explaining the change. For a free-form prompt: creates a new Jira ticket from the outcome, prompting for issue type and optional sprint before creating. Based on Matt Pocock's [`grill-me`](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md) skill.                                                                  |
| `publish-sdd-to-confluence` | "publish SDD pages", `/publish-sdd-to-confluence`, or a Jira issue key with a request to push to Confluence | Takes a (grilled) Jira ticket and publishes it to Confluence as a four-page spec set: a landing page titled `[KEY]: Change name` with a compressed summary and table of contents, plus three children — **Intent**, **Requirements** (each a testable statement led by a stable `REQ-<slug>` ID), and **Specs** (exhaustive Gherkin scenarios, each tagged with a stable `@SCN-NNN` ID so subtasks can reference a subset, and each citing the requirement(s) it satisfies with `@REQ-<slug>` tags on the same tag line). The `REQ-<slug>` ↔ `@REQ-<slug>` pairing closes the requirement→scenario link so it is machine-checkable rather than re-derived from prose. The landing page is created under a configurable Confluence root page, and a remote link back to the landing page is added to the Jira ticket.                                                                                                                                                                                               |
| `create-subtasks`           | "split this ticket", "create subtasks for PROJ-123", `/create-subtasks`, or a Jira parent key with a request to subdivide | Splits a published SDD parent into Jira subtasks where each subtask delivers a fully working increment for a real consumer (a deployable API service, a runnable CLI, a shipping UI). The split is judgement-based, optionally guided by a user hint (e.g. `frontend and backend`, `one per repo`). Each subtask references a subset of the parent's `@SCN-NNN` scenarios under a `## Covers scenarios` heading; overlap between subtasks is allowed but every scenario must be claimed by at least one. Inventories existing subtasks first and reconciles stale or untagged ones (trim retired IDs, retrofit a claim, or post a close-recommendation) before proposing new ones. Refuses horizontal slices that do not ship on their own. |
| `implement-lite`            | "fast lane", "lightweight implement", "implement without specs", "just build this ticket", `/implement-lite`, or a grilled Jira issue key with no Confluence SDD | **Fast lane for trivial changes.** Implements a grilled Jira ticket directly from the **Acceptance criteria** in its description, skipping the four-page Confluence SDD set entirely (the "easy, not complex" lane). Numbers the criteria `AC-1`, `AC-2`, …, discovers the host project's language and test framework, writes at least one passing test per criterion, and tags each test with an `SDD-LITE: <KEY> AC-N` comment — a namespace deliberately distinct from the full path's `SDD: <KEY> @SCN-NNN` tags so `reconcile-sdd` never picks up lite tests. Reads and writes nothing in Confluence and never creates subtasks. Refuses (and routes to the full path) when a Confluence SDD landing page already exists, when the change spans multiple deployable increments, or when traceability/reconciliation is needed; refuses (and routes to `grill-jira-ticket`) when the criteria are missing or untestable. Reports a coverage table back to the ticket and offers an upgrade hint to `publish-sdd-to-confluence` → `implement-sdd-spec` if the change turned out bigger than a trivial increment. |
| `implement-sdd-spec`        | "implement this ticket", `/implement-sdd-spec`, or a Jira issue key (parent or subtask) with a request to build from the SDD | Takes a Jira ticket whose SDD has been published to Confluence — either the parent or one of its subtasks — fetches its **Intent**, **Requirements**, and **Specs** pages from the parent, and implements the change in the host codebase. On a subtask invocation, walks up to the parent for the SDD and only implements the `@SCN-NNN` scenarios the subtask claims; on a parent invocation, implements the full Specs page. Builds its requirement→scenario traceability column from the `@REQ-<slug>` citation tags rather than re-deriving it from prose. Discovers the project's language, test framework, and conventions from its own manifests; writes production code plus tests so that every in-scope Gherkin scenario is covered by at least one passing test. **Tags every test it writes with a `SDD: <KEY> @SCN-NNN` comment** on the line immediately above the test declaration (in the host language's single-line comment style), so the test↔scenario link is machine-readable and survives refactors. Before writing a new test, looks for an existing tagged test for the scenario and either reuses it (same testing layer, matches current Gherkin), updates it (Gherkin changed since), or asks the user about adding a parallel test (different layer, e.g. frontend on top of an existing backend test). Builds a coverage matrix scoped to the slice, refuses to weaken specs to make tests pass, and reports the result back to the invoked ticket as a comment. When it completes the last unresolved subtask of a parent (or finishes a no-subtask parent in one pass), hints at running `reconcile-sdd` next. Refuses to invent specs: if SDD pages or scenario claims are missing, it routes the user to `publish-sdd-to-confluence`, `create-subtasks`, or `grill-jira-ticket` as appropriate. |
| `reconcile-sdd`             | "reconcile the SDD", "close out PROJ-123", `/reconcile-sdd`, or a Jira issue key (parent or subtask) once the work is resolved | Closes the SDD loop after a ticket and all its subtasks are resolved (`resolutiondate` set). Greps the test tree for the `SDD: <PARENT_KEY> @SCN-NNN` tags written by `implement-sdd-spec` to find which tests cover which scenarios, treats the tests as the source of truth at reconcile-time, and proposes targeted edits to the parent's **Intent**, **Requirements**, and **Specs** pages where the Gherkin or surrounding prose drifted from what was built. Does **not** re-verify coverage (that is `implement-sdd-spec`'s responsibility); only detects content drift. Also verifies the requirement→scenario→test chain mechanically by joining `REQ-<slug>` IDs to their `@REQ-<slug>` citations and `@SCN-NNN` tags, flagging uncovered requirements (no scenario cites them), uncited scenarios (satisfy no stated requirement), and dangling citations. For legacy tests written before tags existed, offers to retrofit the tag with explicit user approval (the only test-source edit this skill is ever permitted to make). Flags genuine gaps (TODO scenarios that never got a test, scenarios with no tagged or legacy test, orphan tags pointing at retired scenarios, and chain link defects) without auto-fixing them. Then, optionally, lets the user pick sibling SDD landing pages under the configured root and reconciles their Specs against the now-canonical as-built scenarios, with per-page approval and a Jira comment on each updated sibling's source ticket. Never edits test assertions or production code, never transitions Jira status, and never writes to a sibling page without also commenting on its source ticket. |

### Bundled MCP servers

| Server      | Package                          | Used by                                            |
|-------------|----------------------------------|----------------------------------------------------|
| `atlassian` | [`sooperset/mcp-atlassian`](https://github.com/sooperset/mcp-atlassian) (`uvx mcp-atlassian`) | All Jira and Confluence reads and writes |

Credentials and endpoints are passed through `env` from the variables listed in [Configuration](#configuration). Nothing is hardcoded.

## Installation

### Prerequisites

| Tool | Why | Install |
|------|-----|---------|
| [Claude Code](https://claude.ai/code) | Runs the plugin | See Claude Code docs |
| [`uv`](https://docs.astral.sh/uv/) | Required by the bundled `mcp-atlassian` MCP server (`uvx mcp-atlassian`) | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |

Verify both are on your PATH before continuing:

```bash
claude --version
uvx --version
```

### 1. Install into Claude Code

Register this repo as a marketplace, then install by name:

```
/plugin marketplace add https://github.com/frtelg/jira-sdd-flow-plugin
/plugin install jira-sdd-flow-plugin@jira-sdd-flow-plugin
```

Verify it loaded:

```
/plugin list
```

### 3. Configure environment variables

The bundled MCP server and every skill in this plugin read connection details
from environment variables. Run the setup skill to be guided through them:

```
/setup-jira-sdd-environment
```

Or set the variables manually in your shell rc file before starting Claude Code
(see [Configuration](#configuration) for the full list).

> **Token safety.** Never paste your Jira API token into the chat window. The
> setup skill will instruct you to set `JIRA_API_TOKEN` out-of-band in your
> shell or secrets manager.

### 4. Restart Claude Code

The MCP server reads environment variables at startup. If Claude Code was
already running when you set the variables, restart it so the new values take
effect.

### 5. Verify connectivity

The setup skill runs a connectivity check automatically (Phase 4). To rerun it
at any time:

```
/setup-jira-sdd-environment
```

When all checks pass — Jira auth, project key, Confluence auth, and SDD root
page — the plugin is ready to use.

## Layout

```
jira-sdd-flow-plugin/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest (name, version, description, author)
│   └── marketplace.json     # marketplace catalog (makes this repo self-hostable as a marketplace)
├── skills/                  # SKILL.md files, one folder per skill
├── commands/                # slash-command prompts (.md), invoked as /<name>
├── agents/                  # subagent definitions (.md with frontmatter)
├── hooks/
│   └── hooks.json           # event-driven hooks (PreToolUse, PostToolUse, etc.)
├── .mcp.json                # MCP servers bundled with the plugin
├── .gitignore
└── README.md
```

Empty component folders are kept with `.gitkeep`. Remove the marker when the first real file lands.

## Adding components

### Skills

Each skill lives in `skills/<skill-name>/SKILL.md`. Minimal frontmatter:

```markdown
---
name: skill-name
description: When this skill should trigger and what it does.
---

Body: the actual instructions Claude follows when the skill activates.
```

### Commands

Each command is a single markdown file in `commands/<name>.md`. The filename becomes the slash command (`/<name>`). Optional frontmatter sets metadata; the body is the prompt template.

### Agents

Each subagent is `agents/<name>.md` with frontmatter:

```markdown
---
name: agent-name
description: When to spawn this subagent.
tools: Read, Grep, Bash
---

System prompt for the subagent.
```

### Hooks

Wire shell commands to Claude Code events in `hooks/hooks.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "echo edited >&2" }
        ]
      }
    ]
  }
}
```

### MCP servers

Declare bundled MCP servers in `.mcp.json` (same shape as user-level config). Pass endpoints and credentials through the `env` block — never inline literal hostnames or tokens:

```json
{
  "mcpServers": {
    "atlassian": {
      "command": "npx",
      "args": ["-y", "some-atlassian-mcp-server"],
      "env": {
        "JIRA_BASE_URL": "${JIRA_BASE_URL}",
        "JIRA_USER_EMAIL": "${JIRA_USER_EMAIL}",
        "JIRA_API_TOKEN": "${JIRA_API_TOKEN}",
        "CONFLUENCE_BASE_URL": "${CONFLUENCE_BASE_URL}"
      }
    }
  }
}
```

## Configuration

The plugin is **organisation-agnostic**: it ships with no hardcoded servers, project keys, space keys, or tenant names. Every connection detail and identifier is read from environment variables at runtime. Set the ones a given skill or MCP server requires before invoking it.

| Variable                | Purpose                                                       | Required by                  | Example                            |
|-------------------------|---------------------------------------------------------------|------------------------------|------------------------------------|
| `JIRA_BASE_URL`         | Jira instance URL                                             | any Jira skill / MCP server  | `https://example.atlassian.net`    |
| `JIRA_USER_EMAIL`       | Jira account email for API auth                               | any Jira skill / MCP server  | `user@example.com`                 |
| `JIRA_API_TOKEN`        | Jira API token                                                | any Jira skill / MCP server  | `***`                              |
| `JIRA_PROJECT_KEY`      | Default Jira project for ticket creation/search               | ticket-creating skills       | `PROJ`                             |
| `CONFLUENCE_BASE_URL`   | Confluence instance URL                                       | any Confluence skill         | `https://example.atlassian.net/wiki` |
| `CONFLUENCE_SPACE_KEY`  | Default Confluence space for spec/intent pages                | doc-writing skills           | `DOCS`                             |
| `CONFLUENCE_SDD_ROOT_PAGE_ID` | Numeric Confluence page ID that SDD landing pages are created under (extract from a Confluence page URL, e.g. `.../pages/1234567890/Specs` → `1234567890`) | `publish-sdd-to-confluence` | `1234567890` |

This table grows as new skills/MCP servers land. Adding a new env var requires adding a row here in the same change.

Skill and command bodies refer to these values as `${VAR_NAME}` placeholders rather than literal strings.

## Two paths: lightweight vs full SDD

The plugin offers two lanes. Pick the lane that fits the size of the change.

### Lightweight path (fast lane)

For trivial, single-increment changes where a four-page spec set would be
ceremony, not clarity — the "easy, not complex" lane:

```
/grill-jira-ticket PROJ-123      # sharpen the ticket; grill writes Acceptance criteria into it
/implement-lite PROJ-123         # build straight from those criteria, no Confluence
```

`implement-lite` reads the **Acceptance criteria** the grill writes into the
ticket description, numbers them `AC-1`, `AC-2`, …, writes one passing test
per criterion (tagged `SDD-LITE: <KEY> AC-N`), and comments a coverage table
back on the ticket. It touches no Confluence pages and creates no subtasks.

Use the fast lane when the change ships as a single increment, needs no
requirement→scenario traceability, and won't be reconciled against sibling
specs. If those constraints don't hold, take the full path.

### Full SDD path

For changes that warrant a published spec, traceability, a subtask split, or
later reconciliation:

```
/grill-jira-ticket PROJ-123
/publish-sdd-to-confluence PROJ-123   # Intent / Requirements / Specs + landing page
/create-subtasks PROJ-123             # optional: split into shippable increments
/implement-sdd-spec PROJ-123          # build from the published Specs, with traceability
/reconcile-sdd PROJ-123               # after resolve: align pages with as-built tests
```

### Promoting a fast-lane ticket

A ticket built via the lite path can be upgraded later: run
`/publish-sdd-to-confluence` to generate the spec set, then
`/implement-sdd-spec`. `implement-lite` itself refuses to run once a
Confluence SDD landing page exists, so the published Specs stay the single
source of truth from that point on.

## SDD workflow (target)

The end state this plugin is building toward:

1. **Refine** — vague Jira ticket or idea → grilled into a sharp problem statement.
2. **Intent** — captured as a Confluence page linked to the Jira parent.
3. **Requirements** — derived from intent, written back to Jira + Confluence.
4. **Spec** — technical design with acceptance criteria, attached to the ticket.
5. **Implement** — branch + MR driven from the spec, with traceability back to each requirement.

Each phase will land as its own skill/command/agent. This README tracks which pieces exist as they are added.

## Design principles

- **Organisation-agnostic.** No hardcoded servers, project keys, space keys, brands, or tenant terminology in plugin content. All such values come from env vars (see [Configuration](#configuration)).
- **Locale-neutral.** The plugin makes no assumptions about language, date format, or currency. Locale is the operator's choice at use time, not a plugin default.
- **Synthetic placeholders only.** Examples use `https://example.atlassian.net`, `PROJ-001`, `DOCS`, `participant_001` — never real tenant data.
- **README is the source of truth.** Any change to structure, components, manifest, env vars, or install steps must update this README in the same commit.

## Credits

- **`grill-jira-ticket`** is a Jira-aware adaptation of the `grill-me` skill by [Matt Pocock](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md). The decision-tree interview pattern, one-question-per-turn cadence, and recommended-answer format come from his original.
