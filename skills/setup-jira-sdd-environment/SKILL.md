---
name: setup-jira-sdd-environment
description: >
  Walk the user through configuring the environment variables this plugin
  needs (Jira and Confluence endpoints, project key, Confluence space, SDD
  root page), get the bundled Atlassian MCP authorised over OAuth, and verify
  it can actually reach them. Detects which variables are already set, prompts
  only for the missing ones, produces a copy-pasteable snippet for the user's
  shell or `.env`, and runs a connectivity check at the end. Use on first
  install of the plugin, when
  a downstream skill fails with a missing-env-var error, or when the user
  asks to "set up the plugin", "configure env vars", or invokes
  /setup-jira-sdd-environment.
---

# Setup Jira SDD Environment

Help the user get this plugin into a working state by configuring the
environment variables every other skill in this plugin reads. Detect what
is already set, prompt for what is missing, persist the values where the
user wants them, and verify connectivity before declaring the setup done.

This skill is the entry point a new user should run first. Other skills
in this plugin should suggest invoking this skill when they detect a
missing or invalid env var.

## Authentication

The bundled `atlassian` MCP server is the official Atlassian remote MCP
(`https://mcp.atlassian.com/v1/mcp/authv2`). It authenticates over **OAuth**,
not an API token: the first time a skill calls it, Claude Code opens a browser
to authorise against the user's existing Jira and Confluence permissions.
There is no email or token env var to set and nothing secret to persist in
this skill. If the server is not yet authorised, the user completes the OAuth
flow by running `/mcp` in an interactive Claude Code session and authorising
`atlassian`.

## Variables this plugin reads

These are non-secret connection settings read by the **skills** — for
rendering links and choosing the default project, space, and SDD root page.
They are not credentials and are not consumed by the MCP server (which
authenticates over OAuth, see above). Authoritative list (mirror of the
README config table):

| Variable                       | Required by                        | Notes                                                                                                  |
|--------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| `JIRA_BASE_URL`                | All Jira skills (link rendering)   | Full URL with scheme, no trailing slash. Cloud example: `https://example.atlassian.net`.                |
| `JIRA_PROJECT_KEY`             | Ticket-creating skills             | Project prefix, e.g. the part before the dash in an issue key.                                         |
| `CONFLUENCE_BASE_URL`          | All Confluence skills (link rendering) | For Atlassian Cloud this is usually `${JIRA_BASE_URL}/wiki`.                                       |
| `CONFLUENCE_SPACE_KEY`         | Doc-writing skills                 | Space key, found in space settings.                                                                    |
| `CONFLUENCE_SDD_ROOT_PAGE_ID`  | `publish-sdd-to-confluence`        | Numeric page ID. Extract from a Confluence page URL: `.../pages/<ID>/<slug>` → `<ID>`.                |

If this skill is updated to add or remove a variable, the README config
table must be updated in the same change.

## Phase 1 — Detect

1. For each variable above, check whether it is currently set in the
   environment. Use a single shell call like `printenv` or
   `env | grep -E '^(JIRA_|CONFLUENCE_)'`.
2. Render a status table to the user. Two columns: variable, status.
   Status is one of `set`, `missing`, or `empty`. It is fine to print
   the current value of each so the user can sanity-check it — none of
   these variables is secret.
3. Note any variables that are set but look wrong:
   - URLs without a scheme or with trailing slashes.
   - `JIRA_PROJECT_KEY` containing a dash or digits (project keys are
     letters/underscores).
   - `CONFLUENCE_SDD_ROOT_PAGE_ID` that is not numeric.
   Flag these as `set, looks wrong` and offer to overwrite during
   Phase 2.

## Phase 2 — Prompt for missing values

Walk the user through the missing or wrong variables, in the order they
appear in the table. One variable per turn. For each:

1. Explain in one or two sentences what the variable is and where to
   find it. Use synthetic placeholders only (`https://example.atlassian.net`,
   `PROJ`, `DOCS`, `1234567890`). Never reference a real tenant.
2. Show the recommended format and an example.
3. Ask for the value.
4. If the user pastes a value that looks wrong by the heuristics in
   Phase 1, point that out and ask them to confirm or correct before
   moving on.

None of these variables is a secret — authentication to Atlassian is
handled over OAuth by the MCP server, not by an env var (see
Authentication above).

Per-variable guidance:

- **`JIRA_BASE_URL`** — the URL you use to access Jira in a browser, up to
  and including the host. For Atlassian Cloud, it looks like
  `https://<tenant>.atlassian.net`.
- **`JIRA_PROJECT_KEY`** — open any issue in the project the plugin will
  create tickets in. The key is the prefix of the issue key (e.g. the
  `PROJ` in `PROJ-123`).
- **`CONFLUENCE_BASE_URL`** — on Atlassian Cloud, this is normally
  `${JIRA_BASE_URL}/wiki`. Confirm by visiting Confluence in the browser
  and copying the URL up to and including `/wiki`.
- **`CONFLUENCE_SPACE_KEY`** — open the target space in Confluence, then
  Space settings → Space details. The key is short and uppercase.
- **`CONFLUENCE_SDD_ROOT_PAGE_ID`** — open the Confluence page that should
  act as the SDD root in your browser. Its URL contains
  `/pages/<digits>/<slug>`. The `<digits>` portion is the page ID. Paste
  only the digits, not the full URL.

