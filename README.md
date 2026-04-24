# RFR — Review-Fix-Repeat

A Claude Code skill that runs an automated multi-agent review/fix loop after you complete a feature.

## What it does

After you finish a feature, invoke RFR (or say "проверь что я сделал" / "done with the feature"). The skill will:

1. **Scope the change** — reads the git diff and dynamically picks the right review perspectives (UI, API, security, DB, performance, etc.) based on what actually changed. No fixed list of reviewers.
2. **Parallel review swarm** — spawns one subagent per relevant area plus a system-integrity agent.
3. **Verifier pass** — a single agent dedupes findings, rejects hallucinations, and flags which issues are mechanically auto-fixable vs. which need your decision.
4. **Parallel fix swarm** — auto-fixable issues are partitioned so no two fixers touch the same file, then dispatched in parallel.
5. **Re-review** — new reviewers check the fixes AND re-scan the whole feature.
6. **Loop** — up to 3 iterations. Then a clean report with what was auto-fixed, what needs your input, and what stayed broken.

Commit is NOT automatic. The final report includes a suggested commit message; you decide when to commit.

## Install

**Global install** (available in all projects):
```bash
mkdir -p ~/.claude/skills/rfr
cp -r ./rfr/* ~/.claude/skills/rfr/
```

**Project-local install** (only in one repo):
```bash
mkdir -p <your-repo>/.claude/skills/rfr
cp -r ./rfr/* <your-repo>/.claude/skills/rfr/
```

Add `.rfr/` to your `.gitignore` — that's where session telemetry lives.

## Use

After finishing a feature, just say any of these to Claude Code:
- "Done with the feature — run RFR"
- "/rfr" (if you set up a slash command)
- "Проверь что я сделал"
- "Ready to commit, review first"

The skill triggers automatically on those cues (and similar ones) per the description.

## What RFR does NOT do

- Does not run on tiny changes (typos, color tweaks). Use `/gsd:quick` or direct prompts for those.
- Does not commit your code. You stay in control of git.
- Does not replace tests. It reviews what exists; it does not enforce TDD. (If you want enforced TDD, layer Superpowers on top.)

## Known limitations

Claude Code is prompt-driven, so the skill works ~80% as strictly as specified. The main agent may occasionally skip a phase or merge steps. If you notice consistent deviations, the fix is to refine the skill's SKILL.md — or graduate to GSD v2 / Pi SDK which offer actual programmatic control over the pipeline.

## Files

```
rfr/
├── SKILL.md                          # Main skill (loaded whenever RFR triggers)
└── references/
    ├── non-overlap.md                # File-partitioning algorithm for fix groups
    ├── verifier.md                   # Detailed verifier rubric
    └── exit-criteria.md              # When to stop looping
```

Reference files are loaded on-demand when the skill needs the detail.
