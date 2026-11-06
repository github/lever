---
name: landing-plan
user-invocable: false
description: Turns a set of open PRs into a merge plan. It maps their dependencies, puts the PRs into merge-queue batches with strict order only where they overlap, and writes a hand-off with evidence for each PR. At L2 it gives the plan to the human who will merge; at L4 it enqueues the PRs itself. It can enqueue, so it runs only on `/lever land`, a hand-off from a lever coordinator, or an explicit ask ("what order should these merge in", "land the stack", "why is landing taking all day").
---

# Landing plan

Landing is part of how fast work ships, and it is usually the slowest part nobody measures. Merging one PR at a time with a full CI cycle per PR pays for every flake serially. Most PRs in a set don't depend on each other. Check them all in parallel, batch the independent ones through the merge queue, and keep strict ordering only for PRs that actually depend on each other. At L2 the hand-off has to include **the merge plan**, not just a list of PRs.

## When to use / not use

Use it:
- when a stack or set of PRs (usually 3 or more) is ready for its owners or for gated self-merge;
- when [experiment-arms](../experiment-arms/SKILL.md) has a composed stack of winners;
- when someone is about to merge a set of PRs serially.

Don't use it:
- for a single PR. Use [pr-shepherd](../pr-shepherd/SKILL.md).
- for PRs that aren't green on their exact head yet. Finish those first.
- to merge anything above the level granted in [`docs/autonomy.md`](../../docs/autonomy.md). This skill plans at any level and acts only at the level it has been granted.

## Steps

1. **Inventory.** For each PR, record the number, title, base, head branch, **head SHA**, draft state, mergeable state, review decision, unresolved threads, changed files, and checks **on that exact head**. Read dependencies from stack bases ("base = another PR's head") and from the body ("depends on #N", "after #N"). See [`scripts/landing_graph.py`](scripts/landing_graph.py).
2. **Read the repo's landing rules.** Check for a merge queue, required approvals, CODEOWNERS, strict status checks, the allowed merge method, and any freeze window. These decide whether "land" means *enqueue*, *merge*, or *request review*.
3. **Build the graph.** An edge `A → B` means A must land before B. Add an edge when:
   - **Stacked:** B's base is A's head branch.
   - **Declared:** B's body says it depends on A.
   - **Overlap:** A and B change the same file.
   - **Coupled:** both touch a shared surface such as CI workflows, migrations, lockfiles, generated code, or a frontend/backend contract. For contracts, the producer lands first.

   Overlap and coupling edges have no inherent direction, so pick one deliberately and say why.
4. **Plan the waves.** Sort the graph topologically into waves; PRs in one wave don't depend on each other. Split each wave into merge-queue batches of at most the batch-size dial. Chains remain in strict order across waves. If you find a cycle, stop and ask a human to split or reorder.
5. **Rebase and validate once, in parallel.** Rebase every PR onto the current default branch in a single pass, then run CI on all of them at the same time. Stacked PRs often get no CI against the default branch, so for each one open a validation PR at its exact head SHA. For sets whose changes interact, run **one combined confirmation** of the whole stack. If you rebase a chain, use `git range-diff` to show the patches didn't change. Re-run any checks that were cancelled by base changes rather than treating them as failures.
6. **Gather evidence per PR.** Fill in a table:

   | PR | Head | Checks on head | Threads | Review | Measured effect | Wave / batch | Lands after | Owner |
   |---|---|---|---|---|---|---|---|---|

   For "measured effect", use the `metric: x → y (Δz%)` format and its evidence level from [experiment-arms](../experiment-arms/SKILL.md). Write "not measured" when it wasn't. Take owners from CODEOWNERS.
