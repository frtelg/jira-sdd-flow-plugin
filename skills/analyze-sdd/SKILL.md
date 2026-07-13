---
name: analyze-sdd
description: >
  Read-only consistency gate over a published SDD, run before
  implementation. Takes a Jira ticket (parent or subtask), fetches its
  Confluence SDD set (Requirements, Specs), and detects incoherence that
  would make implementation ambiguous before any code is written:
  unresolved open questions, in-scope `# TODO:` scenario stubs,
  requirements with no covering scenario (orphan requirements),
  scenarios that cite no requirement (uncited scenarios), `@REQ-<slug>`
  citations pointing at no requirement (dangling citations), internal
  contradictions between scenarios or requirements, drift between the
  Jira ticket and the Confluence pages, and near-duplicate scenarios.
  Classifies every finding by severity and reports a gate verdict. It
  writes nothing — no code, no Jira comments, no Confluence edits — and
  never refuses; it reports and recommends. Use before
  /implement-sdd-spec, when the user says "analyze the spec", "check the
  SDD for consistency", "is this spec ready to build", "gate this before
  implementing", or invokes /analyze-sdd with a Jira issue key (parent or
  subtask). Full SDD path only; the lite path has no Confluence artefacts
  to analyse.
---

# Analyze SDD

The pre-implementation consistency gate. Before `implement-sdd-spec`
turns a published SDD into code, this skill reads the same Confluence
pages and reports whether they are coherent enough to build from. It is
the SDD equivalent of a static analysis pass: it finds incoherence early,
while fixing it is still a spec edit rather than a code rewrite.

This skill is **read-only**. It fetches Jira and Confluence, analyses, and
reports. It never edits a page, never comments on a ticket, never touches
code, and never transitions a status. It also never refuses to run — when
prerequisites are missing it says so and recommends the skill that fixes
them, but it produces a report either way.

The ticket may be a **parent** (analyse every scenario on its Specs page)
or a **subtask** (the SDD set lives on the parent; the in-scope set is the
`@SCN-NNN` IDs the subtask claims under its `## Covers scenarios`
heading). Scope-sensitive checks respect that boundary; spec-quality
checks (near-duplicates, dangling citations) look at the whole Specs page.

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

**Assume the Atlassian MCP and the env vars above are already configured.**
Do not check them up front and do not prompt the user for missing values
before attempting an MCP call. Use whichever read tools the MCP exposes
(`getJiraIssue`, `getConfluencePage`, `getConfluencePageDescendants`,
`searchConfluenceUsingCql`, or equivalents). This skill calls **no write tools**.

If an MCP call fails in a way that points at configuration — server
unavailable, 401/403, a space key that does not resolve, a required field
rejected as missing — surface the verbatim error to the user and offer two
options: run `/setup-jira-sdd-environment` to fix the env, or set the
offending variable directly themselves. Do not retry blindly and do not
invent ticket or page content.

## Phase 1 — Load context

1. **Fetch the Jira ticket.** Capture: summary, description, status, issue
   type, acceptance criteria field, labels, parent/epic, and remote issue
   links.
2. **Detect whether this is a parent or a subtask.** Treat as a subtask
   if either:
   - the issue type is the project's subtask type (`Sub-task` in standard
     Jira schemas), or
   - the ticket has a parent link pointing at another issue **and** that
     parent has the SDD landing page (the canonical pointer).
   Otherwise, treat as a parent.

   If subtask, set `SDD_KEY` to the parent's key for the SDD lookup and
   keep `WORK_KEY` as the subtask's own key for the report. If parent,
   both are the same.
3. **Locate the Confluence landing page** for `SDD_KEY`, in priority
   order:
   1. A remote issue link on `SDD_KEY` whose title starts with `SDD:` and
      whose URL is under `${CONFLUENCE_BASE_URL}` (the canonical pointer
      written by `publish-sdd-to-confluence`).
   2. A Confluence search in `${CONFLUENCE_SPACE_KEY}` for a page titled
      `[KEY]: ...` where `KEY` is `SDD_KEY`.
4. **Fetch the SDD set.** From the landing page ID, list its child pages
   and fetch the two needed here: `[SDD_KEY] Requirements` and
   `[SDD_KEY] Specs`. (The Intent page is not analysed mechanically; it is
   only consulted for the drift check in Phase 2.)
5. **Decide whether a meaningful analysis is possible.** This skill does
   not refuse, but some inputs leave nothing to analyse. In each case
   below, emit a single CRITICAL finding, skip the checks that depend on
   the missing artefact, and continue to the report:
   - No landing page found on `SDD_KEY` → finding: "No SDD published; run
     `/publish-sdd-to-confluence` on the parent first." Nothing further to
     analyse.
   - Landing page exists but the Specs child is missing or empty →
     finding: "Specs page missing or empty; regenerate via
     `/publish-sdd-to-confluence`."
   - Specs page has no Gherkin scenarios → finding: "No scenarios to
     analyse; regenerate the Specs page."
   - Specs page has `Scenario:` blocks but no `@SCN-NNN` tags **and** this
     is a subtask invocation → finding: "Scenarios are untagged, so the
     subtask's claim cannot be resolved; re-run
     `/publish-sdd-to-confluence` on the parent." Analyse the page at the
     parent level only.
