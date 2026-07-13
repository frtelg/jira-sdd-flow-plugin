---
name: implement-lite
description: >
  Fast lane for trivial changes: implement a grilled Jira ticket directly
  from the inline Acceptance criteria in its description, skipping the
  four-page Confluence SDD set entirely. Discovers the host project's
  language and test framework, writes one passing test per acceptance
  criterion, and reports a coverage table back to the ticket. Use for
  small, single-increment changes where a full Intent/Requirements/Specs
  set would be overkill — the "easy, not complex" lane. Use when the user
  says "fast lane", "lightweight implement", "implement without specs",
  "just build this ticket", invokes /implement-lite with a Jira issue key,
  or asks to implement a grilled ticket that has no Confluence SDD. For
  anything that needs requirement→scenario traceability, sibling
  reconciliation, or a subtask split, route to the full path
  (publish-sdd-to-confluence → implement-sdd-spec) instead.
---

# Implement Lite (fast lane)

Turn a grilled Jira ticket into code plus tests **straight from its inline
Acceptance criteria**, with no Confluence pages in the loop. This is the
"easy, not complex" lane for small, single-increment changes where the
four-page SDD set (Intent / Requirements / Specs) would add ceremony
without adding clarity.

The source of truth here is the **Acceptance criteria** section that
`grill-jira-ticket` writes into the ticket description (Phase 4). Each
criterion is treated like a single in-scope behaviour and gets at least one
passing test. There are no `@SCN-NNN` scenarios, no requirement IDs, and no
Confluence reads.

This skill assumes the ticket has been grilled (see `grill-jira-ticket`) so
its Acceptance criteria are testable and mutually exclusive. It will run
without that, but if the criteria are missing or too vague to test it stops
and routes the user to grill first rather than inventing behaviour.

## When to use the full path instead

This skill is deliberately narrow. Recommend the full SDD path and stop if
any of these hold:

- A Confluence SDD landing page **already exists** for the ticket (a remote
  issue link whose title starts with `SDD:`). The full path is canonical
  once it exists; use `implement-sdd-spec` so the published Specs stay the
  source of truth. Do not duplicate that work here.
- The change clearly spans **multiple deployable increments** (several
  services, repos, or independently shippable slices). That is what
  `publish-sdd-to-confluence` + `create-subtasks` are for. The lite lane
  never creates subtasks and never splits work.
- The user needs **requirement→scenario→test traceability** or sibling-SDD
  reconciliation. Those live in the full path and `reconcile-sdd`; the lite
  lane opts out of that loop by design (there are no pages to reconcile).

When you route the user away, name which skill to run and why in one line,
then stop.

## Argument Handling

The skill expects a Jira issue key matching `^[A-Z][A-Z0-9_]+-\d+$` (for
example `PROJ-123`).

If no argument is given, ask the user once for the issue key. Do not proceed
without one.

## Configuration

Reads:

- `${JIRA_BASE_URL}` — Jira instance URL (for rendering issue references).

This lane reads nothing from Confluence, so it needs none of the Confluence
env vars.

**Assume the Atlassian MCP and the env var above are already configured.**
Do not check it up front and do not prompt the user for missing values
before attempting an MCP call. Use whichever Jira tools the MCP exposes
(`getJiraIssue`, `addCommentToJiraIssue`, or equivalents).

If an MCP call fails in a way that points at configuration — server
unavailable, 401/403, a required field rejected as missing — surface the
verbatim error to the user and offer two options: run
`/setup-jira-sdd-environment` to fix the env, or set the offending variable
directly themselves. Do not retry blindly and do not invent ticket content.

## Phase 1 — Load context

1. **Fetch the Jira ticket.** Capture: summary, description, status, issue
   type, the acceptance-criteria field or in-body section, labels, the
   remote issue links, and the comments.
