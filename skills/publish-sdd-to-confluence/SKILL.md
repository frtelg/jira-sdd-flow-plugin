---
name: publish-sdd-to-confluence
description: >
  Take a (grilled) Jira ticket and publish it to Confluence as a four-page
  spec set: a landing page titled "[KEY]: Change name" with a compressed
  summary and table of contents, plus three child pages — Intent (why and
  what is changing), Requirements (bullet list), and Specs (exhaustive
  Gherkin scenarios for AI or human validation). The landing page is
  created under a configurable Confluence root page. Use when the user
  asks to "publish the SDD pages", "push this ticket to Confluence as
  intent/requirements/specs", or invokes /publish-sdd-to-confluence with a
  Jira issue key.
---

# Publish SDD to Confluence

Convert a grilled Jira ticket into a four-page Confluence spec set
(landing + Intent + Requirements + Specs) under a configured root page.

This skill assumes the input ticket has already been grilled (typically by
`grill-jira-ticket`). It will still run on an ungrilled ticket, but the
output quality depends on what is already captured in the description; in
that case, recommend the user runs the grill first.

## Argument Handling

The skill expects a Jira issue key matching `^[A-Z][A-Z0-9_]+-\d+$`
(for example `PROJ-123`).

If no argument is given, ask the user once for the issue key. Do not
proceed without one — this skill never invents ticket content.

## Configuration

Reads:

- `${JIRA_BASE_URL}` — Jira instance URL (for rendering issue references).
- `${CONFLUENCE_BASE_URL}` — Confluence instance URL (for rendering page
  links back to the user).
- `${CONFLUENCE_SPACE_KEY}` — Confluence space the pages live in.
- `${CONFLUENCE_SDD_ROOT_PAGE_ID}` — numeric Confluence page ID of the
  root page under which the landing page is created. Example: for a root
  page whose URL ends in `.../pages/1234567890/Specs`, the value is
  `1234567890`.

**Assume the Atlassian MCP and the env vars above are already configured.**
Do not check them up front and do not prompt the user for missing values
before attempting an MCP call. Use whichever tools the MCP exposes
(`jira_get_issue`, `confluence_create_page`, `confluence_update_page`,
`jira_create_remote_issue_link`, or equivalents).

If an MCP call fails in a way that points at configuration — server
unavailable, 401/403, a space key or page ID that does not resolve, a
required field rejected as missing — surface the verbatim error to the
user and offer two options: run `/setup-jira-sdd-environment` to fix the
env, or set the offending variable directly themselves. Do not retry
blindly and do not invent page content.

## Phase 1 — Load and Brief

1. Fetch the Jira ticket. Capture: summary, description, issue type,
   acceptance criteria field (if any), labels, status, parent/epic link.
2. Detect whether the description looks grilled — i.e. it contains the
   sections produced by `grill-jira-ticket` (Context / Decisions /
   Acceptance criteria / Out of scope / Open questions / Grill log).
3. If the ticket does not look grilled, say so and recommend the user
   runs the grill first. Ask whether to proceed anyway. Default
   recommendation: do not proceed.
4. Decide the **change name**. Default to the Jira ticket summary,
   trimmed and Title Cased. Show it to the user and offer to edit before
   the page title is locked in. The final landing-page title must read
   exactly `[KEY]: <change name>` (square brackets included).

## Phase 2 — Draft the Four Pages

Draft all four pages in markdown before publishing anything. Show each
draft to the user and collect edits in this order: landing → Intent →
Requirements → Specs. Re-show any draft that changes downstream of an
edit (e.g. if the user reframes Intent, regenerate Requirements and
Specs to match).

### Formatting rules for Jira descriptions

All four pages this skill drafts are **Confluence** pages, and Confluence
is the clean case: author them in Markdown and create them with
`content_format: 'markdown'`. The Confluence MCP converts that Markdown
faithfully — curly braces inside code spans and nested bullet lists
survive the round-trip, so none of the Jira restrictions below apply to
the four spec pages.

The only Jira write this skill performs is the remote issue link in
Phase 3 step 6, which carries a title and a URL but no description body —
so today there is nothing here that needs Jira wiki markup. The rule
below exists for the moment this skill ever writes a Jira **description**
(its own or via reconciliation): in that case the description must use
Jira wiki markup syntax, NOT Markdown, because the Atlassian MCP converts
Markdown to wiki lossily for content with curly braces inside code spans
or nested bullet lists:

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
  treats as a list terminator. If you need sub-items, put them on the
  same bullet line with an italic marker like `_Why:_` separating the
  parts.
- Code blocks: `{code}...{code}` for multi-line snippets that
  legitimately need braces. These are block-level and disable wiki
  parsing inside.

To restate the split: Confluence writes (the four spec pages) stay in
Markdown via `content_format: 'markdown'`; the wiki rules above are
Jira-only.

### Title prefix rule (all four pages)

