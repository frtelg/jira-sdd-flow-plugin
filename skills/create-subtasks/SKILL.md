---
name: create-subtasks
description: >
  Split a published SDD Jira ticket into Jira subtasks, each of which
  delivers a fully working increment for some real consumer (a deployable
  API service, a runnable CLI, a shipping UI). The split is
  judgement-based, optionally guided by a user hint such as "split into
  frontend and backend" or "one subtask per repo". Subtasks reference a
  subset of the parent's `@SCN-NNN` scenarios on its Confluence Specs
  page under a `## Covers scenarios` heading; overlap between subtasks
  is allowed, but every scenario must be claimed by at least one
  subtask. Horizontal slices that do not ship on their own are refused.
  Use when the user says "split this ticket", "create subtasks for
  PROJ-123", "subdivide this SDD", or invokes /create-subtasks with a
  Jira issue key.
---

# Create Subtasks

Divide an SDD parent ticket into Jira subtasks whose union covers every
scenario on the parent's Confluence Specs page, where each subtask is an
independently shippable working increment.

This skill assumes the parent has already been grilled (see
`grill-jira-ticket`) and that its four-page SDD has been published (see
`publish-sdd-to-confluence`). It will refuse to operate on a parent that
has no Specs page or whose Specs page has no `@SCN-NNN` scenario tags.

## Argument Handling

The skill expects a Jira issue key matching `^[A-Z][A-Z0-9_]+-\d+$` (for
example `PROJ-123`). The key must point at a parent ticket, not an
existing subtask. If the user passes a subtask, surface that and ask for
the parent key.

An optional second argument is a free-form **split hint**, e.g.
`frontend and backend`, `one subtask per repo`, `mobile, backend, admin`.
The hint is advisory: the skill still applies the "fully working
increment per subtask" rule and refuses splits that violate it.

If no issue key is given, ask the user once. Do not invent a key.

## Configuration

Reads:

- `${JIRA_BASE_URL}` — Jira instance URL (for rendering issue references).
- `${CONFLUENCE_BASE_URL}` — Confluence instance URL (for resolving and
  rendering page links).
- `${CONFLUENCE_SPACE_KEY}` — Confluence space the SDD pages live in.

**Assume the Atlassian MCP and the env vars above are already configured.**
Do not check them up front and do not prompt the user for missing values
before attempting an MCP call. Use whichever tools the MCP exposes
(`jira_get_issue`, `jira_create_issue`, `confluence_get_page`,
`confluence_get_page_children`, `confluence_search`, or equivalents).

If an MCP call fails in a way that points at configuration — server
unavailable, 401/403, a space key that does not resolve, a required field
rejected as missing — surface the verbatim error to the user and offer two
options: run `/setup-jira-sdd-environment` to fix the env, or set the
offending variable directly themselves. Do not retry blindly and do not
invent subtask content.

## Phase 1 — Load parent and SDD

1. **Fetch the parent Jira ticket.** Capture: key, summary, issue type,
   status, parent/epic, project key, remote issue links.
2. **Reject if the input is itself a subtask.** Detect by issue type
   (`Sub-task` in standard Jira schemas) or by a parent link pointing at
   another issue. Tell the user and ask for the parent's key.
3. **Locate the Confluence landing page** for the parent in priority
   order:
   1. A remote issue link on the ticket whose title starts with `SDD:`
      and whose URL is under `${CONFLUENCE_BASE_URL}`.
   2. A Confluence search in `${CONFLUENCE_SPACE_KEY}` for a page
      titled `[KEY]: ...` where `KEY` is the parent's key.
4. **Fetch the SDD set.** From the landing page, list its child pages
   and fetch the three titled `[PARENT-KEY] Intent`,
   `[PARENT-KEY] Requirements`, and `[PARENT-KEY] Specs` — the `[KEY]`
   prefix is mandated by `publish-sdd-to-confluence` to keep titles
   unique within the space. Capture the Specs page URL — subtask
   descriptions link to it.
5. **Parse the Specs page.** Extract every `Scenario:` together with
   its `@SCN-NNN` tag. Build the full set of parent scenario IDs.