2. **Check for an existing full SDD.** If the ticket carries the back-link
   written by `publish-sdd-to-confluence` — a remote issue link whose title
   starts with `SDD:`, or (when the MCP has no remote-link tool) a comment
   whose first line starts with `SDD:` and points at a Confluence landing
   page — stop and recommend `implement-sdd-spec` (see "When to use the full
   path instead"). Do not implement here.
3. **Extract the acceptance criteria.** Read the **Acceptance criteria**
   section from the grill (a custom field or an in-body heading). Number
   them `AC-1`, `AC-2`, … in document order — this numbering is the
   in-scope set and the anchor for test tags and the coverage table. Keep
   the verbatim text of each criterion.
4. **Decide whether to continue.** Refuse and stop, with a clear message,
   in any of these cases:
   - No acceptance-criteria section is found, or it is empty. Recommend
     running `grill-jira-ticket` on the ticket first so the criteria exist.
   - The criteria are present but not testable (vague, contradictory, or
     describing scope rather than observable behaviour). Name the offending
     criterion verbatim and recommend a grill pass to sharpen it. Do not
     guess at what "done" means.
   - The scope clearly needs multiple increments (see "When to use the full
     path instead"). Recommend the full path.
   In every refusal case, do not improvise behaviour and do not change code.
5. **Read any Open questions.** If the description has an **Open questions**
   section from the grill, any unanswered question blocks implementation
   until the user answers or explicitly parks it.
6. **Discover the host project.** Inspect the working directory:
   - language and runtime from manifests (`package.json`, `pyproject.toml`,
     `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `composer.json`,
     `Gemfile`, etc.)
   - test framework and test command from the same manifests, CI config
     (`.github/workflows/`, `.gitlab-ci.yml`, etc.), or any `Makefile` /
     `justfile` targets
   - lint and format commands, if any
   - existing test layout and naming conventions
   Do not assume. If the language or test framework cannot be determined,
   ask the user once before continuing.

## Phase 2 — Plan

1. **Build a coverage plan** in two columns: Acceptance criterion
   (`AC-N` + verbatim text) → planned test(s). Every numbered criterion
   must map to at least one planned test.
2. **Flag mismatches** before writing code:
   - A criterion you cannot turn into an observable test. Ask the user to
     sharpen it (grill) or park it explicitly; do not invent behaviour.
   - A criterion that contradicts the ticket's own summary/description.
     Name both sides verbatim and ask which is canonical.
   - Open questions that have not been answered. Block on these.
3. **Sketch the implementation.** A short bullet list per criterion: which
   files or modules change, what new ones are added, where the tests live.
   Match existing project layout; do not refactor adjacent code or
   introduce abstractions the ticket did not ask for.
4. **Show the plan to the user and wait for explicit approval.** No code
   change happens until the user confirms.

## Phase 3 — Implement

1. Make changes incrementally. Match the existing code style, even if you
   would write it differently from scratch. Do not reformat or "improve"
   adjacent code the ticket does not touch.
2. Write the test for a criterion before or alongside the production code
   that satisfies it, never strictly after. Use the test framework
   discovered in Phase 1; do not introduce a new one.
3. Translate each acceptance criterion into one or more idiomatic tests in
   the host project's style: arrange the precondition, exercise the change,
   assert the criterion holds.
4. **Tag every test with its criterion** using a comment on the line
   immediately above the test declaration:

   ```
   SDD-LITE: <KEY> AC-N
   ```

   `<KEY>` is the ticket's Jira key and `AC-N` is the criterion number from
   Phase 1. The `SDD-LITE:` prefix is mandatory and deliberately distinct
   from the full path's `SDD: <KEY> @SCN-NNN` tags so that `reconcile-sdd`
   (which greps the `SDD:`/`@SCN-NNN` form) never picks up lite tests, while
   the lite tests stay greppable on their own. Use the host language's
   single-line comment style:
   - `// SDD-LITE: PROJ-123 AC-1` for JS / TS / Java / Kotlin / Go / Rust /
     Swift / C / C++ / C#.
   - `# SDD-LITE: PROJ-123 AC-1` for Python / Ruby / Shell / Perl.
   - `-- SDD-LITE: PROJ-123 AC-1` for SQL / Haskell / Lua.

   If a single test exercises multiple criteria, stack tags — one per line,
   in number order.
5. Use synthetic placeholders in fixtures (e.g. `participant_001`,
   `PROJ-001`, `user@example.com`). Never use real tenant data.
6. Keep the diff scoped to the acceptance criteria. Every changed line
   should trace to a criterion or to a test for one. If you find unrelated
   issues, mention them in the final report rather than fixing them in the
   same change.

## Phase 4 — Verify

1. Run the project's full test command (the one discovered in Phase 1).
2. Build a **coverage table** by grepping the test tree for
   `SDD-LITE: <KEY> AC-N`: every numbered criterion → the tagged test(s)
   that cover it → pass / fail. Every criterion must have at least one
   tagged test, and every test in the table must be green before the skill
   considers itself done. A criterion with no tag hit is a coverage gap —
   write a tagged test for it before continuing.
3. If a test fails, fix the production code; do not weaken the test to make
   it pass. Acceptance criteria are never softened to go green — either the
   code changes or the criterion is re-grilled.
4. If a criterion cannot be implemented as written (impossible in the
   codebase, contradicts another criterion, depends on infra the project
   does not have), stop, name the conflict verbatim, and ask the user how
   to proceed: re-grill the criterion, defer it explicitly, rework the
   implementation, or promote the ticket to the full SDD path.
5. If the project has a lint or type-check command, run it and resolve any
   error introduced by this change. Do not chase pre-existing warnings
   unrelated to the diff.

## Phase 5 — Report

When every criterion has a passing test:

1. Post a comment on the ticket using the MCP. Include:
   - the numbered acceptance criteria as they were implemented (so the
     `AC-N` numbering is anchored for anyone reading later)
   - the coverage table as a compact table: `AC-N` → test → status
   - the test command that was run and its summary line
   - a link to the branch / MR if one was created (only if the user asked
     for that; this skill does not push or open MRs on its own)
   - the line: "Implemented via the lite path on {ISO-8601 date}. No
     Confluence SDD; acceptance criteria are the spec of record."
2. Return a short summary to the user in chat: criteria covered, tests
   green, files changed.
3. **Upgrade hint.** If, while implementing, the change turned out larger or
   more cross-cutting than a trivial increment, tell the user the lite path
   may have been the wrong call and that they can promote the ticket to the
   full SDD path by running `/publish-sdd-to-confluence {KEY}` and then
   `/implement-sdd-spec`. Offer it; do not do it automatically.

### Formatting rules for the Jira comment

Compose the Phase 5 comment in **pure standard Markdown**. The Atlassian
MCP converts Markdown to Jira wiki markup, which Jira renders correctly
only when the input is clean Markdown. Do not mix in wiki syntax (`h2.`,
`{{...}}`, `*bold*` where Markdown bold is meant); wiki tokens render as
literal text in the Jira UI. Use `**bold**` for bold, `` `inline code` ``
for monospace (`AC-N` tags and test paths sit fine inside backticks), `-`
bullets with 2-space indentation for nesting, a Markdown table for the
coverage table, and standard `[text](url)` links.

## Hard rules

- No code change happens without an explicit user confirmation on the plan
  in Phase 2.
- No new test framework, language runtime, or build tool is introduced. If
  the project lacks a test setup, stop and ask the user how they want
  testing wired up before continuing.
- Acceptance criteria are never weakened to make tests pass. Either the
  code changes or the criterion is re-grilled; never the test in isolation.
- The skill never invents acceptance criteria. If the criteria are missing
  or untestable, it routes the user to `grill-jira-ticket` and stops.
- The skill never reads or writes Confluence, never creates subtasks, and
  never produces an Intent / Requirements / Specs set. Those belong to the
  full path. When the change needs any of them, it routes the user to
  `publish-sdd-to-confluence` (and `create-subtasks`) and stops.
- Every test this skill writes carries an `SDD-LITE: <KEY> AC-N` tag comment
  on the line immediately above its declaration, in the host language's
  single-line comment style. The prefix is `SDD-LITE:` (not `SDD:`) so the
  full-path reconcile loop never mistakes a lite test for a scenario test.
- The skill never pushes commits, opens merge requests, or transitions Jira
  tickets unless the user explicitly asks for those actions in the same
  turn.
- If the MCP write of the final Jira comment fails, surface the error
  verbatim. Do not retry destructively; the code change is the record on
  disk and the user can rerun the comment step later.
