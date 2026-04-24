# Verifier Rubric

The verifier sits between Phase 1 (review swarm) and Phase 3 (fix swarm). Its existence is the single biggest differentiator between RFR and naive "ask reviewer then fix" patterns. Reviewers hallucinate, duplicate, misjudge severity, and sometimes invent plausible-sounding issues that don't actually exist in the code. Without a verifier step, fix agents chase ghosts.

## The verifier's four jobs

### 1. Confirm existence (anti-hallucination)

For every candidate issue, grep or read the referenced file(s) and confirm the described problem is actually present.

Common hallucination patterns to watch for:
- Reviewer says "missing null check on `user.email`" but `user.email` is already validated upstream.
- Reviewer says "no rate limiting on this endpoint" but there's middleware that applies globally.
- Reviewer says "type `UserDto` doesn't match the API response" but the types do match (reviewer was looking at an old version).
- Reviewer says "this uses deprecated API X" but X isn't actually deprecated in the installed version.

If the problem doesn't exist as described → drop it, log reason `hallucination`.

If the problem exists but is slightly different from the reviewer's description → keep it, rewrite the description to be accurate.

### 2. Merge duplicates

Two reviewers often find the same underlying issue from different angles. Example:
- Security reviewer: "No input validation on POST /api/users body"
- API reviewer: "Request body accepted without schema — could cause runtime errors"

Same root cause. Merge:
- Combined areas: `["security", "api"]`
- Pick the more actionable description (usually the one with clearer remediation).
- Union affected_files.
- Take the higher severity.

Do NOT merge issues that share files but are actually different problems. Example: "missing auth check" and "N+1 query" on the same endpoint are two separate issues — they go into the same fix GROUP (by the non-overlap algorithm) but stay distinct in the verified list.

### 3. Calibrate severity

Reviewers in isolation often mis-grade. Common corrections:

| Reviewer said | Actual severity is probably... | When |
|---|---|---|
| critical | high or medium | If it's a theoretical attack with no realistic path |
| critical | stays critical | If it's production data loss, auth bypass, exposed secrets |
| high | medium | If it's a code-smell, not a correctness issue |
| medium | low | If it's a stylistic preference |
| low | drop entirely | If it's bikeshedding |

Rule of thumb: would a senior engineer block a PR over this? If no, probably not critical/high.

### 4. Set auto_fix flag

This is where the verifier filters issues into "fix right now without asking" vs. "show the user".

**Set `auto_fix: true` when ALL of these hold:**
- `requires_discussion` was false (reviewer's own assessment).
- The fix is mechanically clear (rename, add field, missing check, fix type, close resource).
- The fix scope is bounded — you can predict which files change and roughly how many lines.
- There's no architectural tradeoff.

**Set `auto_fix: false` (escalate to user) when ANY of:**
- The reviewer marked `requires_discussion: true`.
- There are multiple valid fixes and choosing between them is a tradeoff.
- The fix requires changing a public API / exported interface / shared type used elsewhere.
- The fix would add a new dependency.
- The fix would introduce new architectural concepts (new abstraction, new pattern).
- The fix crosses 5+ files or is otherwise large — even if clear, the blast radius deserves review.

When `auto_fix: false`, fill in `reason_if_not_auto` with a short explanation. The user will see this in the final report.

## Verifier output format

```json
{
  "verified_issues": [
    {
      "id": "sec-missing-csrf-1",
      "areas": ["security", "api"],
      "severity": "critical",
      "affected_files": ["src/api/admin/delete.ts"],
      "description": "DELETE endpoint lacks CSRF protection; CSRF middleware is applied globally but this route is registered before the middleware.",
      "suggested_fix": "Move route registration after csrfMiddleware, or apply csrfMiddleware inline.",
      "auto_fix": true
    },
    {
      "id": "perf-n-plus-one-1",
      "areas": ["performance", "db"],
      "severity": "medium",
      "affected_files": ["src/pages/dashboard/index.tsx", "src/lib/queries/dashboard.ts"],
      "description": "Dashboard loads projects then iterates to fetch each project's task count — N+1.",
      "suggested_fix": "Use Prisma `_count` include or aggregate in a single query.",
      "auto_fix": false,
      "reason_if_not_auto": "Two valid approaches (include vs. aggregate) with different tradeoffs for large counts — worth your decision."
    }
  ],
  "dropped_issues": [
    { "id": "ui-copy-1", "reason": "hallucination — referenced text does not exist in file" },
    { "id": "sec-sql-injection-2", "reason": "duplicate of sec-sql-injection-1, merged into it" }
  ]
}
```

## What makes a good verifier

- Reads fast, reasons carefully.
- Isn't intimidated by a confident-sounding reviewer. Confidence ≠ correctness.
- Treats every issue as a claim to verify, not a fact to accept.
- Errs on the side of DROPPING rather than KEEPING borderline issues. A false positive wastes a fix-agent cycle; a false negative becomes a Phase 4 finding (cheaper).
- When in doubt about `auto_fix`, defaults to `false`. Asking the user costs 30 seconds; auto-fixing something architectural costs hours to unwind.

## Anti-patterns

- Verifier trusts the reviewers blindly — defeats the whole point.
- Verifier becomes another reviewer and adds new issues — it shouldn't. Its scope is strictly filter + calibrate on inputs.
- Verifier rewrites descriptions to be more verbose — keep them 1–3 sentences, actionable.
- Verifier marks everything `auto_fix: true` to avoid bothering the user — wrong calibration. The user would rather approve 3 decisions in 30 seconds than wake up to a refactored codebase they didn't sign off on.
