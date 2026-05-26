---
name: reconcile-sdd
description: >
  Run the closing reconciliation phase of the SDD workflow once a Jira
  ticket and all its subtasks are resolved. Diffs the as-built test
  cases against the parent's Confluence SDD pages, treats the tests as
  authoritative, and proposes targeted edits so the Intent /
  Requirements / Specs pages reflect what was actually built. Then,
  optionally, lets the user pick sibling SDD landing pages under the
  configured SDD root and reconciles those siblings' Specs against the
  newly-canonical as-built scenarios, with per-page approval and a
  comment on each sibling's source ticket. Refuses to write code, run
  tests, or transition Jira statuses. Use when the user says
  "reconcile the SDD", "close out PROJ-123", "the implementation is
  done — sync the spec", or invokes /reconcile-sdd with a Jira issue
  key (parent or subtask; if a subtask, the parent is reconciled).
---

# Reconcile SDD

The closing phase of the SDD workflow: align the Confluence pages with
the code that was actually shipped, then propagate any scenarios that
contradict sibling SDD pages in the same root space.

This skill assumes upstream work has finished:

- The ticket was grilled (`grill-jira-ticket`).
- The SDD was published (`publish-sdd-to-confluence`), so the Specs
  page has `@SCN-NNN` tags.
- Optionally split into subtasks (`create-subtasks`).
- The change was implemented (`implement-sdd-spec`), which posted a
  per-ticket coverage-matrix comment mapping `@SCN-NNN` → test path.
- Every involved Jira issue is resolved (has `resolutiondate` set).

If any of those are missing the skill refuses, names what is missing,
and stops without writing.

## Argument Handling

The skill expects a Jira issue key matching `^[A-Z][A-Z0-9_]+-\d+$` (for
example `PROJ-123`). The key may be a parent or a subtask; if a subtask,
the skill reconciles the **parent** (whose pages are the SDD set).

If no argument is given, ask the user once. Do not proceed without one.

## Configuration

Reads:

- `${JIRA_BASE_URL}` — Jira instance URL.
- `${CONFLUENCE_BASE_URL}` — Confluence instance URL.
- `${CONFLUENCE_SPACE_KEY}` — Confluence space the SDD pages live in.
- `${CONFLUENCE_SDD_ROOT_PAGE_ID}` — numeric ID of the SDD root page;
  used as the scope for cross-space sibling discovery.

All Jira and Confluence reads and writes go through whatever Atlassian
MCP server is wired into the host (`jira_get_issue`,
`jira_get_issue` with subtasks, `jira_add_comment`,
`confluence_get_page`, `confluence_get_page_children`,
`confluence_update_page`, or equivalents). If no Atlassian MCP server
is available, say so and stop.

## Phase 1 — Verify readiness

1. **Resolve the work key.** Fetch the ticket. If it is a subtask, set
   `PARENT_KEY` to the parent's key; otherwise `PARENT_KEY` is the
   ticket itself. The parent is the reconciliation target throughout.
2. **Check the parent is resolved.** Refuse and stop if the parent's
   `resolutiondate` field is empty. Tell the user the parent must be
   resolved in Jira before reconciliation runs.
3. **Check every subtask is resolved.** Fetch the parent's subtasks.
   Refuse and stop if any subtask has an empty `resolutiondate`. List
   the unresolved subtasks by key so the user can act.
4. **Locate the SDD landing page** on `PARENT_KEY` using the canonical
   sources (SDD-titled remote issue link, then a search in
   `${CONFLUENCE_SPACE_KEY}` for `[PARENT_KEY]: ...`).
5. **Fetch the SDD set.** Get `[PARENT_KEY] Intent`,
   `[PARENT_KEY] Requirements`, and `[PARENT_KEY] Specs` from under
   the landing page. Refuse if any of the three is missing or empty.
6. **Parse the Specs page.** Extract every `@SCN-NNN` tag and the
   Gherkin Given/When/Then beneath it. Note any scenario with a
   `# TODO:` comment.
7. **Discover the host project.** Same as `implement-sdd-spec`:
   language, runtime, test framework, test command, test layout. If
   the project's test setup cannot be determined, ask the user once.

## Phase 2 — Build the as-built picture

