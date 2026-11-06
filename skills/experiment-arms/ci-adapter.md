# CI adapter for experiment arms

This adapter covers the case where the metric is **PR time-to-green (TTG)** on GitHub Actions. The method itself lives in [`SKILL.md`](SKILL.md); this file holds only the CI-specific mechanics.

## Metric

- **TTG** runs from the first workflow run created for the head SHA to the completion of the last *required* check. Reruns count, because a person waiting on the PR waits through them too.
- **Critical lane** is measured from the first run created to the last job of the slowest required lane, using attempt 1 only.
- **Guards** are tracked alongside TTG: billed minutes per run (attempt 1, and all attempts), first-try green rate k/n, reruns per PR and rerun causes.
- **Billed minutes** = Σ ceil(job minutes) × runner multiplier. Leave out path-filtered workflows that were triggered only by the treatment's own diff, because they inflate the treatment's cost. A job on a runner label missing from `runner_multipliers`, or a run with no saved job snapshot, makes that run's cost *unknown*, never zero, and blocks any cost verdict until it is fixed.

## Arm mechanics

- **Point arms at the default branch.** CI then runs on the merge ref, which already contains current `main`, so no rebase is needed. A plain rerun reuses the old merge ref; close and reopen the PR to refresh it.
- **Stacked PRs often get no CI**, since many workflows only fire for PRs into `main`. Open a draft validation PR into `main` at the *exact head SHA*. The checks attach to the SHA and show up on the stacked PR as well.
- **Use a nonce, not an empty commit.** Add the same nonce line to one source file that busts the main compile cache *and* to the CI workflow file, in every arm and the control. Otherwise path filters skip lanes and the control gets warm caches the treatment can't get.
- **Reject skipped heavy lanes.** The driver marks an arm `invalid-skipped` if any `heavy` check ended in anything other than `success`.
- **Every required check must run.** An arm is `green` only when every check in `required` appeared and passed. A required check that never shows up keeps the arm pending until `timeout_min`, and then the arm is `invalid-missing`, never `green`. Every arm ends `green`, `red`, `invalid-skipped`, `invalid-missing` or `timeout`; only `green` counts as a measurement, and the rest count as failed attempts in `arm_stats.py`.
- **Control the cache.** If a shared compiler cache would let the control hit `main`'s entries, force re-caching in both arms (for sccache: `SCCACHE_RECACHE=1`).
- **Download artifacts before rerunning**, because a rerun drops them.
- **Retargeting or restacking cancels in-flight runs**, and those show as phantom reds. Re-run them before you report anything.
- **Watch for external drift.** If something outside the experiment changes, such as a package-registry deprecation that rewrites the lockfile, cherry-pick the fix into every arm.

## Before you blame capacity

Queue time is the gap between a job being created and starting. Subtract any time spent pending on a `concurrency:` group first. Check flake reruns before "busy hours": in one project the busy-hour slowness turned out to be reruns, not queueing.

## Seed hypotheses that often pay

Use these as starting rows for the hypothesis table, not as conclusions. Every one still has to be measured on your fleet.

| Hypothesis | Why it can pay |
|---|---|
| Restore-only caches on PRs; save only on `main` | PR saves evict `main`'s entries in a saturated cache |
| Audit SHA-keyed caches by their hit rate in the consuming job | many are never read back, yet dominate cache inflow |
| Treeless partial fetch (`--filter=tree:0`) instead of `fetch-depth: 0` | full history is rarely needed for merge-base or diff |
| Avoid `git log --follow` on partial clones | it lazily fetches every blob |
| Split coupled per-OS matrix jobs | a `needs:` on the matrix waits for the slowest leg |
| More test workers per runner before adding runners | cheaper than new runners; watch memory |
| Duration-balanced, longest-first shards | equalizes shard finish times |
| Grow the page file or swap on memory-bound builds | "invalid metadata" or "cache corruption" errors can be memory exhaustion |
| Job and step timeouts | the default timeout can hide hour-long hangs |
| Rerun only failed tests on a rerun attempt | faster flake recovery |
| Fix the top flake at its product root cause | flakes cost a full cycle each |

Never ship these, whatever they save: widened `permissions:`, `pull_request_target`, new secrets, triggering publish or release workflows, or any speed-up that weakens the signal (fewer tests, retries that hide failures).

## Scripts

- [`scripts/ci_arm_driver.py`](scripts/ci_arm_driver.py) pushes all arms together each round with an identical nonce, opens draft do-not-merge PRs, polls by SHA under the rate floor, reruns failed jobs as a person would, and appends results to `arms_<tag>.jsonl`. It can resume where it left off; create `<tag>.stop` to stop. It needs git 2.38 or newer. Each run pushes to its own branches, `lever/arm-<tag>-<run id>-<arm>` by default, and only ever rewrites branches it created, with an expected-old-SHA lease on every push; it refuses to touch a branch that someone else wrote to.
- [`scripts/ci_arm_metrics.py`](scripts/ci_arm_metrics.py) summarizes each arm's TTG, critical lane, billed minutes, first-try green rate and reruns, and writes `samples_<tag>.jsonl`.
- [`scripts/arm_stats.py`](scripts/arm_stats.py) is metric-agnostic. It reports paired Δ vs control, failed attempts per arm, wins per round, a bootstrap CI, Mann-Whitney U, the A/A noise floor, and a KEEP / DROP / INCONCLUSIVE verdict against the bar. A treatment that fails more rounds than the control is never kept. Lower is better by default; pass `--higher-is-better` for metrics like throughput.

Example config (`arms.json`):

```json
{
  "repo": "OWNER/REPO",
  "git_dir": "/path/to/a/worktree",
  "default_branch": "main",
  "tag": "r1",
  "arms": {"c": "measure-base", "c2": "measure-base", "cache": "arm-cache", "shards": "arm-shards"},
  "base_must_contain": "measure-base",
  "fix_prs": [],
  "nonce_files": {".github/workflows/ci.yml": "# arm-nonce {n}", "src/lib.rs": "// arm-nonce {n}"},
  "required": ["Build", "Test", "E2E"],
  "heavy": ["Build", "E2E"],
  "save_jobs_for_workflows": ["/ci.yml"],
  "rounds": 3, "reruns": 2, "poll_s": 240, "timeout_min": 150,
  "rate_floor": 3000, "max_arms": 10,
  "pr_title": "[measurement — do not merge] {tag} arm {arm}",
  "labels": ["do-not-merge"],
  "runner_multipliers": {"ubuntu-latest": 1, "windows-latest": 2, "macos-latest": 10}
}
```

Pointing `c` and `c2` at the same branch gives you an A/A pair in every round for free. Leave `branch_prefix` unset so each run gets its own branches; if you set it, the driver still refuses to overwrite a branch this run did not create.

```bash
python3 scripts/ci_arm_driver.py arms.json            # one driver per arm set
python3 scripts/ci_arm_metrics.py arms.json --lane-prefix "E2E"
python3 scripts/arm_stats.py samples_r1.jsonl --control c --aa c2 --bar 1.0
python3 scripts/arm_stats.py samples_r1.jsonl --control c --aa c2 --field billed_a1   # cost guard
```
