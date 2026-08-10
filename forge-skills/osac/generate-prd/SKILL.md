---
name: generate-prd
description: Autonomous PRD workflow that runs ingest, self-clarify, draft, and self-review phases end-to-end without human interaction. Uses merged PRDs as exemplar reference and leaves unresolvable gaps as structured open questions.
---
# Unattended PRD Workflow

Runs the PRD phases from `/ingest` through `/draft` without stopping for
human input. Designed for autonomous first-draft generation where the human
refines afterward via `/revise`.

This workflow reuses the individual phase skills (`skills/osac/generate-prd/steps/ingest.md`, `skills/osac/generate-prd/steps/clarify.md`,
`skills/osac/generate-prd/steps/draft.md`) but orchestrates them with self-resolution and self-review loops
instead of waiting for human input between phases.

## Input

The user provides a Jira Feature issue key, URL, or description. No prior
phases are required.

## Configuration

Before starting, determine these values from your context. Use the defaults
if not specified.

| Setting | Default | Override by stating in input |
|---------|---------|------------------------------|
| `max_revisions` | 2 | e.g. "max_revisions: 3" |
| `exemplar_path` | *(auto-discover)* | e.g. "exemplar_path: docs/enhancements" |

## Ground Rules

- **Do not stop for human input.** Run all phases to completion.
- **Do not fabricate requirements.** Every claim must trace to the Jira
  source, exemplar pattern, or codebase observation. When information is
  missing, write `[Open Question: ...]` — never invent.
- **Do not create git commits.** All artifacts remain in the working tree.
- **Do not push code or create PRs.** The calling system or human handles
  publication via `/publish`.
- **Write files only to `.artifacts/prd/{issue-key}/` and the project
  source tree.** Scratch files and notes go in the artifact directory.
- **Follow `skills/osac/generate-prd/guidelines.md`** for principles, hard limits, and quality
  standards. Where guidelines say "stop and request human guidance," write
  an escalation report (see Escalation below) and terminate.
- **Read and follow the project's `AGENTS.md`** if one exists. Project
  conventions take precedence over generic defaults.

## Quick Start

1. Read the bug report / issue key / description provided as input
2. Extract the issue key (e.g. `OSAC-1234` from a Jira URL)
3. Create the artifact directory: `mkdir -p .artifacts/prd/{issue-key}`
4. Execute the phase loop below, starting at Ingest
5. After each phase, proceed directly to the next — do not stop or wait

## Phase Loop

Run these phases in order:

| Order | Phase | Skill file | Done signal |
|-------|-------|------------|-------------|
| 1 | Ingest | `skills/osac/generate-prd/steps/ingest.md` | `01-requirements.md` written |
| 2 | Exemplar Selection | *(inline below)* | 2-3 similar merged PRDs read |
| 3 | Dimension Triage | *(inline below)* | Applicable dimensions identified |
| 4 | Self-Clarify | `skills/osac/generate-prd/steps/clarify.md` *(overridden)* | `02-clarifications.md` written |
| 5 | Draft | `skills/osac/generate-prd/steps/draft.md` *(overridden)* | `03-prd.md` written |
| 6 | Self-Review | *(inline below)* | Score computed |
| 7 | Revise | *(inline below, conditional)* | Score >= 7/10 or max rounds hit |
| 8 | Session Context | *(inline below)* | `session-context.md` written |

## How to Execute a Phase

1. Announce the phase: *"Starting Ingest (unattended mode)."*
2. Read the skill file from the table above. While executing it, apply
   these overrides:
   - "Never auto-advance" / "Stop and wait" / "re-read the controller" —
     ignore; proceed to the next phase in this table
   - "Stop and request human guidance" (escalation) — write an escalation
     report (see Escalation below) and terminate
   - "Present to user" — skip; proceed to the next phase
3. Execute the skill's steps fully, including its Output section — every
   artifact the skill specifies must be written to disk
4. Proceed to the next phase in the Phase Loop

## Phase Overrides

### Ingest

Read and follow `skills/osac/generate-prd/steps/ingest.md` with these overrides:

- **Re-invocation (Step 5a):** If a prior `01-requirements.md` exists,
  overwrite it without waiting for user confirmation. Write the diff to
  `.artifacts/prd/{issue-key}/ingest-diff.md` for reference.
- **Report (Step 6):** Skip — proceed directly to Exemplar Selection.

### Exemplar Selection

Select 2-3 merged PRDs as reference for how good PRDs look in this project.