Confluence Cloud requires page titles to be unique within a space.
Generic titles like `Intent`, `Requirements`, or `Specs` will collide
with any pre-existing page of the same name (e.g. the SDD root page
itself is often named `Specs`).

Every page this skill creates must therefore have a title beginning
with `[KEY]`, where `KEY` is the Jira issue key:

- Landing: `[KEY]: <change name>` (the colon and change name follow
  the historical landing format).
- Intent: `[KEY] Intent`
- Requirements: `[KEY] Requirements`
- Specs: `[KEY] Specs`

These exact titles are what downstream skills (`implement-sdd-spec`,
`create-subtasks`) look up under the landing page's children. Do not
deviate from the format.

### Bidirectional linking (landing page only)

The landing page is the single Confluence anchor for the Jira ticket.
The two directions of the link are:

- **Confluence → Jira.** The landing page carries a `Source ticket:`
  header line linking to the Jira issue (see 2a).
- **Jira → Confluence.** A remote issue link is added to the Jira ticket
  in Phase 3 step 6, pointing at the landing page.

Intent, Requirements, and Specs pages do **not** link directly to the
Jira ticket. They are reached via Confluence's parent breadcrumb to the
landing page, and from there to the ticket. This keeps the ticket's
remote-link list to a single canonical entry and prevents the Jira
issue from accumulating four near-duplicate Confluence references over
re-publishes.

### 2a. Landing page

- Title: `[KEY]: <change name>`
- Header line (the landing page is the only Confluence page that
  carries this link — see the bidirectional linking rule above):
  `Source ticket: [<KEY>](${JIRA_BASE_URL}/browse/<KEY>)`
- Body sections, in order:
  1. **At a glance** — three or four sentences max. State the change in
     plain language, the user or system it affects, and the desired
     outcome. No bullet points, no jargon.
  2. **Status** — current Jira status of the ticket.
  3. **Contents** — table of contents listing the three child pages.
     Use real Confluence links once they exist (Phase 3 fills these in).
     Until then, use placeholder bullets in the draft.

### 2b. Intent page

- Title: `[KEY] Intent`
- Body sections:
  1. **Why** — the motivation. Pull from the grilled Context and any
     business rationale in the ticket or parent epic.
  2. **What is changing** — concrete description of the change in
     observable terms (system behaviour, UX, data, contracts). Avoid
     implementation detail; this page answers "why and what", not "how".
  3. **What is not changing** — restate explicit out-of-scope items from
     the grilled ticket.

### 2c. Requirements page

- Title: `[KEY] Requirements`
- Body: a single bulleted list. Each bullet is one requirement, phrased
  as a testable statement of need, led by a stable `REQ-<slug>` ID.
  Group with H2 sub-headings only if the list is long enough to need
  grouping (>10 items).
- Pull from the grilled Decisions and Acceptance criteria. Each
  requirement must trace back to at least one of those — if you find a
  decision that does not map to a requirement, flag it to the user
  rather than dropping it.
- **Every requirement must carry a stable ID tag** of the form
  `REQ-<slug>`, where `<slug>` is a short lowercase kebab-case phrase
  naming the requirement's core need (e.g. `REQ-cancel-window`,
  `REQ-refund-eligibility`). The bullet format is
  `- REQ-<slug> — <testable statement>`. These IDs are the requirement
  end of the requirement↔scenario contract: scenarios cite them with
  `@REQ-<slug>` tags on the Specs page (see 2d), which is what lets
  `reconcile-sdd` verify the full intent→requirement→scenario→test
  chain mechanically rather than re-deriving it from prose.
- Slug stability rules (mirror the `@SCN-NNN` lifecycle):
  - **First publish.** Assign each requirement a slug in declaration
    order. Keep slugs short and distinct; if two requirements would
    collide on a slug, disambiguate with a trailing number
    (`REQ-refund-1`, `REQ-refund-2`).
  - **Re-publish.** Preserve every existing `REQ-<slug>` verbatim for
    any requirement that is still present, **even if its wording
    changed**. The slug is an identity, not a summary — do not
    regenerate it from new wording. Give a genuinely new requirement a
    new slug. Do not reuse the slug of a removed requirement; it is
    retired so old `@REQ-<slug>` citations resolve to "no longer a
    requirement" rather than to an unrelated one. If the slug plan
    would touch any existing slug, surface it to the user and ask
    before publishing.

### 2d. Specs page (Gherkin)

