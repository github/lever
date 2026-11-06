# ci-speed automation

**Purpose.** Keep CI fast and trustworthy continuously, not in a quarterly cleanup sprint. CI gets worse every week: the code grows, tuning such as compile locks and page-file size drifts, and each bottleneck you fix exposes the next one. This recipe turns [`experiment-arms`](../../skills/experiment-arms/SKILL.md) into a standing lane.

The recipe has two automations:

| Automation | Trigger | Tier | Job |
|---|---|---|---|
| `ci-speed-watch` | `interval: daily` (off-peak) | Standard | Measures time-to-green p50/p90, first-try pass rate and the critical path. When a metric misses its target, it opens or updates **one** goal issue. Otherwise it stays silent. |
| `ci-speed-lane` | `issue_labeled` with `labels: [ci-goal]` | Premium | Owns one goal at a time. Runs paired arms on real CI within hard caps, hands a stacked PR set with evidence to the owners, measures again after merge, then closes the goal or files the next bottleneck. |

The cheap watch decides when the expensive lane wakes. **Triggers decide cost.**

The watch is Copilot, so its label won't wake the lane unless that one trigger has `allow_copilot_triggers: true`. Turn it on there and nowhere else. Two things keep the chain from looping: the lane dedupes on the goal (one open goal at a time; a second label event on an owned goal is `NOOP(owned)`), and the lane gets a daily run limit (start at 2).

## What a run looks like

1. **Measure what people feel:** time from push until every required check is green, at p50 and p90. Only the slowest required lane matters, so find the critical path.
2. **One change per PR,** small and revertible, labeled do-not-merge while it's being measured.
3. **Prove it on real CI** with paired runs on the same SHA, at least 6 per arm, with the same test counts and verdicts. A failed treatment run counts against the arm. Keep a change only if it moves the critical path by a minute or more, or clearly cuts cost or flakes. Write down the arms that fail.
4. **Run arms in parallel, decide the order later.**
5. **Hand the stack to the owners** with the evidence in each description, landed at the resolved [autonomy level](../../docs/autonomy.md). Validate in parallel, merge in batches, and serialize only PRs that touch the same thing ([`landing-plan`](../../skills/landing-plan/SKILL.md)).
6. **Measure again on real traffic after merge,** and post the numbers.

## Dials

| Dial | Default | Options |
|---|---|---|
| `TARGET_P50_MIN` / `TARGET_P90_MIN` | Set from the baseline | Minutes |
| `TARGET_FIRST_TRY_PASS` | Set from the baseline | Percent |
| `KEEP_BAR` | Critical path −1 min, or a clear cost or flake cut | Your bar, written before measuring |
| `RUNS_PER_ARM` | 6 paired | 6–12 |
| `MAX_CONCURRENT_ARMS` | 4 | Never above your spend tier's lane cap or `budgets.max_parallel_arms`. Keep it low enough not to starve teammates' runners |
| `MAX_OPEN_PRS` | 8 | Review capacity is the real limit; a stack that outruns its owners rots |
| `CI_MINUTES_PER_GOAL` | Set per repo | Hard stop; report Actions minutes spent per minute saved |
| `API_RATE_FLOOR` | Pause below 20% of the hourly quota | Agent polling can burn a full quota and break unrelated tools |
| `RUN_WINDOW` | Off-peak hours | Any |
| `AUTONOMY` | L1: PRs for owners. A safe floor for a team new to agents. | Raise it as the team trusts the evidence, up to the repo ceiling; L4 with a standing grant for CI-config changes |

## Guardrails

These are fixed and are not dials:
- No speedups that weaken the signal: no skipped tests, blanket retries or lower coverage.
- Never widen `permissions:`, add `pull_request_target` or touch secrets. Keep code-owner review on `.github/workflows/`.
- Flakes become issues for the owning team with a repro, not retries.
- Close every measurement PR at the end of a goal.
- Re-baseline whenever the runner images change.

## Outcome metric

- **Primary:** p90 time-to-green on real PR traffic. The long tail is what makes CI feel unpredictable.
- **Also:** p50, first-try pass rate, flake rate, billed Actions minutes.
- **Cost:** Actions minutes spent per minute of p50 saved.
- **Guardrail:** test count and coverage never drop.

## What's possible

In one run of this method on two busy repos (100–250 PRs a day each), p50 time-to-green fell **43%** in one repo and **34%** in the other, in about a day and a half. The top flake went to zero after it was fixed in product code rather than retried. Some of the largest wins were not speed at all: cache scoping (restore on PRs, save only on the default branch), memory pressure on build runners, shallow fetches, and timeouts on hung jobs. Several "obvious" wins turned out to be noise once they were measured in pairs.

## Install via GitHub MCP

1. Run [`automate-it`](../../skills/automate-it/SKILL.md). Start with a manual baseline: give [`experiment-arms`](../../skills/experiment-arms/SKILL.md) to an agent and ask for time-to-green p50/p90 and the critical path. Set the targets from it.
2. Create the label `ci-goal`. Fill PARAMETERS in [prompt.md](prompt.md) with `MODE: dry_run`.
3. Create both automations paused. Give the watch read tools plus `github/create_issue`, `github/add_issue_comment`, `github/update_issue_body` and `github/update_issue_labels`. Give the lane read tools plus `report_progress`, `create_pull_request`, `github/add_issue_comment`, `github/update_issue_body`, `github/update_issue_state`, `github/update_pull_request_title`, `github/update_pull_request_draft_state` and `retry_workflow_run`. Add `github/merge_pull_request` only with a standing grant. There's no cancel tool, so the lane never relies on cancelling a run; it pushes a new commit or waits. Set the lane's `issue_labeled` trigger to `labels: [ci-goal]` and `allow_copilot_triggers: true`, and give the lane a daily run limit.
4. Dry-run the watch on the last 7 days and check its numbers against `skills/experiment-arms/scripts/ci_arm_metrics.py`. Dry-run the lane on one goal issue, with arms capped at 1.
5. Enable on approval. Pair the lane with an [improver](../improver/README.md): `GOOD_OUTCOME: a kept arm held its gain after merge`, `BAD_OUTCOME: a kept arm regressed on real traffic or an owner reverted it`.