1. **Discover exemplar sources.** Check for merged PRDs in these locations
   (in priority order):
   - `exemplar_path` if configured
   - A project-level `context/exemplars/` directory
   - An `enhancement-proposals/enhancements/` directory (OSAC convention)
   - Any docs repo referenced in the project's `AGENTS.md` or `CLAUDE.md`

   Search for files named `prd.md` or `*-prd.md`:
   ```bash
   find {exemplar_source} -name "prd.md" -o -name "*-prd.md" | head -30
   ```

2. For each candidate, read the first 40 lines (title, metadata, problem
   statement, in-scope section) to understand its service area and scope.

3. Select 2-3 PRDs most similar to the current feature based on:
   - **Domain area** (same service, component, or capability type)
   - **Feature type** (new resource, lifecycle enhancement, API addition,
     cross-cutting feature)
   - **Scope size** (similar complexity)

4. Read the selected exemplars fully. Note their patterns:
   - How they structure user stories by persona
   - How they scope In Scope vs Out of Scope
   - How they handle combined personas
   - Their line count relative to feature complexity

5. Record selected exemplars for session context.

If fewer than 2 merged PRDs are found, proceed with whatever is available.
If none are found, skip this phase and note "No exemplars available."

### Dimension Triage

Determine which cross-cutting dimensions apply to this feature. This phase
is project-specific — skip it if no dimensions file exists.

1. **Discover dimensions file.** Check these locations:
   - `.design/context/osac-dimensions.md` (OSAC convention)
   - `.design/context/dimensions.md`
   - A dimensions file referenced in the project's `AGENTS.md`

   If no dimensions file exists, skip to Self-Clarify.

2. Read the dimensions file.

3. For each dimension defined in the file, check whether the Jira feature
   plausibly touches it:
   - Does the feature description mention this dimension's domain?
   - Does the feature create, modify, or depend on resources in this area?
   - Would a reviewer ask about this dimension for this feature?

4. Apply the triage rule: if a dimension clearly doesn't apply, skip it
   silently. When genuinely unsure, mark it as
   `[Open Question: Does this feature affect {dimension}?]`.

5. Record results:
   ```text
   Applicable dimensions: {list}
   Skipped (not relevant): {list}
   Uncertain: {list} → [Open Question]
   ```

Also identify which **personas** and **services** are in scope using the
dimensions file's definitions.

### Self-Clarify

This replaces the interactive `/clarify` phase. Instead of asking the user
questions, attempt to resolve gaps from available context and document what
remains unresolvable.

1. Read `.artifacts/prd/{issue-key}/01-requirements.md`.

2. Run the gap analysis from `skills/osac/generate-prd/steps/clarify.md` Step 2 — analyze requirements
   against: Scope, Users/Personas, User capabilities, Acceptance criteria,
   Edge cases, Dependencies, Contradictions, Assumptions. Also check
   against applicable dimensions from Dimension Triage.

   **Persona ownership test:** For each of the 4 OSAC personas, before
   marking any as "Not affected," ask: does this persona currently perform
   the manual process this feature automates or replaces? If yes, they are
   a primary affected persona — write stories about what changes for them.

   **Lifecycle decomposition:** For any resource, pool, or capacity being
   introduced, enumerate all lifecycle operations a user can perform:
   create, list/view, update/configure, scale up, scale down, delete.
   Each operation that's in scope needs coverage in the PRD.

3. For each gap, attempt self-resolution in priority order:

   a. **Exemplar pattern:** Do selected exemplar PRDs address a similar
      gap? If a pattern is clear, adopt it:
      `**Self-resolved (exemplar):** {resolution} [Pattern from {exemplar}]`

   b. **Codebase context:** Can the codebase answer it? (grep, read
      existing code, check API definitions):
      `**Self-resolved (codebase):** {resolution} [Found in {file}]`

   c. **Dimension context:** Does the dimensions file provide guidance?
      `**Self-resolved (dimensions):** {resolution}`

   d. **Reasonable inference:** Can a reasonable inference be made?
      `**Self-resolved (inference):** {resolution} [Assumption: ...]`

   e. **Open question:** If unresolvable:
      `**[Open Question]:** {specific question} — {why it matters}`

4. Write `.artifacts/prd/{issue-key}/02-clarifications.md`:

   ```markdown
   # Self-Clarification Log — {issue-key}

   ## Status

   - Mode: Unattended (self-clarify)
   - Gaps analyzed: {count}
   - Self-resolved: {count}
   - Open questions: {count}

   ## Self-Resolved Gaps

   ### SR-1: {gap description}
   **Resolution:** {what was determined}
   **Source:** {exemplar / codebase / dimensions / inference}
   **Impact:** {how this shapes the PRD}

   ## Open Questions

   ### OQ-1: {specific question}
   **Why it matters:** {impact on the PRD if left unresolved}
   **Best guess:** {if a reasonable guess exists, state it}

   ## Dimension Coverage

   | Dimension | Applicable | Notes |
   |-----------|-----------|-------|
   | {dim} | Yes/No/Uncertain | {brief note} |
   ```

