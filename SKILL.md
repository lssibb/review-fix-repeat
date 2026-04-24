---
name: rfr
description: Review-Fix-Repeat automated quality loop. Use this skill immediately after completing a feature, fix, or non-trivial change — whenever the user indicates the work is "done", "finished", "ready to commit", asks to "review what I did", "check this feature", or runs commands like /ship, /review, /rfr. Also trigger when the user says things like "проверь что я сделал", "закончил фичу", "можно коммитить?". The skill orchestrates parallel review subagents scoped to the actual areas touched by the change (dynamically determined — could be UI, API, DB, auth, performance, etc.), filters false positives through a verifier pass, dispatches non-overlapping fix subagents, then re-reviews until no actionable issues remain. This is NOT for small edits like typos or color changes — use only when the change is substantive enough to warrant multi-perspective review.
---

# Review-Fix-Repeat (RFR)

A disciplined loop that runs after a feature/fix is complete. Instead of single-pass review, RFR spawns a swarm of specialized reviewers, filters their findings through a verifier, dispatches parallel fixers on non-overlapping zones, and iterates until the feature is clean.

## When this skill should NOT run

Skip RFR entirely when:
- The change is a single-line edit, typo fix, color tweak, copy change, or comment adjustment.
- The user explicitly asks for a quick fix.
- The change is pure documentation (README edits, etc.) with no code.

RFR is designed for substantive changes — new features, non-trivial refactors, multi-file edits, anything touching business logic, auth, data, or API contracts.

## Core loop

```
[feature just completed]
    ↓
PHASE 1: Scope & Dispatch Review Swarm  (parallel)
    ↓
PHASE 2: Verifier Pass                   (single agent)
    ↓
    ├── no actionable issues → PHASE 5 (exit, report clean)
    └── actionable issues found
            ↓
PHASE 3: Fix Swarm                       (parallel, non-overlapping)
    ↓
PHASE 4: Re-Review Swarm                 (parallel — fixes + whole-feature)
    ↓
    ├── no new issues → PHASE 5 (exit, report)
    ├── new issues & iteration < 3 → back to PHASE 2
    └── iteration == 3 → PHASE 5 (exit, report residual issues for human)
```

Hard limits:
- Maximum 3 review/fix iterations. If issues remain after iteration 3, stop and present them to the user — further auto-loops rarely converge and waste tokens.
- Each phase writes its output to `.rfr/session-<timestamp>/phase-<N>/` for observability and debugging.

## Phase 1 — Scope & Dispatch Review Swarm

### Step 1.1: Identify what changed

Determine the scope of the completed work:
- Run `git diff HEAD` (or `git diff <base-branch>` if the user mentions a branch).
- If no git context, ask the user: "What files did this feature touch?"
- Group changed files by conceptual area (see Step 1.2).

### Step 1.2: Determine relevant review areas dynamically

Do NOT use a fixed list of reviewers. Look at what the change actually touches and decide which perspectives are needed. Common areas and when to include them:

| Area | Include when the change touches... |
|---|---|
| **UI / UX** | React/Vue/Svelte components, views, templates, styling, user-facing copy |
| **API contracts** | REST endpoints, GraphQL schemas, tRPC routers, request/response shapes |
| **Database** | Migrations, schema files (Prisma/Drizzle/SQL), queries, ORM models |
| **Auth & permissions** | Login, session handling, middleware, role/permission checks, JWT, cookies |
| **Security** | Input validation, injection surfaces, secrets handling, CORS, CSRF, XSS, rate limiting |
| **Performance** | Loops over large collections, N+1 queries, bundle imports, re-renders, heavy computation |
| **Error handling** | Try/catch coverage, error boundaries, fallback states, user-facing error messages |
| **Types / contracts** | TypeScript types, Zod/Yup schemas, API DTOs, shared interfaces |
| **Tests** | Test files added/modified, or test coverage obviously missing |
| **Observability** | Logging, metrics, tracing, error reporting |
| **i18n** | User-facing strings, locale handling |
| **Accessibility** | Form controls, keyboard nav, ARIA, focus management (when UI changed) |
| **System integrity** | ALWAYS include — checks whether the change broke adjacent unchanged code |

**Selection logic:** if a file in the diff matches any indicator for an area, include that area. Always include **System Integrity** as the last reviewer — its job is to catch breakage in code that wasn't intentionally changed.

Target 3–7 reviewers for most features. Fewer means you missed a perspective. More means you're splitting hairs.

