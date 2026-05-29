---
name: reconcile-sdd
description: >
  Run the closing reconciliation phase of the SDD workflow once a Jira
  ticket and all its subtasks are resolved. Diffs the as-built test
  cases against the parent's Confluence SDD pages, treats the tests as
  authoritative, and proposes targeted edits so the Intent /
  Requirements / Specs pages reflect what was actually built. Also
  verifies the requirement→scenario→test chain is intact by joining
  `REQ-<slug>` IDs to their `@REQ-<slug>` citations and `@SCN-NNN`
  tags, flagging uncovered requirements and uncited scenarios. Then,
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
- The SDD was published (`publish-sdd-to-confluence`), so the
  Requirements page has `REQ-<slug>` IDs and the Specs page has
  `@SCN-NNN` tags with `@REQ-<slug>` citations alongside them.
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

**Assume the Atlassian MCP and the env vars above are already configured.**
Do not check them up front and do not prompt the user for missing values
before attempting an MCP call. Use whichever tools the MCP exposes
(`jira_get_issue`, `jira_get_issue` with subtasks, `jira_add_comment`,
`confluence_get_page`, `confluence_get_page_children`,
`confluence_update_page`, or equivalents).

If an MCP call fails in a way that points at configuration — server
unavailable, 401/403, a space key or page ID that does not resolve, a
required field rejected as missing — surface the verbatim error to the
user and offer two options: run `/setup-jira-sdd-environment` to fix the
env, or set the offending variable directly themselves. Do not retry
blindly and do not invent page or ticket content.

## Formatting rules for Jira descriptions

This skill writes to two surfaces. The Confluence pages it edits (Intent /
Requirements / Specs, and sibling Specs) are the clean case: keep
authoring them in Markdown via `content_format: 'markdown'` — the
Confluence MCP converts that faithfully, so curly braces in code spans and
bullet lists survive.

The Jira writes this skill performs are **comments** (the sibling-ticket
comment in Phase 7 and the parent comment in Phase 8). The same rules that
govern Jira descriptions govern these comments, because the Atlassian MCP
converts Markdown to wiki lossily for content with curly braces inside
code spans or nested bullet lists — and these comments are full of
`@SCN-NNN`, `@REQ-<slug>`, and file-path tokens. Author the comment body
in Jira wiki markup, NOT Markdown:

- Headings: `h2.`, `h3.` (not `##` `###`).
- Bold: `*text*` (not `**text**`).
- Italic: `_text_`.
- Inline monospace: `{{token}}` — and the token must NOT contain `{` or
  `}`. There is no working escape; backslash escapes get doubled by the
  converter. Wrap tokens like `{{@SCN-001}}`, `{{@REQ-cancel-window}}`,
  and tag lines such as `{{SDD: PARENT-123 @SCN-001}}` this way.
- For path templates with placeholders: use `:placeholderName` style
  instead of curly braces, e.g.
  `/api/v1/campaigns/:campaignId/items/:itemId/open`. Italicise the
  whole path.
- For literal angle-bracket placeholders in file paths (e.g. directory
  templates): use `&lt;LSO&gt;` HTML entities.
- Bulleted lists: single level only. Do not use nested bullets — the MCP
  layer inserts blank lines between adjacent bullet lines, which Jira
  treats as a list terminator. If you need sub-items (e.g. a scenario and
  what changed), put them on the same bullet line with an italic marker
  like `_Why:_` separating the parts.
- Code blocks: `{code}...{code}` for multi-line snippets that
  legitimately need braces. These are block-level and disable wiki
  parsing inside.

So Confluence writes stay in Markdown via `content_format: 'markdown'`;
the wiki rules above apply only to the Jira comments this skill posts.

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
   From the Requirements page, extract the full set of `REQ-<slug>`
   IDs (the leading tag on each bullet).
6. **Parse the Specs page.** Extract every `@SCN-NNN` tag, the
   `@REQ-<slug>` citation tags stacked alongside it, and the Gherkin
   Given/When/Then beneath it. Note any scenario with a `# TODO:`
   comment.