**Do not create locked decisions.** Locked decisions require human
confirmation. Self-resolutions are best-effort — the human may override
any of them via `/revise`.

**Prefer open questions over wrong answers.** When genuinely unsure,
`[Open Question]` is better than a fabricated resolution.

### Draft

Read and follow `skills/osac/generate-prd/steps/draft.md` with these overrides:

- **Step 1 (Locate the Template):** Use the OSAC PRD template at
  `skills/osac/generate-prd/prd-template.md`. Do NOT use the generic
  ai-workflows template. The OSAC template has exactly 6 sections:
  Problem Statement, In Scope, Out of Scope, User Stories, Assumptions,
  Dependencies. Do NOT add Goals/Non-Goals, FR-N/NFR-N requirement IDs,
  Acceptance Criteria, Risks, or Open Questions sections.
- **Step 2 (Read Source Material):** Also read the selected exemplar PRDs
  and the self-clarification log.
- **Step 4 (Write the PRD):**
  - **Author field:** Derive from the Jira ticket assignee or reporter
    name. Never leave as Open Question, TBD, or "To be determined."
  - Use self-resolved gaps as if they were clarification answers. Tag
    self-resolved items with their source marker (e.g.,
    `[Exemplar: {id}]`, `[Codebase: {file}]`).
  - For open questions with a "best guess," write the requirement using the
    best guess and tag it:
    `[Open Question: {question} — using best guess: {guess}]`
  - For open questions with no best guess, write:
    `[Open Question: {question}]` in place of the requirement content.
  - Apply applicable dimensions to ensure the draft covers relevant
    cross-cutting concerns.
  - Follow exemplar PRDs' patterns for structure, persona grouping, and
    scope sizing.
  - **Problem Statement:** Pain only — no solutions. Do NOT describe what
    the feature introduces or how it works. That belongs in In Scope.
  - **In Scope — status visibility:** For every resource or operation
    created asynchronously, verify status tracking is addressed. Can the
    user see the current state and failure reasons?
  - **In Scope — billing default:** Billing, metering, and cost management
    are platform-wide concerns. Default to Out of Scope unless billing IS
    the feature's primary purpose.
  - **Out of Scope — security boundary:** For features involving shared
    physical infrastructure (bare metal, GPUs, storage backends), address
    the tenant data boundary: what happens between assignments? If host
    sanitization is not in scope, state it as Out of Scope with the
    responsible service noted.
  - **Persona ownership (re-check after Dimension Triage):** The Dimension
    Triage phase may have marked personas as "Not affected" based on
    dimension definitions alone. Before finalizing user stories, re-apply
    the persona ownership test: does this persona currently perform the
    manual process this feature automates? If yes, they need stories
    regardless of what Dimension Triage concluded. This override takes
    precedence over Dimension Triage persona assignments.
  - **Persona-story alignment:** After writing user stories, verify each
    story's capability matches the persona's role. Infrastructure
    operations (sanitization, hardware lifecycle) belong to Cloud
    Infrastructure Admin. Tenant onboarding, quotas, catalog management
    belong to Cloud Provider Admin.
- **Step 6 (Resolve Outstanding Items):** Skip — leave `[Assumption: ...]`
  and `[Open Question: ...]` markers in place for human resolution.
- **Step 9 (Present to User):** Skip — proceed to Self-Review.

#### Size Calibration

Match output depth to feature complexity:
- Simple feature (1-2 capabilities): 30-50 non-blank lines
- Medium feature (3-5 capabilities): 50-80 non-blank lines
- Complex feature (5+ capabilities): 80-120 non-blank lines

If your PRD is significantly longer than the exemplars for a comparable
feature, you are likely over-engineering it.

### Self-Review

Score the draft against a quality rubric (5 criteria, 0-2 each, /10 total).

| Criterion | 0 | 1 | 2 |
|-----------|---|---|---|
| **WHAT** | Vague, no personas/services | Partially clear, some personas missing | Clear, specific, all affected personas have stories |
| **WHY** | No justification | Generic justification | Concrete pain, impact, strategic tie |
| **User-Facing Focus** | Reads like a design doc | Mostly user-focused, some leakage | Pure user-observable outcomes |
| **Right-Sized** | Bundles 3+ independent capabilities | 1-2 separable capabilities | Focused, capabilities require each other |
| **Testability** | Requirements describe internals | Some testable, some vague | Every requirement PM-verifiable |