6. **Parse the Specs page.** Extract every `Feature:` and `Scenario:`
   block from fenced ```gherkin``` code blocks. For each scenario,
   capture: its `@SCN-NNN` tag, any `@REQ-<slug>` citation tags stacked
   alongside it, the Given/When/Then body, and whether it carries a
   `# TODO:` comment.
7. **Parse the Requirements page.** Extract the full set of `REQ-<slug>`
   IDs (the leading tag on each bullet) and the requirement text beside
   each.
8. **Determine the in-scope scenario set.**
   - **Parent** invocation: every scenario on the Specs page is in scope.
   - **Subtask** invocation: parse the subtask's description for a
     `## Covers scenarios` heading; the in-scope set is exactly the
     `@SCN-NNN` IDs listed there. If the heading is missing, the list is
     empty, or a listed ID does not appear on the Specs page, record it as
     a HIGH finding (scope claim broken; route to `/create-subtasks`) and
     analyse the page at parent level for the spec-quality checks.
9. **Read the Open questions section** (`## Open questions`) on the Specs
   page, and any "Open questions" block in the ticket description left by
   the grill. Note which questions are still open.

## Phase 2 — Analyse coherence

Run every applicable check below. Each finding records: the check, the
offending IDs verbatim, whether it is in scope, a one-line explanation,
the severity, and the recommended route to fix it. **This skill proposes
no edits and applies none** — fixes are always a re-run of an upstream
skill, named in the finding.

Severity scale (mirrors a static-analysis gate):

- **CRITICAL** — the spec is ambiguous or self-contradictory; building
  from it would guess at intent. Must be resolved before implement.
- **HIGH** — a real traceability or coverage defect; intent could be
  silently dropped. Resolve before implement, or proceed only with an
  explicit, recorded decision.
- **MEDIUM** — a spec-quality smell that will not block a correct build
  but invites drift. Recommend cleanup.
- **LOW** — minor; note and move on.

### Scope-sensitive checks (respect the in-scope set)

1. **Unresolved open questions** — CRITICAL. Any question still open in
   the Specs `## Open questions` section or the ticket's grill block that
   bears on in-scope scenarios. Implementation cannot proceed without an
   answer. Route: `/grill-jira-ticket` to answer, then
   `/publish-sdd-to-confluence` to fold the answer into the spec.
2. **In-scope TODO stubs** — CRITICAL. An in-scope `@SCN-NNN` scenario
   that carries a `# TODO:` comment, i.e. its behaviour is not yet
   specified. Do not let implement invent behaviour for it. Route:
   `/publish-sdd-to-confluence` to specify it, or `/grill-jira-ticket` if
   the underlying decision was never made.
3. **Orphan requirement (uncovered)** — HIGH on a parent; not flagged on a
   subtask when the covering scenarios are out of scope. A `REQ-<slug>`
   that no scenario cites via `@REQ-<slug>`. The requirement→scenario link
   is broken, so the intent reaches no scenario and would be silently
   dropped at implement-time. On a subtask, a requirement whose only
   citing scenarios are owned by a sibling is expected, not a defect — do
   not flag it. Route: `/publish-sdd-to-confluence` to add a covering
   scenario or retire the requirement.
4. **Uncited scenario** — HIGH. An in-scope `@SCN-NNN` carrying no
   `@REQ-<slug>` tag. The scenario satisfies no stated requirement: either
   a requirement is missing or the scenario is out of scope. Route:
   `/publish-sdd-to-confluence` to add the citation or remove the
   scenario.
5. **Internal contradiction** — CRITICAL. Two in-scope scenarios, or a
   scenario and a requirement, that assert behaviour that cannot both
   hold (same trigger, incompatible outcome). Quote both sides verbatim.
   Route: `/grill-jira-ticket` to decide which is correct, then
   `/publish-sdd-to-confluence`.
6. **Jira ↔ Confluence drift** — CRITICAL when material. The ticket
   description or acceptance criteria contradict the SDD pages on a
   material point (scope, behaviour, data shape, ownership). Name both
   sides verbatim. The SDD pages are the recommended canonical source
   (they are the result of the grill), but the user must decide. Route:
   `/grill-jira-ticket` / `/publish-sdd-to-confluence` to reconcile.

### Spec-quality checks (whole Specs page, regardless of scope)