7. **Discover the host project.** Same as `implement-sdd-spec`:
   language, runtime, test framework, test command, test layout. If
   the project's test setup cannot be determined, ask the user once.

## Phase 2 — Build the as-built picture

Tests are the source of truth. The canonical mapping from scenario to
test lives in the test source itself as a tag comment written by
`implement-sdd-spec` on the line directly above each test:

```
SDD: <PARENT_KEY> @SCN-NNN
```

This skill **trusts** that `implement-sdd-spec` enforced coverage at
implementation time. It does not re-verify that every scenario has a
test, and it does not run the tests. Its only job here is detecting
**content drift** between Gherkin and the test that was tagged to it.

1. **Grep the test tree for tags.** Search every test file under the
   test layout discovered in Phase 1 for lines matching
   `SDD: <PARENT_KEY> @SCN-NNN` (`<PARENT_KEY>` exactly; ignore tags
   pointing at any other ticket). For each hit, capture the file
   path, the line number, and the test block that begins on the next
   non-comment line. A test may carry multiple tags (stacked
   comments) — record each tag separately. The same scenario ID may
   appear on multiple tests (intentional parallel coverage at
   different layers) — that is fine, classification handles it.
2. **Build the scenario → tests map.** Key by `@SCN-NNN`, value is
   the list of tagged test locations. Tags pointing at scenario IDs
   that do **not** appear on the parent's Specs page are recorded as
   **orphan tags** (the scenario was retired by a re-publish, or the
   tag has a typo).
3. **Read each tagged test.** From its source, extract what the test
   actually asserts in observable terms (arrange / act / assert). Do
   not run the tests; they were green when `implement-sdd-spec`
   reported and re-running is operator workflow.
4. **Classify each scenario** on the Specs page:
   - **Aligned.** At least one tagged test exists, and the test's
     behaviour matches the Gherkin closely enough that no edit is
     needed. (If multiple tagged tests at different layers all
     match, still Aligned.)
   - **Drift.** At least one tagged test exists, the test is green
     (per the implementation comment), but the Gherkin and the
     test's assertions diverge in observable behaviour. **Tests are
     leading at reconcile-time** — the Gherkin needs an update to
     match the as-built test. If multiple tagged tests disagree
     among themselves, surface that as a flag (do not silently pick
     one); reconciliation can resolve at most a Gherkin/test
     disagreement, not a test/test one.
   - **TODO with implementation.** Scenario carries a `# TODO:` in
     the Specs page but a real tagged test covers it. The TODO
     should be removed and the Gherkin made to match the test.
   - **TODO still TODO.** Scenario carries a `# TODO:` and **no
     tagged test** anywhere in the tree (after also checking the
     legacy fallback below). Flag as unfinished work despite
     resolved status; do not auto-edit.
   - **Untagged-but-legacy.** No tagged test in the tree, but
     `implement-sdd-spec`'s coverage-matrix Jira comment from a
     pre-tag implementation names a test for this scenario. Read
     that test, classify Aligned / Drift as above, and offer to
     retrofit the source tag at the named location with user
     approval. The retrofit is a test-source edit and follows the
     hard rule "tests are leading, Gherkin gets updated, not the
     test" — only the tag comment is added, never assertion logic.
   - **Missing.** No tagged test, no legacy comment match. List in
     the report; do not auto-edit and do not try to derive coverage
     from semantic similarity. Recommend re-running
     `implement-sdd-spec` if this is a real gap.
