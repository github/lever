---
name: experiment-arms
user-invocable: false
description: Runs many single-hypothesis "arms" as do-not-merge PRs, measures each with paired, interleaved runs against a cost-matched control, keeps only the arms that clear a keep bar set in advance, records every negative, and composes the winners into a stack that is re-measured on real traffic after merge. It opens PRs and burns CI, so it runs only on `/lever arms` or an explicit ask to measure ideas ("which of these ideas actually helps?", "is this change worth it?", "try that tip I saw", "run arms on this") for a CI, perf, eval, benchmark or cost hill-climb where the system can be run on demand.
---

# Experiment arms

The experiment arm method turns a pile of ideas into measured, landed improvements. Each hypothesis gets its own small, revertible PR (an **arm**) that exists only to be measured. Arms run in parallel against a shared control that does the same work, and each one is judged against a keep bar you wrote down before looking. Winners get composed into a stack and landed at whatever autonomy level the repo allows. Losers become closed `[negative result]` PRs plus a ledger row, so no one has to try them again. The number you report at the end comes from real traffic after merge, not from the estimate.

Start from a Loop Card ([`templates/loop-card.md`](../../templates/loop-card.md)). If you don't have one yet, run `find-the-loop` first.

## When to use / not use

Use it when:
- the metric has run-to-run noise and you can run the real system on demand (CI, a perf harness, an eval, a benchmark);
- there are several candidate changes, or one popular tip nobody has measured on your setup;
- the result must survive review by someone who will ask "compared to what?".

Don't use it when:
- one deterministic test settles the question. Write the test.
- the expected effect is smaller than the noise floor and you can't afford the n needed to see it. Pick a different instrument or a bigger lever.
- the system can't be run on demand. Observe production instead and say the evidence is weaker.

## Roles

| Role | Owns |
|---|---|
| Coordinator (you, or one session) | Loop Card, hypothesis table, ledger, budget, cutoff, status |
| One worker per arm | One change, its arm PR, its measurement, a KEEP/DROP report with numbers |
| Reviewer on a different model | Critique of the hypothesis table and the final stack (see `cross-model-review`) |

A coordinator that starts editing code stops coordinating. Keep it to orchestration.

## Steps

1. **Name the felt metric and the statistic.** Use the number a person actually experiences: PR time-to-green, not one job's duration; long frames, not render count. Report p50 and p90. Note any guard metrics (cost, failure rate, test count) that must not regress.
2. **Baseline and critical path.** Pull 50–150 historical samples. Decompose them and name the slowest required lane, since only changes to the critical path move the felt metric. Write the result to `baseline.md`.
3. **Measure noise first (A/A).** Run the control against itself (two identical arms) for at least 3 rounds. The spread of those deltas is the noise floor. If the keep bar is inside it, raise n or change the instrument before you run a single treatment.
4. **Write the hypothesis table.** One row per arm, with these columns: concept, expected Δ, on critical path?, risk, equivalence proof, owner. One concept per arm and no drive-by edits. Read the ledger first, and don't re-run a recorded negative without a new reason.
5. **Pre-register the keep bar** in the Loop Card before any treatment runs. The default: the arm wins against control in **every** round, the median paired Δ beats the bar and is outside the noise floor, the 95% CI of the paired Δ excludes zero, and equivalence passes (same test identities, byte-identical outputs, same eval answer key).
6. **Freeze a measurement base**, a branch or SHA that stays fixed for the whole round. Branch every arm from it.
7. **Build arms in parallel, in batches.** Run as many at once as your spend tier's lane cap and the repo's `budgets.max_parallel_arms` allow, whichever is lower; queue the rest for the next batch. Each gets one worktree, one worker and one draft PR from a `lever/arm-<name>` branch, titled `[measurement — do not merge] <arm>` with a do-not-merge label. Push early so other work can stack on it. Don't serialize measurement while waiting for an earlier arm's result. Run everything now and decide the order afterwards.
8. **Make the control cost-matched.** The control must do the same work as the treatment. Put the same nonce change in every arm *and* the control so that caches bust equally and path filters can't skip lanes. Reject any arm whose heavy checks were skipped. An empty commit that goes green in 2 minutes is not a control.
9. **Run paired, interleaved rounds.** In each round, push every arm and the control together. Pilot with 3 pairs, decide with n ≥ 6 per arm (12 for small effects), and use n ≥ 10 for any headline number. A failed or missing treatment run counts against the arm, not as a skipped pair. Run at least 2 rounds. Label small-n p90s as exploratory. For CI, use [`scripts/ci_arm_driver.py`](scripts/ci_arm_driver.py) (see [`ci-adapter.md`](ci-adapter.md)).
10. **Watch the base.** If `main` (or the harness) breaks, void every round that ran on it. Merge the fix into every arm's measurement commit, since that's for measurement only, and re-run.
11. **Decide keep or kill** against the bar from step 5, using [`scripts/arm_stats.py`](scripts/arm_stats.py) (`--min-pairs`, default 6; it checks reliability before speed). Write every DROP to the ledger ([`templates/ledger.md`](../../templates/ledger.md)) with its numbers, close its PR, and retitle it `[negative result] <arm>`. Re-test popular tips like any other arm: a widely shared one can lose on your hardware.
12. **Compose the winners.** Order them by dependency, then by measured value. Run one combined confirmation round (about 5 pairs) of the full stack against control, because effects don't always add up. If interactions are unclear, report a range ("between A and B depending on whether X and Y stack").
13. **Land at the configured level.** Open the real PRs, validate each one on its exact head SHA, and hand the set to `landing-plan`. Arms stay at L1. Winners land at the level resolved by [`docs/autonomy.md`](../../docs/autonomy.md).
14. **Re-measure after merge on real traffic** at +24 h and +72 h. Publish the estimate next to the realized result, and say which is which. Paired arms are the *decision* instrument; post-merge traffic is the *reporting* instrument.
15. **Clean up.** Close every remaining arm and validation PR, file follow-ups, and feed the realized-vs-estimated error back into the next Loop Card.