**Additional deterministic checks (run alongside the rubric):**

1. **Persona ownership check:** For each persona marked "Not affected,"
   verify the feature does not automate or replace a process they currently
   perform. If it does, they need user stories — mark as a review failure.

2. **Persona-story alignment check:** For each user story, verify the
   capability matches the persona's role definition. Infrastructure
   operations under Cloud Provider Admin, or tenant management under Cloud
   Infrastructure Admin, is a misattribution.

3. **Problem Statement solution check:** The Problem Statement must not
   describe what the feature introduces or how it works. Search for
   "introduces", "eliminates", "provides", "enables" used to describe the
   feature itself. Any match is a failure.

4. **Async status check:** If any In Scope item describes asynchronous
   resource creation, verify that status/progress visibility is also
   addressed in In Scope or User Stories.

**PASS:** total >= 7/10 AND no zeros AND all deterministic checks pass.
Proceed to Session Context.

**FAIL:** total < 7 OR any zeros OR any deterministic check failed.
Enter revision loop.

### Revise (Conditional)

Only runs if Self-Review failed. Capped at `max_revisions` (default: 2).

1. For each criterion that scored 0 or 1, apply targeted fixes:
   - **WHAT:** Add missing persona headings and specific user stories
   - **WHY:** Strengthen problem statement with available evidence (do NOT
     fabricate)
   - **User-Facing Focus:** Reframe design leakage as user outcomes
   - **Right-Sized:** Note the concern but do NOT remove capabilities
   - **Testability:** Rewrite vague requirements with PM-verifiable outcomes

2. Edit `03-prd.md` to address findings.

3. Re-run Self-Review.

4. If still failing after `max_revisions` rounds, document unresolved
   issues and proceed.

### Session Context

Write `.artifacts/prd/{issue-key}/session-context.md`:

```markdown
# Session Context — {issue-key}

## Summary
[1-2 sentences: what feature, what was produced, current state]

## Exemplars Used
[List the exemplar PRDs read and why they were selected]

## Self-Clarify Results
- Self-resolved: {count} gaps
- Open questions: {count} items need human resolution
- Key open questions: [list the most impactful ones]

## Dimension Coverage
[Which dimensions were triaged as applicable and addressed]

## Self-Review Score
- Score: {N}/10 ({per-criterion scores})
- Revision rounds: {N}
- Unresolved concerns: [any criteria still below threshold]

## Open Questions Requiring Human Input
[Consolidated list of all [Open Question: ...] markers from the PRD,
grouped by section, with impact notes]

## Artifacts
- `01-requirements.md` — Raw requirements
- `02-clarifications.md` — Self-clarification log
- `03-prd.md` — Generated PRD
```

## Escalation

The following trigger a **hard stop** — write
`.artifacts/prd/{issue-key}/escalation.md` and terminate:

- **Empty/stub Jira description:** The Feature issue has no meaningful
  description or acceptance criteria. A PRD cannot be generated from
  nothing.
- **Bundled features:** The Feature clearly bundles 3+ independent
  capabilities that serve different personas or purposes. Recommend
  splitting.
- **Content-only deliverable:** The Feature's sole deliverable is
  documentation, example files, or configuration samples with no new
  platform capability. Recommend tracking as a task, not a PRD.

Write the escalation report with:
- Which check triggered escalation
- What was found in the Jira source
- Suggested next steps for the human

Then **stop**. Do not continue past the failed phase.

## Completion Report

When all phases finish, output the following as conversation text only.
Do NOT write the completion report into the PRD file — it is not part of
the PRD artifact:

```text
## Unattended PRD Run Complete

Phases: ingest ✓ → exemplars ✓ → dimensions ✓ → self-clarify ✓ → draft ✓ → review ✓ → context ✓
Artifacts: .artifacts/prd/{issue-key}/
Score: {N}/10 (WHAT:{n} WHY:{n} UF:{n} RS:{n} T:{n})
Open questions: {count} items need human resolution
Revision rounds: {N}
Result: {summary}

Next steps:
- Review open questions in session-context.md
- Run /revise to refine the PRD interactively
- Run /publish when ready for external review
```

## Guidelines

For principles, hard limits, safety, and quality rules, follow
`skills/osac/generate-prd/guidelines.md`. Where guidelines say "stop and request human guidance,"
write an escalation report and terminate (see Escalation above).