5. **Verify the requirement↔scenario chain, then check for
   contradictions.** Two passes:
   - *Mechanical link check (the closed traceability chain).* Join the
     `REQ-<slug>` IDs from the Requirements page to the `@REQ-<slug>`
     citation tags parsed from the Specs page. Report three link
     defects:
     - **Uncovered requirement.** A `REQ-<slug>` that no scenario
       cites. The requirement→scenario link is broken, so the chain
       cannot reach a test for it. Flag it; do not auto-edit.
     - **Uncited scenario.** A `@SCN-NNN` scenario carrying no
       `@REQ-<slug>` tag. The scenario satisfies no stated
       requirement; either a requirement is missing or the scenario is
       out of scope. Flag it; do not auto-edit.
     - **Dangling citation.** A `@REQ-<slug>` tag on the Specs page
       whose slug is not on the Requirements page (retired or
       mistyped). Flag it; do not auto-edit.
     Because the link is an explicit tag rather than prose, this check
     is mechanical: it does not re-derive the mapping by reading
     requirement and scenario wording. Combined with the scenario→test
     tags from Phase 2 step 1, this closes the full
     intent→requirement→scenario→test chain instead of verifying only
     the scenario→test tail.
   - *Semantic contradiction check (as before).* A bullet on the
     Requirements page that asserts behaviour contradicting an
     Aligned/Drift scenario is itself drift. A statement in the
     Intent's "What is changing" section that contradicts the as-built
     scenarios is drift. Build a list of suspect lines; do not edit
     yet.
6. **Report orphan tags.** Any `SDD: <PARENT_KEY> @SCN-NNN` tag
   pointing at an ID that no longer exists on the Specs page is
   reported as drift. Recommended actions are: (a) re-tag to the new
   ID if the scenario was renumbered (rare; `publish-sdd-to-confluence`
   tries to preserve IDs), (b) delete the test if its scenario was
   retired, or (c) leave alone and add a `// SDD: retired` note. Do
   not auto-delete tests.

## Phase 3 — Propose current-page updates

Show the user a single reconciliation summary:

- Counts by classification (Aligned / Drift / TODO with impl /
  TODO still TODO / Untagged-but-legacy / Missing).
- Count of orphan tags found in the test tree.
- Chain integrity: counts of uncovered requirements, uncited
  scenarios, and dangling `@REQ-<slug>` citations from the Phase 2
  link check, with the offending IDs listed.
- A per-scenario table for everything not Aligned.
- A list of suspect Requirements / Intent lines.

For each non-Aligned row that **can** be auto-drafted (Drift, TODO
with implementation, Untagged-but-legacy), draft updated Gherkin that
matches the test. Preserve the existing `@SCN-NNN`. Show the diff per
scenario.

For Requirements and Intent, draft minimal targeted edits — not
rewrites. Each edit names the line that changes and the proposed
replacement.

For each **Untagged-but-legacy** row, also draft the source-tag
retrofit: the exact comment line to insert above the existing test
and its file/line location. The retrofit is a test-source edit; show
it as a diff and require user approval per occurrence in Phase 4.

Collect the user's per-page decisions: approve, edit, or skip. Do not
proceed to writes until the user has acted on every flagged item.

For **TODO still TODO** and **Missing** rows, do **not** draft an
edit. These are gaps `implement-sdd-spec` is responsible for, not
this skill. Recommended options for the user are:

- Re-open the relevant Jira issue and re-run `/implement-sdd-spec`
  to complete the work.
- Re-run `publish-sdd-to-confluence` to drop or rewrite the
  scenario if it is no longer wanted.
- Park explicitly with a comment if the gap is intentional.

For **orphan tags**, do not draft an auto-fix. Surface the list and
let the user decide per tag in Phase 4 whether to re-tag, delete the
test, or annotate it as retired.

For **chain link defects** (uncovered requirements, uncited scenarios,
dangling `@REQ-<slug>` citations), do not draft an auto-fix. The
requirement↔scenario link is authored by `publish-sdd-to-confluence`;
surface each defect and recommend re-running it to add the missing
`@REQ-<slug>` citation, add or retire the requirement, or correct the
slug. Park explicitly with a comment if the gap is intentional.

## Phase 4 — Apply current-page updates (and tag retrofits)

For each approved Confluence page:

1. Apply the edits in a single update per page (one `update_page`
   call per page, not per scenario).
2. Show the user the updated page URL.

For each approved **tag retrofit** on a legacy test:

3. Insert exactly the comment line drafted in Phase 3 directly above
   the existing test declaration. Touch no other line, no
   surrounding indentation, and no test logic. The retrofit is the
   only test-source edit this skill is permitted to make.