Report status only when something changes, in the form `metric: x → y (Δz%)`, with p50/p90, arms running/kept/dropped, budget used, and evidence level ([`templates/status-digest.md`](../../templates/status-digest.md)).

## Dials

| Dial | Default | Options |
|---|---|---|
| Metric / statistic | felt metric, p50 + p90 | any metric with an instrument the agent can run |
| Keep bar | wins every round; median paired Δ ≥ bar and outside A/A noise; 95% CI excludes 0; equivalence passes | absolute (e.g. ≥ 1 min on the critical path), relative (≥ X%), cost or reliability win with no regression |
| A/A rounds before treatments | 3 | 0 (only if the noise floor is already known and recent) |
| Pairs per arm | pilot 3 → decide 6 → headline 10+ | 12+ for small effects |
| Rounds | 2 | 3 for borderline arms |
| Max concurrent arms | min(tier lane cap, `budgets.max_parallel_arms`), batched | hard cap 10, so shared runners aren't starved for real users |
| API rate floor | pause below 3,000 remaining core calls | any; set a shared budget across all agents |
| Poll interval | ≥ 4 min, query by head SHA only | longer for long jobs; never tight loops |
| Drivers per arm set | 1 | a duplicate driver's push cancels runs |
| Compute budget | the budget `/lever` hands down; billed minutes per arm, reported (unknown runner cost says unknown, not $0) | hard cap per round |
| Rerun policy | rerun failed jobs ≤ 2× as a person would; count reruns in the metric | 0 for pure-speed metrics on a stable fleet |
| Broken-base policy | void the round; merge the fix into every arm | pause until the base is green |
| Arm PRs | draft + do-not-merge label, closed at hand-off | local-only arms (no PR) for harness-only metrics |
| Negative results | closed `[negative result]` PR + ledger row | ledger row only |
| Merge authority for winners | repo ceiling via `docs/autonomy.md` | L2 hand-off · L3 on a recorded grant · L4 gated self-merge or enqueue |
| Cutoff | none | wall-clock time after which no new arms start |
| Post-merge re-measure | +24 h and +72 h | off only if the metric has no production signal |

## Outputs

- `baseline.md`: historical p50/p90, critical path, noise floor.
- A hypothesis table and a ledger, with every arm, kept or dropped, and numbers.
- One draft do-not-merge PR per arm, plus closed `[negative result]` PRs for the drops.
- A stack of real PRs, each with a before/after table, cost delta and risk.
- Final report: estimated vs realized `x → y (Δz%)`, p50/p90, n, cost delta.

## This skill's own feedback loop

- **Estimate error:** realized post-merge Δ vs pre-merge paired Δ. If they keep disagreeing, the instrument is missing something, such as flake reruns or load.
- **Hit rate:** kept arms / total arms. A rate near 100% means the bar is too loose or you're only testing safe bets.
- **Repeat negatives:** arms re-run without a new reason. This should be 0.
- **Budget incidents:** rate-limit exhaustion and starved runners. This should be 0.

## Failure modes it prevents

- Comparing against history instead of a paired control, or against a cheap control that skipped the real work.
- Declaring wins from n = 1, or from deltas inside the noise floor.
- Waiting for each arm in turn when they could all run now.
- Speed-ups that weaken the signal: fewer tests, skipped lanes, retries hiding failures.
- Agents exhausting the shared API quota or runner pool.
- Re-trying an idea the team already disproved.
- Reporting a projection as a result.

## Why this exists

- In one CI speedup project across two repos, 79 do-not-merge experiment PRs produced 27 kept changes in about 1.5 days. Measured on real traffic after merge, p50 time-to-green fell 43% in one repo and 34% in the other.
- The first control was an empty commit. Path filters skipped the heavy lanes, so it went green in 2.25 minutes. Controls must do the same work as treatments.
- "Why do we have to wait until 4pm?" That one question turned serial measurement into parallel arms with ordering decided afterwards.
- The project began from a viral Windows Dev Drive tip. It lost in both paired runs on those runners and broke checkout credentials. Meanwhile "cache corruption" turned out to be page-file exhaustion (15.2% → 0% failures), so the speed work also found reliability bugs.
- Agents polling CI made about 45k API calls in 3 hours and took down unrelated tools. The rate floor, poll interval and arm cap are here because of that.
