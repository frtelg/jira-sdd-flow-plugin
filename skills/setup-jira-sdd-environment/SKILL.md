---
name: setup-jira-sdd-environment
description: >
  Walk the user through configuring the environment variables this plugin
  needs (Jira and Confluence endpoints, credentials, project key, Confluence
  space, SDD root page) and verify the Atlassian MCP can actually reach them.
  Detects which variables are already set, prompts only for the missing ones,
  produces a copy-pasteable snippet for the user's shell or `.env`, and runs
  a connectivity check at the end. Use on first install of the plugin, when
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

## Variables this plugin reads

Authoritative list (mirror of the README config table):

| Variable                       | Required by                        | Notes                                                                                                  |
|--------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------|
| `JIRA_BASE_URL`                | All Jira skills + Atlassian MCP    | Full URL with scheme, no trailing slash. Cloud example: `https://example.atlassian.net`.                |
| `JIRA_USER_EMAIL`              | All Jira skills + Atlassian MCP    | The account email used to mint the API token.                                                          |
| `JIRA_API_TOKEN`               | All Jira skills + Atlassian MCP    | **Secret.** Never echo; never write into chat history. See "Secret handling" below.                    |
| `JIRA_PROJECT_KEY`             | Ticket-creating skills             | Project prefix, e.g. the part before the dash in an issue key.                                         |
| `CONFLUENCE_BASE_URL`          | All Confluence skills              | For Atlassian Cloud this is usually `${JIRA_BASE_URL}/wiki`.                                           |
| `CONFLUENCE_SPACE_KEY`         | Doc-writing skills                 | Space key, found in space settings.                                                                    |
| `CONFLUENCE_SDD_ROOT_PAGE_ID`  | `publish-sdd-to-confluence`        | Numeric page ID. Extract from a Confluence page URL: `.../pages/<ID>/<slug>` → `<ID>`.                |

If this skill is updated to add or remove a variable, the README config
table must be updated in the same change.

## Phase 1 — Detect

1. For each variable above, check whether it is currently set in the
   environment. Use a single shell call like `printenv` or
   `env | grep -E '^(JIRA_|CONFLUENCE_)'`.
2. Render a status table to the user. Two columns: variable, status.
   Status is one of `set`, `missing`, or `empty`. For `JIRA_API_TOKEN`,
   never print the value — just `set` or `missing`. For all other
   variables it is fine to print the current value so the user can
   sanity-check it.
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
   `PROJ`, `DOCS`, `2494758913`). Never reference a real tenant.
2. Show the recommended format and an example.
3. Ask for the value, unless the variable is the API token (see Secret
   handling below).
4. If the user pastes a value that looks wrong by the heuristics in
   Phase 1, point that out and ask them to confirm or correct before
   moving on.

Per-variable guidance:

- **`JIRA_BASE_URL`** — the URL you use to access Jira in a browser, up to
  and including the host. For Atlassian Cloud, it looks like
  `https://<tenant>.atlassian.net`.
- **`JIRA_USER_EMAIL`** — the email address tied to the Atlassian account
  whose API token you will use.
- **`JIRA_API_TOKEN`** — created in your Atlassian account settings, under
  Security → API tokens. Do not paste it into chat. See Secret handling.
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
pick a destination silently. Offer four options and recommend based on
context:

1. **Shell rc file** (`~/.zshrc`, `~/.bashrc`, `~/.config/fish/config.fish`,
   etc.). Best for a single user on a single machine. Produce a block
   like:

   ```bash
   export JIRA_BASE_URL="..."
   export JIRA_USER_EMAIL="..."
   # JIRA_API_TOKEN: set this separately so it does not appear in chat
   export JIRA_PROJECT_KEY="..."
   export CONFLUENCE_BASE_URL="..."
   export CONFLUENCE_SPACE_KEY="..."
   export CONFLUENCE_SDD_ROOT_PAGE_ID="..."
   ```

2. **Project `.env` file** loaded by a tool like `direnv` or
   `dotenv-cli`. Useful if the user wants per-project values. Same
   keys, no `export` prefix. Remind the user to add `.env` to
   `.gitignore` if it is not already.

3. **A secrets manager** (1Password CLI, macOS Keychain, AWS SSO, etc.)
   surfaced into the shell via the manager's own command. Recommend this
   when the user mentions one. Do not invent the exact command — ask the
   user which manager they use and tailor accordingly.

4. **Session-only** — `export` in the current shell, not persisted.
   Useful for a one-off run. Warn the user the variables will vanish
   when the shell exits.

In every case, show the block, do **not** include the API token value
in it (always represent the token line as a comment instructing the
user to set it separately), and have the user copy or run it. Do not
write to the user's rc file or `.env` for them unless they explicitly
ask you to and confirm the destination path.

After the user applies the changes, remind them that the Atlassian MCP
reads these env vars at server startup. If the MCP is already running,
they need to restart Claude Code (or reload the MCP server) before the
new values take effect.

## Phase 4 — Verify

Once the user confirms the env vars are applied and the MCP has been
restarted, run a short connectivity check. Use whichever tools the
Atlassian MCP exposes (`jira_get_all_projects`, `jira_search`,
`confluence_get_page`, or equivalents). Run these checks in this order
and stop at the first failure:

1. **Jira auth.** Call a low-cost authenticated endpoint, e.g. a
   user-profile or accessible-projects lookup. If it fails with 401 or
   403, the URL, email, or token is wrong — surface the verbatim error
   and recommend regenerating the token.
2. **Project key.** Verify `${JIRA_PROJECT_KEY}` resolves to a project
   the authenticated user can read. If it returns "not found" or "no
   permission", tell the user the key is wrong or their account lacks
   access.
3. **Confluence auth.** Call any authenticated Confluence read. Same
   401/403 handling as for Jira.
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

- Never ask the user to paste `JIRA_API_TOKEN` into chat. Always tell
  them to set it in their shell or secrets manager out-of-band.
- If the user pastes it anyway, flag the paste explicitly ("that looks
  like an API token — please rotate it now"), and continue the
  conversation without echoing the value or storing it in any draft
  command block. Treat any shell command you produce as something that
  will be screenshared or pasted; the token must not appear in those
  blocks.
- Apply the same rule to any other apparent secret the user pastes
  (Atlassian OAuth tokens, JWTs, etc.).

## Hard rules

- This skill never writes to the user's shell rc or `.env` files
  without explicit, in-turn confirmation including the destination path.
- This skill never prints, repeats, or stores secret values.
- This skill never invents tenant data — examples use synthetic
  placeholders only.
- This skill is the single source of truth for which env vars exist.
  If a downstream skill needs a new variable, this skill and the README
  must be updated in the same change.