### Step 1.3: Spawn reviewers in parallel

For each selected area, launch a subagent in the same turn (parallel). Each reviewer gets:
- The git diff (or list of changed files with their content).
- The specific area they're reviewing (from the table above).
- A strict output contract (see below).
- Read-only file access — reviewers must not edit code.

**Reviewer prompt template:**

```
You are the {AREA} reviewer in a Review-Fix-Repeat loop.

Your scope: review the following change strictly from a {AREA} perspective.
Ignore issues outside your area — other reviewers cover them.

Changed files:
{list or diff}

For each issue you find, emit a JSON object with these exact fields:
  - id: short stable identifier (e.g., "sec-missing-csrf-1")
  - area: "{AREA}"
  - severity: one of "critical" | "high" | "medium" | "low"
  - affected_files: array of absolute or repo-relative file paths
  - description: 1-3 sentence explanation of the issue
  - requires_discussion: boolean (see criteria below)
  - suggested_fix: short hint, not full code

requires_discussion = true when:
  - The fix is an architectural choice (refactor vs. wrap, extract vs. inline).
  - There are multiple valid approaches and the tradeoffs matter.
  - The issue is a suggestion for a NEW feature not in the original task.
  - The performance cost of the fix is non-trivial and unclear whether it's worth it.

requires_discussion = false when:
  - It's a bug, regression, or type error.
  - A required check is missing (auth, validation, migration).
  - The fix is mechanical (rename, add field, close resource, handle case).
  - There's one obviously correct way to fix it.

If you find no issues, emit an empty array.

Return ONLY valid JSON: {"area": "{AREA}", "issues": [...]}
No preamble, no explanation outside JSON.
```

Save each reviewer's output to `.rfr/session-<timestamp>/phase-1/<area>.json`.

## Phase 2 — Verifier Pass

Spawn ONE verifier agent that reads all `phase-1/*.json` files.

The verifier's job:
1. **Deduplicate** issues reported by multiple reviewers for the same underlying problem — merge into one, keep the most actionable description.
2. **Reject hallucinations** — for each issue, quickly grep the referenced files to confirm the problem actually exists as described. If the reviewer invented the issue, drop it.
3. **Confirm severity** — if a reviewer marked something "critical" but it's clearly a style nit, downgrade.
4. **Flag actionability** — set `auto_fix: true` only if `requires_discussion: false` AND the fix is clearly scoped. Otherwise `auto_fix: false`.

**Verifier prompt template:**

```
You are the Verifier in a Review-Fix-Repeat loop.

Read every JSON file in .rfr/session-{ts}/phase-1/.
Produce a single consolidated issue list.

For each candidate issue from the reviewers:
1. Check it's real — use grep/read to verify the problem exists in the referenced file.
2. Merge duplicates — if two reviewers reported the same root issue, keep one entry
   with merged affected_files and the clearer description.
3. Validate severity against the actual impact.
4. Set auto_fix: true only when the fix is mechanical and requires_discussion is false.

Output a single JSON object:
{
  "verified_issues": [
    {
      "id": "...",
      "areas": ["sec", "api"],        // merged from reviewers
      "severity": "critical|high|medium|low",
      "affected_files": [...],
      "description": "...",
      "suggested_fix": "...",
      "auto_fix": true|false,
      "reason_if_not_auto": "..."     // only when auto_fix is false
    }
  ],
  "dropped_issues": [
    { "id": "...", "reason": "hallucination | duplicate | out-of-scope" }
  ]
}

Save to .rfr/session-{ts}/phase-2/verified.json.
```

If `verified_issues` is empty or all entries have `auto_fix: false`, skip Phase 3 and go to Phase 5 (report).

## Phase 3 — Fix Swarm (non-overlapping)

### Step 3.1: Partition fixes by file ownership

Read `phase-2/verified.json`. Filter to entries with `auto_fix: true`.

Build a file → issues map: for each unique file across all auto-fixable issues, collect the issues that touch it. Then group issues so that no two groups share any file. This is a connected-components problem on a graph where issues are nodes and shared files are edges:

- Start with each issue as its own group.
- For each pair of issues sharing any file, merge their groups.
- The resulting groups are your fix units. Each group goes to ONE fix agent.

This guarantees: two fix agents never touch the same file in parallel. See `references/non-overlap.md` for the detailed algorithm and edge cases.

### Step 3.2: Spawn fix agents in parallel

For each fix group, spawn a subagent:

