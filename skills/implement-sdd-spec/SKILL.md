---
name: implement-sdd-spec
description: >
  Take a (grilled and published) Jira ticket — either a parent ticket or
  one of its subtasks — fetch its Confluence SDD set (Intent, Requirements,
  Specs), implement the change in the host codebase, and write exhaustive
  tests so every Gherkin scenario in scope is covered by at least one
  passing test. When invoked with a subtask, walks up to the parent for
  the SDD set and only implements the `@SCN-NNN` scenarios listed under
  the subtask's `## Covers scenarios` heading. Discovers the project's
  language, test framework, and conventions from its manifests; never
  introduces a new framework on its own. Use when the user says
  "implement this ticket", "build out the spec for X", invokes
  /implement-sdd-spec with a Jira issue key (parent or subtask), or
  otherwise asks to turn a published SDD into code.
---

# Implement SDD Spec

Convert a Jira ticket whose SDD has been published to Confluence into code
plus tests, with one passing test per Gherkin scenario in scope.

The ticket may be either a **parent** (implement every scenario on its
Specs page) or a **subtask** (implement only the `@SCN-NNN` scenarios
the subtask claims under its `## Covers scenarios` heading; the SDD set
lives on the parent).

This skill assumes upstream work has already happened: the ticket has been
grilled (see `grill-jira-ticket`), the four-page Confluence SDD has been
published (see `publish-sdd-to-confluence`), and — for subtasks —
`create-subtasks` has assigned a scenario claim. It will still run when
those artefacts are missing, but it will refuse to invent specs; instead
it tells the user what is missing and stops.

## Argument Handling

The skill expects a Jira issue key matching `^[A-Z][A-Z0-9_]+-\d+$` (for
example `PROJ-123`).

If no argument is given, ask the user once for the issue key. Do not
proceed without one.

## Configuration

Reads:

- `${JIRA_BASE_URL}` — Jira instance URL (for rendering issue references).
- `${CONFLUENCE_BASE_URL}` — Confluence instance URL (for resolving and
  rendering page links).
- `${CONFLUENCE_SPACE_KEY}` — Confluence space the SDD pages live in.

All Jira and Confluence reads go through whatever Atlassian MCP server is
wired into the host. Use whichever tools the MCP exposes (`jira_get_issue`,
`confluence_get_page`, `confluence_get_page_children`, `confluence_search`,
`jira_add_comment`, or equivalents). If no Atlassian MCP server is
available, say so and stop.

## Phase 1 — Load context

1. **Fetch the Jira ticket.** Capture: summary, description, status, issue
   type, acceptance criteria field, labels, parent/epic, comments, and
   remote issue links.
2. **Detect whether this is a parent or a subtask.** Treat as a subtask
   if either:
   - the issue type is the project's subtask type (`Sub-task` in standard
     Jira schemas), or
   - the ticket has a parent link pointing at another issue **and** that
     parent has the SDD landing page (the canonical pointer).
   Otherwise, treat as a parent.

   If subtask, set `SDD_KEY` to the parent's key for the SDD lookup, and
   keep `WORK_KEY` as the subtask's own key for status comments and the
   final report. If parent, both are the same.
3. **Locate the Confluence landing page** for `SDD_KEY`. In priority
   order:
   1. A remote issue link on `SDD_KEY` whose title starts with `SDD:` and
      whose URL is under `${CONFLUENCE_BASE_URL}`. This is the canonical
      pointer written by `publish-sdd-to-confluence`.
   2. A Confluence search in `${CONFLUENCE_SPACE_KEY}` for a page titled
      `[KEY]: ...` where `KEY` is `SDD_KEY`.
4. **Fetch the SDD set.** From the landing page ID, list its child pages
   and fetch the three titled `[SDD_KEY] Intent`, `[SDD_KEY] Requirements`,
   and `[SDD_KEY] Specs` — the `[KEY]` prefix is mandated by
   `publish-sdd-to-confluence` to keep titles unique within the space.
   Keep the markdown of each. If any of the three children is missing
   under the expected title, treat it as a missing SDD page (see step 5).
5. **Decide whether to continue.** Refuse and stop, with a clear message,
   in any of these cases:
   - No landing page is found on `SDD_KEY`. Recommend running
     `publish-sdd-to-confluence` on the parent first.
   - The landing page exists but the `Specs` child is missing or empty.
     Recommend regenerating the Specs page.
   - The Specs page contains no Gherkin scenarios. Recommend regenerating.
   - The Specs page has `Scenario:` blocks but no `@SCN-NNN` tags, **and**
     this invocation is on a subtask. Without stable IDs the subtask's
     scenario claim cannot be resolved; recommend re-running
     `publish-sdd-to-confluence` on the parent.
   In every refusal case, do not improvise specs and do not change any
   code.