6. **Refuse and stop** if any of:
   - The parent has no SDD landing page. Recommend
     `publish-sdd-to-confluence` first.
   - The Specs page is missing, empty, or has no `Scenario:` blocks.
   - One or more `Scenario:` blocks lack an `@SCN-NNN` tag. Tell the
     user the tags must be added by re-running
     `publish-sdd-to-confluence`; subtask references need stable IDs
     to point at.

## Phase 1.5 — Inventory existing subtasks

A parent may already have subtasks from a previous run of this skill,
from manual creation, or from a re-publish that retired scenario IDs.
Reconcile them before proposing anything new.

1. **List the parent's existing subtasks** via the MCP. For each, fetch
   summary, status, and description.
2. **Classify each subtask:**
   - **Current.** Has a `## Covers scenarios` heading and every
     `@SCN-NNN` it lists is still present on the parent's Specs page.
     Preserve as-is. The scenarios it claims are considered already
     covered.
   - **Stale.** Has a `## Covers scenarios` heading but at least one
     claimed `@SCN-NNN` is no longer on the Specs page (retired by a
     re-publish, or never existed).
   - **Untagged.** No `## Covers scenarios` heading at all — predates
     this skill, was created manually, or otherwise does not declare a
     scenario claim.
3. **Surface the inventory** to the user as a small table: subtask key,
   status, classification, and (for Stale) which IDs are retired.
4. **Ask the user** what to do per classification, with these defaults:
   - Current → **keep as-is** (default, no question per item unless
     the user wants to override).
   - Stale → ask per subtask: (a) drop the retired IDs from its
     `## Covers scenarios` and keep the rest, (b) close the subtask
     because the slice no longer makes sense, or (c) keep as-is and
     note the drift in the final report. Default recommendation
     depends on whether any current IDs remain: if some remain,
     recommend (a); if none remain, recommend (b).
   - Untagged → ask per subtask: (a) retrofit a `## Covers scenarios`
     claim by picking IDs interactively, (b) close, or (c) keep as-is
     and treat as covering nothing. Default recommendation: ask the
     user to pick, since the skill cannot infer scope from a
     description it did not write.
5. **Apply edits** the user approved on existing subtasks before
   Phase 2:
   - For (a) on a Stale subtask, update its description to drop the
     retired IDs from the `## Covers scenarios` list. Do not touch
     any other section.
   - For (a) on an Untagged subtask, update its description to append
     a new `## Covers scenarios` section in the format described in
     Phase 4. Do not rewrite the existing description.
   - For (b), do **not** transition the subtask to a closed status —
     that is operator workflow. Instead, post a Jira comment on the
     subtask: `This slice no longer matches the parent SDD after
     re-publish on {ISO date}. Closing recommended.` and tell the
     user in the final report which subtasks need a manual close.
   - For (c), make no write.
6. **Build the already-covered set.** Union the `@SCN-NNN` claims of
   every Current and reconciled-Stale subtask plus any retrofitted
   Untagged subtasks. This set is what Phase 3 starts with — new
   subtasks only need to claim the **remaining** scenarios, though
   they may also overlap with the already-covered set if the user
   asks for parallel coverage at another layer.

## Phase 2 — Decide whether to split (or split further)

Splitting is a judgement call, not a threshold. The shape of the
question depends on what Phase 1.5 found.

**If the parent has no existing subtasks** (greenfield split):

Default to **not splitting** unless one of these is true:

- The user passed a split hint in the invocation.
- The change touches more than one deployable (e.g. a backend service
  and a separate frontend, or two services in different repos), and
  each deployable can ship independently.
- The user, when asked, says they want subtasks.

Ask the user one question with a recommendation:

- If a hint was passed, repeat it back and ask whether to use it.
- If no hint and the parent looks single-deployable, recommend **do
  not split** and ask for confirmation.
- If no hint but the parent touches multiple deployables you can name,
  surface that and ask whether to split along that seam.

**If the parent already has subtasks** (incremental split):

Splitting has effectively been chosen. Compute the **uncovered set**:
parent `@SCN-NNN` IDs that are not yet in the already-covered set from
Phase 1.5.

- If the uncovered set is empty, tell the user the parent is already
  fully decomposed and ask whether to add additional subtasks anyway
  (e.g. a new test layer that overlaps existing coverage). Default
  recommendation: **stop**, no further subtasks needed.