```
You are a Fix agent in a Review-Fix-Repeat loop.

You own these files (no other agent will touch them this pass):
{affected_files}

Fix these issues:
{issues JSON for this group}

Rules:
- Make only the changes required to fix these issues. Do NOT refactor unrelated code.
- Do NOT modify files outside your owned list.
- After fixing, run relevant local checks (type-check, lint) if commands are obvious
  (tsc --noEmit, eslint, biome, etc.) to catch syntax errors before re-review.
- When done, write a summary to .rfr/session-{ts}/phase-3/fix-<group-id>.json:
  {
    "group_id": "...",
    "issues_addressed": ["sec-1", "api-2"],
    "files_modified": [...],
    "summary": "What you changed and why",
    "checks_run": ["tsc", "eslint"],
    "checks_passed": true
  }
```

Wait for all fix agents to complete before proceeding.

## Phase 4 — Re-Review Swarm

Spawn TWO types of reviewers in parallel:

**Type A — Fix-focused reviewers.** For each fix group, spawn a reviewer that checks whether the fix actually addressed its issues without introducing regressions in the modified files. Use the phase-3 summary as input.

**Type B — Whole-feature reviewers.** Re-run the same area reviewers from Phase 1 (same selection logic — dynamically determined) against the current state of the feature. This catches issues that may have been introduced during fixes, or ones that were shadowed by higher-severity issues in the first pass.

Both types use the same JSON output contract as Phase 1 reviewers. Save to `phase-4/`.

**Branch decision after Phase 4:**
- If Phase 4 returns no actionable issues → Phase 5 (exit clean).
- If iteration count < 3 and new actionable issues exist → go back to Phase 2 with Phase 4 outputs.
- If iteration count == 3 → Phase 5 (exit with residual issues listed).

## Phase 5 — Final Report

Produce a concise report for the user. NO new code changes in this phase. Format:

```
## RFR Session Summary

**Iterations:** {N}
**Reviewers per iteration:** {areas covered}
**Total issues found:** {count}
**Auto-fixed:** {count}
**Requires your decision:** {count}
**Session log:** .rfr/session-{ts}/

### Auto-fixed (no discussion needed)
- [sec-1] Missing CSRF check on /api/admin/delete — added middleware. Files: src/api/admin/delete.ts
- [api-2] Response shape mismatch with client type — aligned. Files: src/api/users/[id].ts, src/types/user.ts
...

### Needs your input (architectural / tradeoff)
- [perf-3] N+1 query in dashboard list. Options:
    a) Add Prisma `include` — simpler, small extra cost per request.
    b) Batch-load via DataLoader — more code, bigger win at scale.
  Files: src/pages/dashboard/index.tsx
...

### Residual issues after iteration 3 (auto-loop gave up)
{list, or "none"}

### Suggested commit message
{follow the repo's conventional commit style if detectable; else a sensible default}
```

Do NOT run `git commit` automatically. The user decides when to commit.

## Session telemetry

Every session writes to `.rfr/session-<timestamp>/`:
```
.rfr/session-20260425-143055/
├── phase-1/
│   ├── ui.json
│   ├── api.json
│   ├── security.json
│   └── system-integrity.json
├── phase-2/
│   └── verified.json
├── phase-3/
│   ├── fix-group-1.json
│   └── fix-group-2.json
├── phase-4/
│   ├── fix-review-1.json
│   ├── fix-review-2.json
│   ├── ui.json       # whole-feature re-review
│   └── ...
└── report.md
```

This directory is gitignored by default — add `.rfr/` to `.gitignore` on first run if not already there.

## References

Read these when you need more detail on a specific phase:
- `references/non-overlap.md` — the file-partitioning algorithm for fix groups, with edge cases.
- `references/verifier.md` — extended rubric for verifier decisions (dedup, hallucination checks, severity calibration).
- `references/exit-criteria.md` — when to stop looping and what counts as "clean".

## Common failure modes to avoid

- **Skipping Phase 2.** The verifier is the most important step — reviewers hallucinate. Without it, fix agents chase ghosts.
- **Letting fix agents "improve" things.** Fix only what the issue says. Any other change = Phase 4 re-reviewers will freak out and you get false-positive loops.
- **Overlapping fixes.** If two agents touch the same file, you get merge chaos. The non-overlap algorithm in Phase 3 is not optional.
- **Running RFR on every tiny change.** Overkill — read the "When this skill should NOT run" section above.
- **Treating requires_discussion=true as auto-fixable.** Those go to the user. Forcing a fix on architectural choices creates worse code than no fix.
