---
name: grill-jira-ticket
description: >
  Interview the user about a Jira ticket (or free-form prompt) until the plan
  is fully understood, then write the resolved decisions back to Jira.
  For an existing ticket, updates its description and reconciles parent and
  sibling tickets when inconsistencies were resolved. For a free-form prompt,
  creates a new Jira ticket from the grill outcome, asking the user for issue
  type and (optionally) sprint before creating. Use when the user says
  "grill this ticket", invokes /grill-jira-ticket, passes a Jira issue key,
  or asks to grill an idea that should land as a new ticket. Credit: based on
  Matt Pocock's grill-me skill
  (https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md).
---

# Grill Jira Ticket

Interrogate a Jira ticket (or a free-form plan) until the open decisions,
dependencies, tradeoffs, and failure modes are explicit and shared. Pull
answers from the ticket, its parent, and its siblings before asking the user.
Write the final understanding back to the ticket and reconcile any drift
between related tickets.

This skill is a Jira-aware extension of the grill-me pattern by
[Matt Pocock](https://github.com/mattpocock/skills/blob/main/skills/productivity/grill-me/SKILL.md).
The underlying grill loop is duplicated here so this plugin has no runtime
dependency on Matt's skill being installed.

## Argument Handling

The skill accepts a single argument:

- A **Jira issue key** matching `^[A-Z][A-Z0-9_]+-\d+$` (for example
  `PROJ-123`). Treat as the ticket under grill.
- Anything else: treat as a **free-form prompt** describing the plan to grill.
  No Jira issue is loaded. The grill follows the same loop, and at completion
  a new Jira ticket is created from the resolved outcome (see Phase 4b).

If both forms appear (e.g. `PROJ-123 add idempotency`), take the issue key as
the target ticket and the remainder as additional framing the user wants you
to consider.

If no argument is given, ask the user once: "Jira issue key, or a plan you
want grilled?" Then proceed.

## Configuration

This skill reads:

- `${JIRA_BASE_URL}` — Jira instance URL (used for any issue links rendered
  back to the user).
- `${JIRA_PROJECT_KEY}` — optional default project key, used only when the
  user mentions a number without a project prefix.

All Jira reads and writes go through whatever Atlassian MCP server is wired
into the host. Do not hardcode tool names that are specific to one server;
use whichever Jira tools the MCP exposes (`jira_get_issue`, `jira_search`,
`jira_update_issue`, `jira_add_comment`, or equivalents). If no Atlassian
MCP server is available, say so and stop — do not invent ticket content.

## Phase 1 — Load Context (only when a ticket key is given)

Before asking the user anything:

1. Fetch the target issue. Capture: summary, description, status, issue type,
   acceptance criteria (custom field or in-body), labels, assignee, parent
   link, epic link, sprint, and the most recent comments.
2. Resolve the **parent** if any. For a Story, that is usually its Epic; for
   a Sub-task, its parent Story; for an Epic, the initiative it rolls up to
   (if modelled).
3. Resolve **siblings**: other issues sharing the same parent / epic. Search
   with the MCP, scoped to the parent link. Cap to the closest ~10 by recency
   and relevance to avoid noise.
4. Build an internal map:
   - **Decisions already stated** in any of these tickets.
   - **Open questions** explicitly called out.
   - **Implicit assumptions** that look load-bearing.
   - **Inconsistencies**: contradictions between the target ticket and its
     parent/siblings (scope, terminology, acceptance criteria, ownership,
     deadlines, non-functional constraints, data shape).

Report a short briefing back to the user: "Target ticket says X. Parent says
Y. Sibling Z disagrees on W." Do not start asking questions until the user
acknowledges the briefing or you've decided the inconsistency is the first
thing to grill on.

## Phase 2 — Grill Loop (the core pattern)

- Treat the plan like a decision tree, not a casual brainstorm.
- Resolve one decision at a time before moving deeper into its children.
- Ask exactly one question per assistant turn.
- For every question, include a recommended answer with brief reasoning.
- If a question can be answered by inspecting the ticket, parent, siblings,
  the codebase, configs, tests, or docs, investigate first instead of asking
  the user.
- Keep going until the important branches are resolved or intentionally parked.

### Default flow per turn

1. Restate the current understanding in your own words (one or two lines).
2. Identify the highest-leverage unresolved decision or inconsistency.
3. Check whether it can be answered from available context (ticket, parent,
   siblings, code).
4. If yes, report the finding and move on without asking the user.
5. If no, ask one focused question with your recommended answer.
6. After the user answers, update your internal map of:
   - decisions made
   - open branches
   - dependencies blocked on earlier choices
   - assumptions still needing confirmation
   - inconsistencies resolved (and how)
7. Move to the next most important unresolved branch.

### Questioning rules

- Prefer upstream decisions before downstream implementation details.
- Surface hidden coupling early: schema shape, ownership boundaries, rollout,
  migration, security, observability, testing, operational support.
- When two decisions depend on each other, choose which resolves first and
  say why.
- Narrow vague answers with the next single question.
- Prefer multiple-choice answers over open-ended prompts when the decision
  fits a small set of realistic options. Keep options mutually exclusive and
  concise. Mark the recommended option.
- If the user skips an important branch, bring it back before declaring the
  design understood.
- Do not dump a questionnaire. Hold a conversational cadence.

### Recommended question format

`Question:` <one concrete question>

`Options:` <2-4 concrete choices; mark the recommended one>

`Recommended answer:` <your best recommendation>

`Why:` <brief reason, tradeoff, or dependency>

### Inconsistency rules (Jira-specific)

When the target ticket disagrees with its parent or a sibling:

1. Name both sides verbatim ("Target ticket says X; parent says Y").
2. Pick which one is canonical. Default heuristic: parent epic wins on scope
   and outcome, sibling tickets win on implementation detail in their own
   slice, the target ticket wins on its own acceptance criteria — but always
   confirm with the user before treating any tie-break as final.
3. Note the resolution: which side changes, and what the new text should be.
   Track this; it drives Phase 4.

### Codebase-first heuristic

Before asking the user, also check:

- existing handlers, lambdas, processors, suppliers, repositories
- OpenAPI specs and generated API types
- tests that reveal intended behaviour
- docs, ADRs, architecture notes
- config, scripts, feature flags

If inspection answers the question, report the finding and move on.

## Phase 3 — Completion Criteria

Stop the grill only when:

- the major decisions and their rationale are shared and recorded
- inconsistencies with parent/siblings are resolved or explicitly deferred
- important dependencies are resolved or explicitly deferred
- remaining unknowns are small and clearly named
- both you and the user could explain the plan the same way

When you believe you are done, say so explicitly and ask the user to confirm
before writing anything back to Jira.

## Phase 4 — Write Back to Jira

Only run this phase after the user has confirmed the grill is complete.

The grill outcome is captured as the same structured description regardless
of mode. Suggested structure (adapt to the issue type):

- **Context** — one paragraph, the problem being solved.
- **Decisions** — bullet list of resolved decisions with one-line rationale
  each.
- **Acceptance criteria** — testable, mutually exclusive bullets.
- **Out of scope** — what was explicitly parked.
- **Open questions** — anything intentionally deferred, each with an owner
  or a follow-up ticket reference.
- **Grill log** — short note: "Captured from grill session on
  {ISO-8601 date}."

What happens next depends on whether a ticket key was provided.

### 4a. Existing-ticket mode (a ticket key was given)

1. Compose the new description as above.
2. Preserve any field that was already correct. Do not overwrite acceptance
   criteria stored in a custom field with criteria in the description body
   unless the user explicitly asked you to consolidate them; in that case,
   update both consistently.
3. Show the user the new description as a diff (or as the new text alongside
   the old) before calling the Jira update tool. Wait for explicit approval
   ("yes, write it" or equivalent) before invoking the update.
4. In the **Grill log** line, use "Description updated after grill on
   {ISO-8601 date}. Previous version preserved in issue history."

### 4b. Prompt mode (no ticket key was given)

Once the user confirms the grill is complete, create a new Jira ticket from
the outcome. Do not call the Jira create tool until every prompt below has
an explicit answer.

1. **Summary.** Draft a single-line summary from the grilled plan. Show it
   to the user and ask for confirmation or an edit.
2. **Project.** Default to `${JIRA_PROJECT_KEY}`. If that env var is unset,
   ask the user which Jira project key to create the ticket in.
3. **Issue type.** Ask the user which type the ticket should be. Use the
   MCP to fetch the available issue types for the chosen project rather
   than hardcoding a list; present them as the options for the question.
   Format:

   `Question:` What kind of ticket should this be?

   `Options:` <issue types returned by the MCP, e.g. Story, Task, Bug, Spike, Epic>

   `Recommended answer:` <your best fit given the grill outcome — usually
   Story for a user-facing increment, Task for an internal change, Spike
   for an investigation, Bug for a defect>

   `Why:` <one line tying the recommendation to the grilled scope>

4. **Sprint.** Ask whether the ticket should join a sprint:

   `Question:` Add this ticket to a sprint?

   `Options:` Yes / No

   `Recommended answer:` <Yes if the grill produced something the user
   intends to start now, otherwise No>

   `Why:` <one line>

   If the user answers **Yes**, fetch the active and upcoming sprints for
   the project via the MCP and present them as a follow-up multiple-choice
   question. Recommend the next sprint that has capacity if known,
   otherwise the active sprint. Do not invent sprint names; if the MCP
   returns none, say so and skip sprint assignment.

5. **Description.** Use the structured description from the top of Phase 4.
   The **Grill log** line in this mode reads "Created from grill session on
   {ISO-8601 date}."

6. **Confirm and create.** Show a single preview block containing project,
   issue type, summary, sprint (or "none"), and description. Wait for
   explicit "yes, create it" before calling the Jira create tool.

7. **After create.** Report the new issue key back to the user. If the user
   asks, render the URL as `${JIRA_BASE_URL}/browse/<KEY>`.

Do not ask about parent/epic, assignee, labels, story points, or other
fields unless the user brings them up. Keep this phase tight.

### 4c. Reconcile parent and siblings (existing-ticket mode only)

Parent and sibling tickets belong to other people's in-flight work. They
must never be rewritten wholesale, even when the grill produces a cleaner
structure for them. Edits to these tickets are strictly **in-place line
patches**: keep every byte of the existing description except the specific
line or contiguous lines that are inconsistent with the resolved
understanding.

For every inconsistency the grill resolved against a parent or sibling
ticket:

1. Identify the exact line (or smallest contiguous span of lines) in that
   ticket's description that is inconsistent.
2. Compose a **patched description** by taking the description verbatim
   and replacing only that line / span with the corrected text. Do not
   reformat, reorder, normalise whitespace, restructure sections, fix
   typos elsewhere, or "tidy" surrounding content. If the inconsistency
   spans multiple non-contiguous lines, treat each one as its own patch.
3. Show the user a diff that contains only the changed line(s). If the
   diff touches anything outside the inconsistent span, you have rewritten
   the ticket — go back to step 2.
4. **Immediately before** calling the MCP write, re-fetch the ticket's
   current description and verify two things:
   - The line(s) you are about to replace still exist verbatim in the
     latest description.
   - No other part of the description has changed in a way that would
     make the patch land in the wrong place.
   If either check fails, stop. Surface the drift to the user and ask
   whether to recompute the patch against the new description or abandon
   the reconcile. Never force the old patch onto a changed description.
5. After approval and a clean re-fetch, apply the patch via the MCP.
6. Add a comment on that ticket explaining the change. The comment must
   include:
   - what changed (a one-line summary of the edit)
   - why (link back to the target ticket: "Aligned with {ISSUE-KEY} after
     grill session on {ISO-8601 date}.")
   - any decision the user made that drove the change
7. Do not comment on the target ticket itself for these edits — the updated
   description is the record there.

If the user declines a parent/sibling edit, record the inconsistency under
**Open questions** in the target ticket's description instead, so it is not
silently lost.

### 4d. Never silently change

- No write happens without an explicit user confirmation in the same turn.
- If the MCP write fails, surface the error verbatim and stop. Do not retry
  with a destructive flag.
- Never delete fields you did not author. Edit, do not replace, when in doubt.
- Only the **target ticket** (the one being grilled, Phase 4a) may have its
  description rewritten end-to-end. Every other ticket touched by this skill
  — parents, siblings, anything reached via reconcile — is edited as an
  in-place line patch only (see 4c). A full-description write on a non-target
  ticket is treated as a bug.

## Output Style

- Be direct, probing, and systematic. Stay collaborative, not adversarial.
- Optimise for clarity and decision quality, not politeness theatre.
- Quote ticket text verbatim when surfacing inconsistencies.
- Render issue references as `${JIRA_BASE_URL}/browse/<KEY>` only when the
  user explicitly asks for a link; otherwise the bare key is enough.