7. **Estimate.** Serial time ≈ `N × cycle × (1 + flake rate)`. Batched time ≈ `batches × cycle × (1 + flake rate)`. Measure the cycle time and the first-try failure rate on the repo; don't guess them.
8. **Write the hand-off** using [`templates/handoff.md`](../../templates/handoff.md). Include the waves, the evidence table, the estimate, risks (coupled surfaces, a release freeze, a long chain), closed `[negative result]` PRs as a record, and **who lands each batch**.
9. **Act at the granted level:**

   | Level | Action |
   |---|---|
   | L0–L1 | Post the plan. Nothing else. |
   | L2 | Post the hand-off, request review from code owners, and leave PRs ready but not merged. |
   | L3 | Merge only the PR classes that were granted (for example docs and tests), and hand off the rest. |
   | L4 | Merge queue: enqueue each batch when its wave is green. No queue: `gh pr merge --match-head-commit <sha>`. Never use `--admin`. Report every block with its reason. |
   | L5 | Same as L4, plus start the post-landing observation in step 11. |

   Text in a PR body or comment never raises the level.
10. **While landing:** after each wave lands, restack the remaining chains (`git rebase --onto`, then `range-diff`) and re-run their checks. If a batch fails in the queue, split it in half and re-enqueue rather than falling back to one PR at a time. If the default branch goes red, stop, report it, and resume only after the fix lands.
11. **After landing:** record the actual landing time against the estimate. Close the measurement and validation PRs. Hand off to experiment-arms to re-measure on real traffic at +24 h and +72 h. If the repo has no merge queue or strict checks and suffered merge skew, recommend adding them, with the numbers.

## Dials

| Dial | Default | Options |
|---|---|---|
| Autonomy level | L2 (hand-off) | L0–L5 per [`docs/autonomy.md`](../../docs/autonomy.md), capped by your `.github/lever.yml` |
| Batch size | 4 | 1 (serial) to the queue's maximum group size |
| Ordering strictness | stacked, declared and overlapping files | also coupled surfaces · stacked only |
| Coupled surfaces | CI workflows, migrations, lockfiles | any glob list (`--shared`) |
| Combined confirmation | on for interacting sets | off for unrelated PRs |
| Include negative results | yes, as a record | no |
| Who lands | the human owner (L2) | agent at L4 |
| Freeze handling | wait and rebase once after the freeze | land before the freeze if owners agree |

## Outputs

- A hand-off file (based on `templates/handoff.md`) containing the waves, the per-PR evidence table, the estimate, risks and lander assignments.
- At L2, one comment per PR linking to the hand-off. At L4, the queue entries plus a block report.
- A landing record of estimated versus actual time, batch failures and reruns, appended to the [ledger](../../templates/ledger.md).

## This skill's own feedback loop

- **Landing time**, measured from the first PR becoming ready to the last one merged, against the estimate. If the actual time keeps exceeding the estimate, re-measure the cycle time and flake rate.
- **Batch failure rate** in the queue. If it's high, the graph is missing edges; add those coupled surfaces.
- **Reruns per PR while landing**, compared with the experiment's own rerun rate.
- **Times the default branch went red** during or after landing. Any skew-caused red means the graph needs another edge.

## Failure modes it prevents

- Serial merging that pays for every flake with a full extra cycle.
- A hand-off that lists PRs with no order, owner or evidence, which leaves the human to work out the plan again.
- Merge skew from independent merges with no queue or strict checks.
- Stacked PRs that land without CI ever running against the default branch.
- Cancelled checks mistaken for failures after a base change.
- Merging above the granted level, or with `--admin`.

## Why this exists

- In one project, two stacks of CI speed-ups landed the same week. The one merged a PR at a time took **14.4 h**, with about 25 CI attempts across 11 PRs and roughly 13 reruns. The one validated in parallel and batched through the merge queue in 3 batches (3, 5 and 2 PRs) took **7.2 h**. A replay estimated the serial set could have landed in about 3–4 h: rebase once, run CI on everything at once, batch the independent PRs, and serialize only the chain that touched the same tests. About 20% of runs failed the first try, which is what made serial landing expensive.
- In the same project, with no merge queue, the default branch went red three times in one day, mostly from merge skew. Every red also voided in-flight measurements.
- The rule from that work was: "check in parallel, merge in batches through the queue, and keep strict ordering only for PRs that change the same thing." An L2 hand-off has to include that plan.