## Phase 3 — Persist

Once values are collected, ask the user where to persist them. Do not
pick a destination silently. Offer three options and recommend based on
context:

1. **Shell rc file** (`~/.zshrc`, `~/.bashrc`, `~/.config/fish/config.fish`,
   etc.). Best for a single user on a single machine. Produce a block
   like:

   ```bash
   export JIRA_BASE_URL="..."
   export JIRA_PROJECT_KEY="..."
   export CONFLUENCE_BASE_URL="..."
   export CONFLUENCE_SPACE_KEY="..."
   export CONFLUENCE_SDD_ROOT_PAGE_ID="..."
   ```

2. **Project `.env` file** loaded by a tool like `direnv` or
   `dotenv-cli`. Useful if the user wants per-project values. Same
   keys, no `export` prefix. Remind the user to add `.env` to
   `.gitignore` if it is not already.

3. **Session-only** — `export` in the current shell, not persisted.
   Useful for a one-off run. Warn the user the variables will vanish
   when the shell exits.

In every case, show the block and have the user copy or run it. None of
these values is a secret, so there is no token line to withhold. Do not
write to the user's rc file or `.env` for them unless they explicitly
ask you to and confirm the destination path.

After the user applies the changes, remind them of two things. First, the
skills read these variables from the environment, so if Claude Code was
already running they need to restart it (or reload the environment) before
the new values take effect. Second, the Atlassian MCP authenticates over
OAuth, not these env vars: if it has not been authorised yet, they complete
the OAuth flow by running `/mcp` in an interactive session and authorising
`atlassian`.

On macOS, be aware of a launch-context gotcha. GUI-launched clients (the
Claude desktop app, and IDE integrations started from the Dock or Finder)
do not source the user's shell startup files, so values set in `~/.zshrc`,
`~/.zshenv`, or `~/.zprofile` are visible in a terminal but never reach a
GUI-launched client. The signature is a skill receiving an unexpanded
placeholder, for example a literal `${JIRA_BASE_URL}`. If the
user runs a GUI client, recommend publishing the values into the per-user
launchd environment with `launchctl setenv NAME "$NAME"` (one per variable,
run from a shell that already has them), then fully quitting and reopening
the app. For persistence across logins, a login `LaunchAgent` that reads
the shell environment and re-runs `launchctl setenv` works well; it keeps
the shell rc file as the single source of truth and stores no secret in the
agent file itself.

## Phase 4 — Verify

Once the env vars are applied and the `atlassian` MCP has been authorised
(see Authentication), run a short connectivity check. Use whichever tools
the Atlassian MCP exposes (`getVisibleJiraProjects`,
`searchJiraIssuesUsingJql`, `getConfluencePage`, or equivalents). Run these
checks in this order and stop at the first failure:

1. **Jira access.** Call a low-cost authenticated read, e.g. an
   accessible-projects or user-profile lookup. If the server is not yet
   authorised (an OAuth / "unauthorized" error, or Claude Code prompts to
   connect), tell the user to run `/mcp` in an interactive session and
   authorise `atlassian`, then re-run this check. A 401/403 after
   authorisation means the connected account lacks permission — surface the
   verbatim error and point the user at their Atlassian access, not at any
   token.
2. **Project key.** Verify `${JIRA_PROJECT_KEY}` resolves to a project
   the authenticated user can read. If it returns "not found" or "no
   permission", the key is wrong or their account lacks access. If instead
   the value arrives as an unexpanded placeholder (a literal
   `${JIRA_PROJECT_KEY}`), the skill was launched without the env var in
   scope; on macOS this is the GUI-launch case covered in Phase 3.
3. **Confluence access.** Call any authenticated Confluence read. Same
   authorisation and permission handling as for Jira.
4. **SDD root page.** Fetch `${CONFLUENCE_SDD_ROOT_PAGE_ID}` and confirm
   it exists in `${CONFLUENCE_SPACE_KEY}`. If the page is in a different
   space, flag it: pages can technically nest cross-space but most users
   set this up by mistake.

Report each check as it runs, with a clear pass / fail line. On the
first failure, stop and recommend the next action; do not continue to
later checks because they are likely to fail for the same reason.

## Phase 5 — Done

When all four checks pass, tell the user the plugin is configured and
list the skills they can now use (pull this list from the README's
Skills section rather than hardcoding it here, so this skill stays in
sync as new skills land).

## Secret handling

- This plugin needs no pasted secret. Authentication to Atlassian is over
  OAuth, handled by Claude Code, and every env var above is non-secret. Do
  not ask the user to paste an API token.
- If the user pastes something that looks like a secret anyway (an API
  token, OAuth token, or JWT), flag the paste explicitly ("that looks like
  a secret — please rotate it now"), and continue without echoing the value
  or storing it in any draft command block. Treat any shell command you
  produce as something that will be screenshared or pasted; the secret must
  not appear in those blocks.

## Hard rules

- This skill never writes to the user's shell rc or `.env` files
  without explicit, in-turn confirmation including the destination path.
- This skill never prints, repeats, or stores secret values.
- This skill never invents tenant data — examples use synthetic
  placeholders only.
- This skill is the single source of truth for which env vars exist.
  If a downstream skill needs a new variable, this skill and the README
  must be updated in the same change.