- If the uncovered set is non-empty, propose creating additional
  subtasks to claim those scenarios. Ask the user whether to proceed
  and whether they have a split hint for the remaining work.

**If the user declines** further splitting in either branch, exit
cleanly with a one-line summary listing the existing subtasks (if any)
and any reconciliation actions taken in Phase 1.5. Recommend
`/implement-sdd-spec <SUBTASK-KEY>` per existing subtask, or
`/implement-sdd-spec <PARENT>` if the parent has no subtasks at all.
Do not create new Jira issues.

## Phase 3 — Propose subtasks

Only run this phase after the user has approved splitting in Phase 2.

For each proposed subtask, draft:

- **Title.** Short and specific to the slice. Format:
  `<parent summary> — <slice>`. Example:
  `Subscription cancel flow — backend API`.
- **Intent (one paragraph).** What this subtask delivers in observable
  terms. Avoid implementation detail.
- **Working increment statement.** One sentence naming the real
  consumer that benefits the moment this subtask ships on its own.
  Examples of valid consumers: "the existing fulfilment service, which
  already calls this endpoint contract"; "the operator CLI used by
  support"; "end users on the web checkout". A subtask whose
  consumer is "the other subtask in this split" is not a working
  increment and must be rejected (see Hard rules).
- **Covers scenarios.** Bullet list of `@SCN-NNN` IDs from the parent's
  Specs page that this subtask is responsible for. Overlap with other
  subtasks is allowed when the same user-visible behaviour is
  exercised at multiple layers (e.g. a backend contract test and a
  frontend end-to-end test both cover the same scenario). In the written
  subtask description each bullet uses the Jira wiki format
  `* @SCN-NNN — <scenario title>` (see Formatting rules for Jira
  descriptions in Phase 4).
- **Target.** Where this subtask lands: repo name, service name, or
  project key. Optional but recommended when multiple targets exist.

Then run the **coverage check** across the full subtask set —
existing subtasks preserved in Phase 1.5 **plus** the new drafts:

- Every `@SCN-NNN` on the parent's Specs page must appear in at least
  one subtask's Covers scenarios list. List any gaps explicitly.
- Overlap is reported but not blocking.
- Existing subtasks are included in the table as read-only rows so
  the user can see the full picture; they are not re-created in
  Phase 4.

Show all drafts and the coverage check to the user. Collect edits and
re-show until the user explicitly approves. Do not create any Jira
issues before that approval.

## Phase 4 — Create subtasks

For each approved subtask, in order:

1. Create a Jira subtask under the parent. Use the host project's
   subtask issue type as exposed by the MCP. Set:
   - **Parent:** the parent ticket key.
   - **Project:** same project as the parent unless the user
     specified a different target project in the draft.
   - **Summary:** the title from Phase 3.
   - **Description:** the structured body below, in Jira wiki markup
     (see Formatting rules for Jira descriptions below — these are Jira
     descriptions, so they must not be authored as Markdown).
2. The subtask description has exactly these sections, in this order:

   ```
   h2. Intent

   <one-paragraph intent>

   h2. Working increment

   <one-sentence consumer statement>

   h2. Covers scenarios

   See parent SDD Specs page: [<Specs page title>|${CONFLUENCE_BASE_URL}/.../Specs]

   * @SCN-NNN — <scenario title>
   * @SCN-NNN — <scenario title>
   ```

   The link to the Specs page is mandatory so `implement-sdd-spec` can
   resolve scenarios from the subtask without re-deriving them. The
   `h2. Covers scenarios` heading reads back from the Atlassian MCP as
   `## Covers scenarios`, which is the literal string the inventory step
   (Phase 1.5) and `implement-sdd-spec` look for — do not rename it.
3. Capture the returned subtask key and URL.

If any create fails, surface the error verbatim and stop. Do not retry
destructively, and do not delete any subtasks that were already
created — the user can re-run the skill on the parent to fill in the
rest after fixing the underlying issue.

### Formatting rules for Jira descriptions

