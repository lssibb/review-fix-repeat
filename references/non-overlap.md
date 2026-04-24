# Non-Overlap Algorithm for Fix Groups

Goal: partition a set of verified auto-fixable issues into groups such that no two groups share any file. Each group is assigned to exactly one fix agent, running in parallel with other groups. Two agents touching the same file causes merge chaos and must never happen.

## The algorithm (connected components)

Model the problem as a graph:
- **Nodes:** verified issues where `auto_fix == true`.
- **Edges:** an edge between two issues iff their `affected_files` sets intersect.

The groups are the connected components of this graph.

### Reference implementation (JS-ish pseudocode)

```js
function partitionIntoFixGroups(issues) {
  // issues: [{ id, affected_files: [...], ... }]

  const parent = new Map();
  const find = (x) => {
    if (parent.get(x) !== x) parent.set(x, find(parent.get(x)));
    return parent.get(x);
  };
  const union = (a, b) => { parent.set(find(a), find(b)); };

  // Initialize: each issue is its own component.
  for (const issue of issues) parent.set(issue.id, issue.id);

  // Index: for each file, which issues reference it?
  const fileToIssues = new Map();
  for (const issue of issues) {
    for (const f of issue.affected_files) {
      if (!fileToIssues.has(f)) fileToIssues.set(f, []);
      fileToIssues.get(f).push(issue.id);
    }
  }

  // Union-find: any two issues sharing a file get merged.
  for (const [, issueIds] of fileToIssues) {
    for (let i = 1; i < issueIds.length; i++) {
      union(issueIds[0], issueIds[i]);
    }
  }

  // Collect components.
  const groups = new Map();
  for (const issue of issues) {
    const root = find(issue.id);
    if (!groups.has(root)) groups.set(root, []);
    groups.get(root).push(issue);
  }

  // Each group now has a disjoint set of files from every other group.
  return [...groups.values()].map((groupIssues, idx) => ({
    group_id: `fix-group-${idx + 1}`,
    issues: groupIssues,
    files: [...new Set(groupIssues.flatMap((i) => i.affected_files))],
  }));
}
```

You don't need to run this as code — you can do it mentally or on paper for small sets (which is the typical case: 3–15 issues). Use the reference when the set is large or when you're unsure.

## Edge cases

### One issue touches many files
That's fine. The issue still goes into one group. The group's `files` is the union.

### All issues touch one common file (e.g., a central config)
They all merge into a single group → one fix agent. This loses parallelism but preserves correctness. Accept it. If you notice this happening often for legitimate reasons (e.g., a `schema.prisma` central to everything), consider asking the user whether to let multiple agents sequentially edit it instead of all-at-once; for now, default to serial.

### An issue affects zero files
That's suspicious — it means the reviewer reported something abstract. Flag it for the verifier to re-check. If it survives as an auto-fixable issue with no files, drop it from Phase 3 and report in Phase 5 as "advisory — no file target identified".

### Two issues with identical file lists
Put them in the same group (they'd union anyway). The fix agent should address both in one pass.

### A fix group has conflicting suggestions
Example: two issues on the same file where fix A would undo fix B. Rare, but possible. The fix agent should notice and resolve holistically, or note the conflict in its summary so Phase 4 reviewers can catch it.

## Anti-patterns

**Don't partition by area.** Sec issues and API issues might touch the same file — two agents editing the same file in parallel breaks everything regardless of "conceptual area".

**Don't partition by severity.** Same reason.

**Don't let a fix agent touch a file outside its assigned list.** If it wants to, it means the partition missed a dependency — the agent should refuse and write to its summary "need file X outside my scope". Phase 4 picks this up.

**Don't run fix agents sequentially when parallel works.** Sequential is the fallback for single-file-bottleneck cases. Parallel is the point of this whole thing.

## Sanity check before dispatch

Before spawning fix agents, verify:
- Every file in the auto-fixable set appears in exactly ONE group.
- No file is shared across groups.
- The total files across all groups == the union of `affected_files` across all auto-fixable issues.

If any check fails, re-run the partitioning. If it fails again, something in the data is malformed — drop back to Phase 2 and ask the verifier to re-check.