6. **Parse the Specs page.** Extract every `Feature:` and `Scenario:` block
   from fenced ```gherkin``` code blocks. For each scenario, capture its
   `@SCN-NNN` tag if present. Note any scenario that contains a
   `# TODO:` comment — those are explicit gaps the spec author left.
7. **Determine the in-scope scenario set.**
   - For a **parent** invocation: every scenario on the Specs page is in
     scope.
   - For a **subtask** invocation: parse the subtask's description for a
     `## Covers scenarios` heading. The in-scope set is exactly the
     `@SCN-NNN` IDs listed there. Refuse and stop if any of:
     - the heading is missing (run `create-subtasks` on the parent
       first);
     - the list is empty;
     - one or more listed IDs do not appear on the parent's Specs page
       (likely retired by a re-publish — surface the drifted IDs and
       recommend re-running `create-subtasks` to reconcile).
8. **Read the Open questions section** (H2 `## Open questions`) on the
   Specs page if present. Any question marked open blocks implementation
   until the user answers or explicitly parks it.
9. **Discover the host project.** Inspect the working directory:
   - language and runtime from manifests (`package.json`, `pyproject.toml`,
     `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `composer.json`,
     `Gemfile`, etc.)
   - test framework and test command from the same manifests, CI config
     (`.github/workflows/`, `.gitlab-ci.yml`, etc.), or any `Makefile`
     / `justfile` targets
   - lint and format commands, if any
   - existing test layout and naming conventions
   Do not assume. If the project's language or test framework cannot be
   determined, ask the user once before continuing.

## Phase 2 — Plan

1. **Build a traceability matrix** in three columns: Requirement → Spec
   scenario(s) → planned test(s). Use the **in-scope scenario set** from
   Phase 1 step 7 — not necessarily every scenario on the Specs page.
   - For a **parent** invocation: every requirement must map to at least
     one in-scope scenario, and every in-scope scenario must map to at
     least one planned test.
   - For a **subtask** invocation: it is expected that some Requirements
     trace to scenarios this subtask does not claim (sibling subtasks
     will cover them). Mark those rows as "out of scope for this
     subtask" rather than flagging them as gaps. Every in-scope scenario
     still must map to at least one planned test.
2. **Flag mismatches** before writing code:
   - On a **parent** invocation: requirements without a covering
     in-scope scenario. Surface to the user and ask whether to add
     scenarios (via `publish-sdd-to-confluence` rerun) or proceed and
     call this out in the final report.
   - On a **subtask** invocation: do **not** flag requirements whose
     scenarios are out of scope; that is the design of the split.
   - Scenarios marked `# TODO:` in the Specs page **that are in scope**.
     These are not yet specified; do not invent behaviour for them. Ask
     whether to park or resolve before implementation.
   - Open questions that have not been answered. Block on these.
3. **Detect drift between Jira and Confluence.** If the ticket's
   description or acceptance criteria contradict the SDD pages on a
   material point (scope, behaviour, data shape, ownership), name both
   sides verbatim and ask the user which is canonical. Default
   recommendation: the SDD pages, because they are the result of the
   grill and the explicit spec; the ticket may have drifted since.
4. **Sketch the implementation.** A short bullet list per requirement:
   which files or modules change, what new ones are added, where the
   tests will live. Match existing project layout; do not refactor
   adjacent code or introduce new abstractions that the spec did not
   ask for.
5. **Show the plan to the user and wait for explicit approval.** No code
   change happens until the user confirms.

## Phase 3 — Implement

1. Make changes incrementally. Match the existing code style, even if you
   would write it differently from scratch. Do not reformat or "improve"
   adjacent code that the spec does not touch.
2. Write the test for a scenario before or alongside the production code
   that satisfies it, never strictly after. Use the test framework
   discovered in Phase 1; do not introduce a new one.
3. Translate Gherkin steps into idiomatic tests in the host project's
   style. The mapping is `Scenario:` → one test case; `Given/When/Then`
   → arrange/act/assert. Keep one scenario per test so coverage is
   traceable one-to-one.
4. Use synthetic placeholders in fixtures (e.g. `participant_001`,
   `PROJ-001`, `user@example.com`). Never use real tenant data.
5. Keep the diff scoped to the requirements. Every changed line should
   trace to a requirement or to a test for one. If you find unrelated
   issues, mention them in the final report rather than fixing them in
   the same change.

## Phase 4 — Verify exhaustively

1. Run the project's full test command (the one discovered in Phase 1).
2. Build a **coverage matrix**: every **in-scope** scenario → the test
   that covers it → pass / fail. Out-of-scope scenarios (subtask
   invocation) do not appear in the matrix. Every row must have a test;
   every row must be green before the skill considers itself done.
3. If an in-scope scenario has no test, write one. If a test fails, fix
   the production code; do not weaken the test to make it pass.
4. If an in-scope scenario cannot be implemented as written (because
   the codebase makes it impossible, because it contradicts another
   scenario, or because it depends on infrastructure the project does
   not have), stop, name the conflict verbatim, and ask the user how
   to proceed. Options: amend the spec via `publish-sdd-to-confluence`,
   defer the scenario with an explicit `# TODO:` note, rework the
   implementation, or (for a subtask) hand the scenario to a different
   subtask via `create-subtasks`.
5. If the project has a lint or type-check command, run it and resolve
   any error introduced by this change. Do not chase pre-existing
   warnings that are unrelated to the diff.

## Phase 5 — Report

When every in-scope scenario has a passing test:

1. Post a comment on the **`WORK_KEY`** ticket using the MCP (i.e. the
   subtask itself on a subtask invocation, the parent on a parent
   invocation). Include:
   - one line per requirement implemented (with a note when invoked on
     a subtask that other requirements are out of scope by design)
   - the coverage matrix as a compact table: `@SCN-NNN` → test → status,
     scoped to the in-scope set
   - the test command that was run and its summary line
   - a link to the branch / MR if one was created (only if the user
     asked for that; this skill does not push or open MRs on its own)
   - the line: "Implemented against SDD on {ISO-8601 date}." On a
     subtask invocation, append: "Parent SDD: {SDD_KEY}."
2. Do not post a separate comment on the parent ticket when invoked on
   a subtask. The parent's overall progress is the union of its
   subtasks' comments; duplicating noise on the parent is not the
   skill's job.
3. Do not edit the Confluence Intent, Requirements, or Specs pages. They
   are the specification, not the implementation log. The Jira comment
   is the persisted record of what was built.
4. Do not transition the ticket's status. That is operator workflow, not
   plugin scope.
5. Return a short summary to the user in chat: requirements covered,
   scenarios green, files changed. On a subtask invocation, also state
   which sibling scenarios were out of scope so the user can see the
   slice boundary at a glance.
6. **Last-subtask hint.** On a **subtask** invocation, check whether
   every sibling subtask of the same parent already has
   `resolutiondate` set (i.e. is resolved in Jira). If so, this
   subtask is the last one left and resolving it will complete the
   parent's slice. Append a hint to the chat summary:

   > "This is the last unresolved subtask of `{SDD_KEY}`. Once you
   > mark it resolved in Jira, consider running
   > `/reconcile-sdd {SDD_KEY}` to align the SDD pages with the
   > as-built tests and reconcile sibling SDDs."

   On a **parent** invocation, append a similar hint when the parent
   has no subtasks (so implementation completed in one pass):

   > "Once you mark `{WORK_KEY}` resolved in Jira, consider running
   > `/reconcile-sdd {WORK_KEY}` to align the SDD pages with the
   > as-built tests and reconcile sibling SDDs."

   The hint is a recommendation, not a status check; do not block on
   it and do not transition the ticket. If the resolution-state check
   fails (MCP error, missing field), drop the hint silently rather
   than guessing.

## Hard rules

- No code change happens without an explicit user confirmation on the
  plan in Phase 2.
- No new test framework, language runtime, or build tool is introduced.
  If the project lacks a test setup, stop and ask the user how they
  want testing wired up before continuing.
- Specs are never weakened to make tests pass. Either the code changes
  or the spec changes; never the test in isolation.
- The skill never invents requirements or scenarios that aren't on the
  Confluence Specs page. If something is missing, route the user back
  to `publish-sdd-to-confluence` (or `grill-jira-ticket` if the ticket
  was never grilled).
- On a subtask invocation, the in-scope set is exactly the `@SCN-NNN`
  IDs the subtask claims under `## Covers scenarios`. The skill never
  expands the in-scope set on its own, never silently picks up
  scenarios a sibling owns, and never trims the set without an explicit
  user instruction (which would itself route through
  `create-subtasks`).
- The skill never pushes commits, opens merge requests, or transitions
  Jira tickets unless the user explicitly asks for those actions in
  the same turn.
- If the MCP write of the final Jira comment fails, surface the error
  verbatim. Do not retry destructively; the code change is the record
  on disk and the user can rerun the comment step later.