Tests are the source of truth. The Jira coverage-matrix comments posted
by `implement-sdd-spec` are the directory that maps `@SCN-NNN` to test
paths.

1. **Gather coverage comments.** Read comments on the parent plus every
   subtask. Look for `implement-sdd-spec`'s coverage-matrix block
   (`@SCN-NNN → test → status`). Build a map `@SCN-NNN → [test paths]`.
2. **Verify the mapping is complete.** Every `@SCN-NNN` on the Specs
   page must appear in the map. If any are missing, list them — they
   are gaps to flag, not auto-fixable.
3. **Read each test.** From its source, extract what the test actually
   asserts in observable terms (arrange / act / assert). Do not run
   the tests in this skill; they were green when `implement-sdd-spec`
   reported, and re-running is operator workflow.
4. **Classify each scenario:**
   - **Aligned.** Test behaviour matches the Gherkin closely enough
     that no edit is needed.
   - **Drift.** Test exists and is green, but the Gherkin and the test
     diverge in observable behaviour. The test is leading; the Gherkin
     needs an update.
   - **TODO with implementation.** Scenario carries a `# TODO:` in the
     Specs page but a real test covers it. The TODO should be removed
     and the Gherkin made to match the test.
   - **TODO still TODO.** Scenario carries a `# TODO:` and no test
     covers it. Flag as unfinished work despite resolved status; do
     not auto-edit.
   - **Missing test.** No coverage-matrix entry. Flag; do not
     auto-edit.
5. **Check Requirements and Intent for contradictions.** A bullet on
   the Requirements page that asserts behaviour contradicting an
   Aligned/Drift scenario is itself drift. A statement in the Intent's
   "What is changing" section that contradicts the as-built scenarios
   is drift. Build a list of suspect lines; do not edit yet.