Subtask descriptions written by this skill must use Jira wiki markup
syntax, NOT Markdown. The Atlassian MCP server converts Markdown to wiki,
but the conversion is lossy for content that contains curly braces inside
code spans or nested bullet lists. To keep descriptions readable after
round-trip:

- Headings: `h2.`, `h3.` (not `##` `###`).
- Bold: `*text*` (not `**text**`).
- Italic: `_text_`.
- Inline monospace: `{{token}}` — and the token must NOT contain `{` or
  `}`. There is no working escape; backslash escapes get doubled by the
  converter.
- For path templates with placeholders: use `:placeholderName` style
  instead of curly braces, e.g.
  `/api/v1/campaigns/:campaignId/items/:itemId/open`. Italicise the
  whole path.
- For literal angle-bracket placeholders in file paths (e.g. directory
  templates): use `&lt;LSO&gt;` HTML entities.
- Bulleted lists: single level only. Do not use nested bullets — the MCP
  layer inserts blank lines between adjacent bullet lines, which Jira
  treats as a list terminator. The `## Covers scenarios` list is a flat
  list of `* @SCN-NNN` bullets; if you need to add context to a bullet,
  put it on the same line with an italic marker like `_Why:_` rather than
  a sub-bullet.
- Code blocks: `{code}...{code}` for multi-line snippets that
  legitimately need braces. These are block-level and disable wiki
  parsing inside.

Confluence writes are NOT affected by this — the Confluence MCP accepts
markdown directly via `content_format: 'markdown'` and the conversion is
clean. The rule above is Jira-only. This skill does not write Confluence
pages; the rule governs the subtask descriptions it creates in Jira.

## Phase 5 — Report

When all subtasks have been created, return a compact summary:

- Parent: key + landing page URL.
- Existing subtasks reconciled in Phase 1.5: key, classification
  (Current / Stale / Untagged), and the action taken (kept, IDs
  trimmed, claim retrofitted, close-recommendation comment posted,
  kept-as-is).
- Subtasks recommended for manual closure: list keys explicitly so
  the user can act in Jira; this skill never transitions status.
- New subtasks created in Phase 4: key + title + URL + the list of
  `@SCN-NNN` claimed.
- A coverage table covering existing **and** new subtasks:
  `@SCN-NNN → [subtask keys]` so the user can see overlap and gaps
  at a glance.
- The line: "Run `/implement-sdd-spec <SUBTASK-KEY>` to build each
  slice."

Do not modify the parent ticket's description, the Confluence pages,
or any sibling ticket beyond the targeted edits and close-recommendation
comments described in Phase 1.5. The new subtasks plus those
reconciliation edits are the only writes this skill produces.

## Hard rules

- No subtask creation without explicit user approval on every draft in
  Phase 3.
- Every subtask must name a **real, currently-existing consumer** in
  its Working increment statement. A subtask whose consumer is another
  subtask in the same split is a horizontal slice and must be rejected.
  Examples of rejected splits: "backend half of feature X" paired with
  "frontend half of feature X" where the backend has no other consumer
  on day one; "data model" subtask with no caller until a later
  subtask lands; "tests only" subtask. Examples of accepted splits:
  backend API service consumed by an existing client today plus a new
  frontend that consumes it later; one subtask per independently
  deployable repo.
- Every `@SCN-NNN` on the parent's Specs page must be claimed by at
  least one subtask before Phase 4 starts. Overlap is allowed.
- Never invent scenario IDs. Subtasks may only claim IDs that appear
  literally on the parent's Specs page.
- Never modify the parent ticket's description or the parent's
  Confluence pages. Edits to **existing subtasks** are limited to the
  targeted reconciliation actions described in Phase 1.5 (trimming
  retired IDs from `## Covers scenarios`, retrofitting a missing
  `## Covers scenarios` section, or posting a close-recommendation
  comment) and only when the user explicitly approved that action for
  that subtask.
- Never transition any ticket's status, including subtasks that the
  skill recommends for closure. The recommendation is a comment plus a
  line in the final report; the actual close is operator workflow.
- If the parent has no published SDD, or its Specs page lacks
  `@SCN-NNN` tags, refuse and route the user back to
  `publish-sdd-to-confluence`.
- Never hardcode hostnames, space keys, or project keys in subtask
  content. All such values come from the env vars listed above.