4. Show the user the file path and the diff (one inserted line).

If any update fails, surface the error verbatim and stop. Do not
retry destructively. The skill never deletes pages, never reverts
prior content beyond the targeted edits the user approved, and never
modifies test assertions or production code under any circumstances.

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
   - Counts by classification from Phase 2 (Aligned / Drift / TODO
     with impl / TODO still TODO / Untagged-but-legacy / Missing) plus
     orphan-tag count.
   - Chain integrity counts: uncovered requirements, uncited
     scenarios, and dangling `@REQ-<slug>` citations.
   - Pages updated on the current ticket (titles + URLs).
   - Tag retrofits applied to legacy tests (file paths only).
   - Siblings updated (ticket keys + page URLs) and siblings the user
     declined or skipped.
   - Gaps that were not fixed: TODO-still-TODO scenarios, Missing
     scenarios, orphan tags, and chain link defects (uncovered
     requirements, uncited scenarios, dangling `@REQ-<slug>`
     citations). List each explicitly so the user can follow up.
   - The line: "SDD reconciled on {ISO-8601 date}. Tests are the
     source of truth at reconcile-time."
2. Return a short chat summary: pages updated, tags retrofitted,
   siblings updated, gaps outstanding.
3. If nothing drifted — every scenario Aligned, the requirement↔scenario
   chain intact (no uncovered requirements, uncited scenarios, or
   dangling citations), no orphan tags, no sibling changes, no
   retrofits — the report is a clean no-op: "All aligned, no
   reconciliation needed." Do not post a parent comment in that case;
   there is nothing to record.

## Hard rules

- No write happens without explicit per-page user approval in the same
  turn as the write request. This applies to the current ticket's
  pages and to every sibling page.
- Tests are the source of truth at reconcile-time. When Gherkin and
  a passing test disagree, the Gherkin gets updated, not the test.
  The skill never edits test assertions, production code, or
  configuration. The **only** test-source edit permitted is inserting
  a `SDD: <PARENT_KEY> @SCN-NNN` tag comment above a legacy test
  during the Phase 4 retrofit, with explicit per-occurrence user
  approval. No other lines may be touched, not even reformatting or
  comment cleanup.
- The skill never runs tests, never pushes commits, never opens MRs,
  and never transitions Jira ticket status. The skill also does not
  re-verify test coverage — `implement-sdd-spec` is responsible for
  ensuring every in-scope scenario has a tagged test at implementation
  time. If reconciliation finds a real gap (TODO-still-TODO, Missing,
  or an orphan tag the user does not resolve), the skill reports it
  and stops — it does not auto-fix and does not back-fill coverage.
- The skill never writes to a sibling's Confluence page without also
  posting an explanatory Jira comment on the sibling's source ticket.
  Never the other way round either.
- The skill never invents `@SCN-NNN` IDs and never changes the IDs of
  scenarios it edits. ID stability is the contract that lets future
  reconciliations resolve.
- The skill never invents `REQ-<slug>` IDs, never edits the
  requirement↔scenario link, and never adds or removes `@REQ-<slug>`
  citations. That link is authored by `publish-sdd-to-confluence`; this
  skill only verifies it mechanically and reports defects (uncovered
  requirements, uncited scenarios, dangling citations) for the user to
  resolve by re-running publish.
- The skill never touches the **grill log** section of any Jira
  description. The grill record is historical and not subject to
  as-built drift.
- The skill never modifies tickets or pages outside the scope: the
  parent's SDD pages plus user-confirmed sibling pages plus the parent
  Jira ticket plus user-approved sibling Jira tickets. No other
  writes.
- Never hardcode hostnames, project keys, or space keys in any output.
  All such values come from the env vars listed above. If an MCP call
  fails because `${CONFLUENCE_SDD_ROOT_PAGE_ID}` (or any other env var)
  is missing or wrong, route the user to `/setup-jira-sdd-environment`
  per the Configuration block; do not silently fall back to asking.