- Title: `[KEY] Specs`
- Body: one or more `Feature:` blocks in Gherkin syntax inside fenced
  code blocks (` ```gherkin `). Each `Feature:` groups related scenarios.
- **Every `Scenario:` must carry a stable ID tag** of the form
  `@SCN-NNN` (three-digit zero-padded), placed on the line directly
  above `Scenario:` per Gherkin convention. Examples: `@SCN-001`,
  `@SCN-042`. These IDs are the contract that lets other skills
  (notably `create-subtasks` and `implement-sdd-spec`) reference a
  subset of scenarios; without them, downstream references rot.
- Numbering rules:
  - **First publish.** Number scenarios `001`, `002`, ... in
    declaration order across the entire Specs page (not per
    `Feature:`).
  - **Re-publish.** Preserve every existing `@SCN-NNN` for any
    scenario whose Given/When/Then is materially unchanged. Append
    new scenarios with the next free number. Do not reuse the ID of
    a removed scenario — that ID is retired so old subtask
    references still resolve to "no longer in spec" rather than to
    an unrelated scenario. If the renumbering plan would touch any
    existing ID, surface it to the user and ask before publishing.
- **Every `Scenario:` must cite at least one requirement** with one or
  more `@REQ-<slug>` tags drawn from the Requirements page (2c). Stack
  them on the same tag line as the scenario's `@SCN-NNN`, in the order
  the requirements appear on the Requirements page. Example:

  ```gherkin
  @SCN-012 @REQ-cancel-window @REQ-grace-period
  Scenario: Participant cancels inside the grace window
  ```

  The `@REQ-<slug>` citation is what turns "every scenario satisfies a
  stated requirement" into a mechanical check. A scenario that
  legitimately satisfies no stated requirement signals that either a
  requirement is missing or the scenario is out of scope — surface it
  to the user and either add the `@REQ-<slug>` citation (adding the
  requirement to 2c if it is genuinely missing) or confirm with the
  user before publishing a scenario with no citation. Likewise, before
  publishing, check the reverse direction: every `REQ-<slug>` on the
  Requirements page must be cited by at least one scenario (a
  requirement with no scenario is handled by the stub rule below).
- Each scenario must be:
  - **Exhaustive** — cover the happy path, the named edge cases from the
    grilled ticket, and the error paths surfaced during the grill.
  - **Concrete** — Given/When/Then steps use real values or named
    placeholders, not "some input" or "valid data". Use synthetic
    placeholders (e.g. `participant_001`, `PROJ-001`) rather than real
    tenant data.
  - **Independently runnable** — no implicit ordering between scenarios.
- If a requirement has no corresponding scenario, flag it to the user
  and add a `Scenario:` stub with a `# TODO: ...` comment naming the
  gap. The stub still gets its own `@SCN-NNN` tag so a subtask can
  claim it once the gap is closed, and it must cite the orphaned
  requirement with that requirement's `@REQ-<slug>` tag — so "every
  requirement is cited by at least one scenario" stays mechanically
  true even while the gap is open.
- If the grilled ticket has **Open questions**, list them under a final
  H2 `## Open questions` on this page rather than inside the Gherkin
  block, so they don't get mistaken for unimplemented specs.

## Phase 3 — Publish

Only run this phase after the user has explicitly approved all four
drafts. Confluence writes happen in this exact order:

1. **Create landing page** as a child of `${CONFLUENCE_SDD_ROOT_PAGE_ID}`
   in space `${CONFLUENCE_SPACE_KEY}`. The Contents section uses
   placeholder bullets at this point. Capture the returned page ID.
2. **Create Intent page** with parent = landing page ID. Capture its ID
   and the URL the MCP returns.
3. **Create Requirements page** with parent = landing page ID. Capture
   its ID and URL.
4. **Create Specs page** with parent = landing page ID. Capture its ID
   and URL.
5. **Update landing page** to replace the placeholder Contents bullets
   with real markdown links to the three child pages, in this order:
   Intent, Requirements, Specs. Render each link with the child page
   title as the link text.
6. **Add a remote issue link** on the Jira ticket pointing at the
   landing page (Jira → Confluence direction of the bidirectional
   link). Title: `SDD: <change name>`. URL: the landing page URL. This
   makes the Confluence pages discoverable from the ticket during
   implementation, complementing the `Source ticket:` header line that
   every Confluence page already carries (Confluence → Jira direction).

After each MCP write, show the user the result (page title + URL or
ticket key + link). If any write fails, surface the error verbatim and
stop — do not continue with subsequent steps and do not retry with
destructive flags. The user can re-run the skill after fixing the
underlying issue; if a partial write happened, name which pages already
exist so the user can clean up or resume.

## Phase 4 — Report

When publishing is complete, return a compact summary to the user:

- Landing page: title + URL
- Three child pages: titles + URLs
- Jira ticket: key + a note confirming the remote link was added

Do not write any state file. The Confluence pages and the Jira remote
link are the persisted record.

## Hard Rules

- No write happens without an explicit user confirmation in the same
  turn the write is requested.
- Never invent ticket content, decisions, requirements, or scenarios
  that aren't in the source ticket or that the user hasn't confirmed.
- Never delete or overwrite Confluence pages that already exist at the
  same path. If a page with the target title already exists under the
  root, surface that to the user and ask how to proceed (rename, update
  in place, or abort).
- Never hardcode space keys, page IDs, or hostnames in the draft
  content. Render references through the env vars above.
