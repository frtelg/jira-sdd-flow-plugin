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
  page whose URL ends in `.../pages/2494758913/Specs`, the value is
  `2494758913`. If unset, ask the user once for the ID before proceeding.

All Jira and Confluence reads and writes go through whatever Atlassian
MCP server is wired into the host. Use whichever tools the MCP exposes
(`jira_get_issue`, `confluence_create_page`, `confluence_update_page`,
`jira_create_remote_issue_link`, or equivalents). If no Atlassian MCP
server is available, say so and stop.

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

- Title: `Intent`
- Body sections:
  1. **Why** — the motivation. Pull from the grilled Context and any
     business rationale in the ticket or parent epic.
  2. **What is changing** — concrete description of the change in
     observable terms (system behaviour, UX, data, contracts). Avoid
     implementation detail; this page answers "why and what", not "how".
  3. **What is not changing** — restate explicit out-of-scope items from
     the grilled ticket.

### 2c. Requirements page

- Title: `Requirements`
- Body: a single bulleted list. Each bullet is one requirement, phrased
  as a testable statement of need. Group with H2 sub-headings only if
  the list is long enough to need grouping (>10 items).
- Pull from the grilled Decisions and Acceptance criteria. Each
  requirement must trace back to at least one of those — if you find a
  decision that does not map to a requirement, flag it to the user
  rather than dropping it.
- Do not number requirements; the list ordering is not stable enough to
  reference by number. If stable IDs are needed, use a leading tag like
  `REQ-<short-slug>:` per bullet.

### 2d. Specs page (Gherkin)

- Title: `Specs`
- Body: one or more `Feature:` blocks in Gherkin syntax inside fenced
  code blocks (` ```gherkin `). Each `Feature:` groups related scenarios.
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
  gap. Do not silently skip.
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
