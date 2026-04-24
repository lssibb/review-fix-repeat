# Exit Criteria

When does the RFR loop stop?

## Clean exit (best case)

The loop exits cleanly when Phase 2 or Phase 4 produces an empty `verified_issues` list OR every verified issue has `auto_fix: false` (all remaining issues are for the user to decide — nothing more to mechanically fix).

Report: "All auto-fixable issues resolved. N items for your review." Done.

## Iteration cap exit (safety)

Hard cap: 3 full iterations of Phase 2 → Phase 3 → Phase 4.

If after the 3rd iteration Phase 4 still finds actionable issues, STOP. Do not start a 4th cycle.

Rationale: loops that don't converge in 3 iterations are almost always oscillating — fix A breaks B, fix B breaks A — or the reviewers are finding progressively more trivial issues while the main agent's context is filling with noise. Forcing a stop preserves signal and token budget.

Report format for iteration-cap exit:
```
Loop stopped after 3 iterations with N residual issues. These issues appear
persistent or introduced during fix cycles — please review manually:
  [list]
```

## Convergence detection (bonus)

If you can detect that iteration K produced essentially the same issues as iteration K-1 (same IDs or very similar descriptions), exit early before the 3-iteration cap. This is optional but saves tokens.

Heuristic: if `set(issue.id for issue in iter[K])` ⊆ `set(issue.id for issue in iter[K-1])` with no new IDs, the loop has converged — the remaining issues are truly stuck. Stop and report.

## What counts as "clean"

"No actionable issues" means:
- Zero verified issues with `auto_fix: true`.
- Any remaining issues (with `auto_fix: false`) are architectural/tradeoff items for the user.

It does NOT mean "zero issues of any kind". Architectural discussions and tradeoff decisions are normal output of RFR — they're features, not failures.

## What NOT to exit on

- **Don't exit because a reviewer said "code looks great".** Reviewers praising code isn't a signal; absence of verified issues is.
- **Don't exit on low-severity issues.** Low severity doesn't mean "skip" — it means "fix is small". Auto-fix low-severity issues the same way as high-severity ones if `auto_fix: true`.
- **Don't exit because tests pass.** RFR is about code review, not test execution. Tests passing is necessary but not sufficient.

## Residual issues reporting

In Phase 5, when reporting residual issues to the user, separate them by category:

1. **Needs your decision** — `auto_fix: false` issues with their reasoning.
2. **Stuck after 3 iterations** — issues that survived the loop. These deserve the user's attention most.
3. **Dropped during verification** — a short note like "also dropped N issues as hallucinations/duplicates" for transparency. User can dig into `.rfr/session-*/phase-2/verified.json` if curious.

## When to recommend starting a fresh RFR

Sometimes the right move is to tell the user: "This feature has grown too complex for one RFR pass — consider splitting it and running RFR on each piece separately." Signals that warrant this:

- Phase 1 would need 10+ reviewers.
- The git diff is >1000 lines.
- Multiple unrelated concerns in one change (new feature + refactor + dependency upgrade).

Bail out early in those cases and suggest the split rather than starting a doomed loop.