7. **Dangling citation** — HIGH. A `@REQ-<slug>` tag on the Specs page
   whose slug is not present on the Requirements page (retired or
   mistyped). The chain points at nothing. Route:
   `/publish-sdd-to-confluence` to correct the slug or restore the
   requirement.
8. **Near-duplicate scenarios** — MEDIUM. Two or more `@SCN-NNN`
   scenarios describing the same observable behaviour (same entity, same
   action, same observable outcome), differing only in wording. This is a
   semantic judgement, not a string match: surface them as candidates,
   show both side by side, and let the user decide whether they are truly
   redundant. Duplicates inflate the coverage matrix and invite the two
   copies to drift apart. Mark which copies are in scope. Route:
   `/publish-sdd-to-confluence` to merge or differentiate them.
9. **Untestable scenario** — MEDIUM. An in-scope scenario whose
   Given/When/Then has no observable outcome a test could assert (vague
   "works correctly", no concrete Then). It cannot become a passing
   tagged test as written. Route: `/publish-sdd-to-confluence` to sharpen
   the Then, or `/grill-jira-ticket` if the expected outcome was never
   decided.

For checks 3, 4, 5, 7 the link logic is mechanical: join `REQ-<slug>` IDs
from the Requirements page to `@REQ-<slug>` citation tags on the Specs
page, and `@SCN-NNN` IDs to the scenarios that carry them. Do not
re-derive the requirement→scenario mapping from prose; the tags are the
contract. Checks 5, 6, 8, 9 are semantic judgements — be conservative and
surface candidates rather than asserting certainty.

## Phase 3 — Report the gate verdict

Produce a single read-only report in chat. Write nothing to Jira or
Confluence.

1. **Header.** The ticket key (`WORK_KEY`, and `SDD_KEY` if different),
   parent/subtask, the Specs page URL, and the size of the in-scope set.
2. **Findings table**, grouped by severity (CRITICAL first), each row:
   severity · check · offending IDs · in-scope? · one-line explanation ·
   recommended route. Quote IDs and contradictory text verbatim.
3. **Gate verdict**, derived from the highest severity present:
   - **Any CRITICAL** → `BLOCKED`: "Resolve the CRITICAL findings before
     running `/implement-sdd-spec {WORK_KEY}`. Building now would guess at
     intent." List the route for each.
   - **HIGH only** → `RESOLVE RECOMMENDED`: "These are real defects;
     resolve them, or proceed with `/implement-sdd-spec {WORK_KEY}` only
     as a deliberate, recorded decision."
   - **MEDIUM / LOW only** → `ADVISORY`: "Safe to build. Consider the
     cleanups above before or after implementing."
   - **None** → `CLEAN`: "The SDD is coherent and in scope. Ready for
     `/implement-sdd-spec {WORK_KEY}`."
4. **Next step.** Always end by naming the single recommended next
   command (the top route for `BLOCKED`/`RESOLVE RECOMMENDED`, or
   `/implement-sdd-spec {WORK_KEY}` for `ADVISORY`/`CLEAN`).

The verdict is a recommendation, not an enforcement. This skill cannot and
does not stop the user from running implement; it gives them the
information to decide. `implement-sdd-spec` performs its own minimal
safety pre-flight and will itself block on the two hardest stops
(unresolved open questions, in-scope TODO stubs) if analyse was skipped.

## Hard rules

- The skill is strictly read-only. It calls no write tool — no
  `updateConfluencePage`, no `addCommentToJiraIssue`, no `editJiraIssue`,
  no `transitionJiraIssue` — and makes no change to code on disk. Its
  only output is the chat report.
- The skill never refuses. When prerequisites are missing it emits a
  CRITICAL finding naming the upstream skill that fixes it
  (`publish-sdd-to-confluence`, `create-subtasks`, or
  `grill-jira-ticket`) and still produces a verdict.
- The skill never invents requirements, scenarios, `REQ-<slug>` IDs, or
  `@SCN-NNN` IDs, and never proposes the text of a fix. Every finding
  routes the user to the upstream skill that owns the artefact; deciding
  and applying the fix is that skill's job, not this one's.
- Scope is honoured exactly as `implement-sdd-spec` honours it: on a
  subtask, the in-scope set is the `@SCN-NNN` IDs claimed under
  `## Covers scenarios`, and a requirement whose only covering scenarios
  belong to a sibling is not a defect. The skill never expands or trims
  the in-scope set on its own.
- Severity reflects impact on building, not aesthetics. Only ambiguity and
  self-contradiction are CRITICAL; traceability and coverage defects are
  HIGH; everything else is MEDIUM/LOW.
- Never hardcode hostnames, project keys, or space keys in any output. All
  such values come from the env vars above. If an MCP call fails because a
  variable is missing or wrong, route the user to
  `/setup-jira-sdd-environment` per the Configuration block; do not
  silently fall back to guessing.