6. **Find orphan tests** (optional, lower priority): tests in the
   project that look like SDD tests (filenames, naming conventions
   that match the as-built ones) but have no `@SCN-NNN` mapping. Flag
   them in the report only; this skill does not propose new
   `@SCN-NNN` IDs (that is `publish-sdd-to-confluence`'s job).

## Phase 3 — Propose current-page updates

Show the user a single reconciliation summary:

- Counts by classification (Aligned / Drift / TODO with impl / TODO
  still TODO / Missing test).
- A per-scenario table for everything not Aligned.
- A list of suspect Requirements / Intent lines.

For each non-Aligned row that **can** be auto-drafted (Drift, TODO
with implementation), draft updated Gherkin that matches the test.
Preserve the existing `@SCN-NNN`. Show the diff per scenario.

For Requirements and Intent, draft minimal targeted edits — not
rewrites. Each edit names the line that changes and the proposed
replacement.

Collect the user's per-page decisions: approve, edit, or skip. Do not
proceed to writes until the user has acted on every flagged item.

For TODO-still-TODO and Missing-test rows, do **not** draft an edit.
Tell the user these are gaps the skill will not touch; recommended
options are:

- Re-open the relevant Jira issue and complete the work.
- Re-run `publish-sdd-to-confluence` to drop or rewrite the scenario.
- Park explicitly with a comment if the gap is intentional.

## Phase 4 — Apply current-page updates

For each approved page:

1. Apply the edits in a single update per page (one `update_page`
   call per page, not per scenario).
2. Show the user the updated page URL.

If any update fails, surface the error verbatim and stop. Do not
retry destructively. The skill never deletes pages and never reverts
prior content beyond the targeted edits the user approved.

After this phase the current ticket's Specs / Requirements / Intent
pages are the **canonical as-built spec**. The next phase compares
sibling SDD pages against these.

## Phase 5 — Pick sibling SDD pages for cross-space sync

Cross-space sync is optional. Default to **offering it**, not running
it silently.

1. **Discover candidate siblings.** List Confluence pages directly
   under `${CONFLUENCE_SDD_ROOT_PAGE_ID}`. Each child landing page
   represents one SDD; capture its title (the `[KEY]: <name>` form)
   and source Jira key.
2. **Suggest likely candidates.** For each candidate sibling, fetch
   its `[KEY] Specs` child page and surface scenarios that mention
   any of the entities / nouns appearing in the as-built scenarios.
   This is a heuristic; treat hits as candidates, not matches.
3. **Show the user the candidate list** with a short reason per row
   (e.g. "mentions `Subscription`, `cancel_flow`"). Ask the user to
   pick the comparison set:
   - all suggested,
   - a subset (by ticket key),
   - explicitly skip cross-space sync.
4. If the user skips, jump to Phase 8.

## Phase 6 — Diff siblings against the as-built spec

For each picked sibling:

1. Fetch its `[KEY] Specs` page; extract scenarios with their
   `@SCN-NNN` tags.
2. For each as-built scenario from the current ticket, find sibling
   scenarios that describe the same observable behaviour (same entity,
   same action, same observable outcome). This is a semantic match,
   not a string match. Show candidates to the user with both
   scenarios side by side and ask: "is this the same scenario,
   sibling stale?"
3. Only the user's confirmed matches are eligible for sibling edits.
   Heuristic candidates the user rejects are dropped silently.
4. For each confirmed match, draft updated Gherkin on the sibling side
   that matches the as-built scenario. Preserve the sibling's own
   `@SCN-NNN` (do not rewrite IDs across tickets).

## Phase 7 — Apply sibling updates with explicit approval

For each sibling page with confirmed changes:

1. Show the user the full per-page diff in the chat.
2. Require explicit per-page approval in the same turn before writing.
3. On approval, update the sibling's Specs page.
4. **Post a Jira comment on the sibling's source ticket** explaining
   the change. Required content:
   - "SDD scenarios reconciled from {PARENT_KEY} on {ISO-8601 date}."
   - For each updated scenario: `@SCN-NNN` and a one-line summary of
     what changed.
   - "Tests for {PARENT_KEY} are the source of truth for this
     reconciliation. Re-verify if this sibling's behaviour has
     diverged independently."
5. If the user declines a sibling, do nothing on it — no write, no
   comment.

The skill never writes to a sibling's Confluence page without also
posting the explanatory Jira comment, and never the other way round.

## Phase 8 — Report and parent comment

1. Post a single comment on `PARENT_KEY` summarising:
   - Counts by classification from Phase 2.
   - Pages updated on the current ticket (titles + URLs).
   - Siblings updated (ticket keys + page URLs) and siblings the user
     declined or skipped.
   - Gaps that were not fixed: TODO-still-TODO scenarios, Missing-test
     scenarios, orphan tests. List each explicitly so the user can
     follow up.
   - The line: "SDD reconciled on {ISO-8601 date}. Tests are the
     source of truth."
2. Return a short chat summary: pages updated, siblings updated, gaps
   outstanding.
3. If nothing drifted — every scenario Aligned, no sibling changes —
   the report is a clean no-op: "All aligned, no reconciliation
   needed." Do not post a parent comment in that case; there is
   nothing to record.

## Hard rules

- No write happens without explicit per-page user approval in the same
  turn as the write request. This applies to the current ticket's
  pages and to every sibling page.
- Tests are the source of truth. When Gherkin and a passing test
  disagree, the Gherkin gets updated, not the test. The skill never
  edits source code, test code, or configuration.
- The skill never runs tests, never pushes commits, never opens MRs,
  and never transitions Jira ticket status. If reconciliation
  uncovers a real implementation gap (TODO-still-TODO or Missing
  test), the skill reports it and stops — it does not auto-fix.
- The skill never writes to a sibling's Confluence page without also
  posting an explanatory Jira comment on the sibling's source ticket.
  Never the other way round either.
- The skill never invents `@SCN-NNN` IDs and never changes the IDs of
  scenarios it edits. ID stability is the contract that lets future
  reconciliations resolve.
- The skill never touches the **grill log** section of any Jira
  description. The grill record is historical and not subject to
  as-built drift.
- The skill never modifies tickets or pages outside the scope: the
  parent's SDD pages plus user-confirmed sibling pages plus the parent
  Jira ticket plus user-approved sibling Jira tickets. No other
  writes.
- If `${CONFLUENCE_SDD_ROOT_PAGE_ID}` is unset, ask the user once. The
  skill cannot discover siblings without it.
- Never hardcode hostnames, project keys, or space keys in any output.
  All such values come from the env vars listed above.
