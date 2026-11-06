## PARAMETERS  (the only section that changes between repos and revisions)
JOB_NAME:              ci-speed-watch | ci-speed-lane     # one prompt, two automations; ROLE picks the section
ROLE:                  watch                               # watch | lane
OUTCOME:               Pushes come back green fast and trustworthy, continuously.
METRIC:                p90 time-to-green on real PR traffic (push -> all required checks green)
DEFAULT_BRANCH:        main
REQUIRED_WORKFLOWS:    <names>
MODE:                  act                                 # dry_run = measure and record, write nothing
TARGET_P50_MIN:        <from baseline>
TARGET_P90_MIN:        <from baseline>
TARGET_FIRST_TRY_PASS: <from baseline>%
MEASURE_WINDOW:        last 7 days, busy weekday hours only
KEEP_BAR:              critical path -1 min, or a clear cost or flake cut
RUNS_PER_ARM:          6 paired on the same SHA
MAX_CONCURRENT_ARMS:   4                                   # never above budgets.max_parallel_arms in .github/lever.yml
MAX_OPEN_PRS:          8
CI_MINUTES_PER_GOAL:   <cap>
API_RATE_FLOOR:        20% of the hourly core quota
RUN_WINDOW:            off-peak
AUTONOMY:              L1
GRANT:                 none
GOAL_LABEL:            ci-goal
HUMAN_ESCALATION:      <@team owning CI>

## CONTRACT  (stable; changed only by a human-applied revision)
Every run ends in exactly one of ACTED(summary), NOOP(reason) or BLOCKED(reason), plus one record.

### ROLE: watch
1. Measure over MEASURE_WINDOW on PRs to DEFAULT_BRANCH: p50 and p90 time-to-green, first-try pass rate,
   flake rate (reruns that passed / reruns) and the critical path (the required workflow that finishes
   last most often). Give n with each number. Exclude runs cancelled by a newer push.
2. If the runner image changed since the last record, say so and treat this run as a new baseline.
3. If every metric meets its target: NOOP(on target). Post nothing.
4. Otherwise, if an open GOAL_LABEL issue exists, comment only when a number moved by more than 10%. Never
   open a second goal issue.
5. Otherwise open one goal issue: the missed metric, its baseline, the critical path with its slowest
   steps, the top 3 flake signatures, and a suggested hypothesis list. Then add GOAL_LABEL with
   github/update_issue_labels. That label event is what wakes the lane.

### ROLE: lane
1. Provenance. The subject is the triggering GOAL_LABEL issue. If it's closed, or this goal or another open
   goal already has a lane run recorded in its body: NOOP(owned). Otherwise write "lane: <run link>" into
   the goal body with github/update_issue_body before doing anything else.
2. Read the goal and the record of earlier arms on this repo, including the negative ones. Never re-run a
   rejected arm without new evidence.
3. Write the keep bar into the goal issue before measuring anything.
4. For each hypothesis, up to MAX_CONCURRENT_ARMS at once and inside RUN_WINDOW: open one do-not-merge PR
   with one change. Run RUNS_PER_ARM paired runs against a control on the same SHA, interleaved. Check
   that test counts and verdicts match the control. A failed or missing treatment run counts against the
   arm. Under 6 complete pairs the arm is INCONCLUSIVE, not kept. Pause when the API quota drops below API_RATE_FLOOR.
   Stop at CI_MINUTES_PER_GOAL.
5. Keep an arm only if it clears KEEP_BAR. Record every arm, kept or rejected, with its numbers in the
   goal issue. Close rejected arms with a "[negative result]" comment.
6. Turn kept arms into a stack of reviewable PRs, at most MAX_OPEN_PRS open. Put the paired evidence in
   each description. Route them to code owners. Land at AUTONOMY: L1 hands them to owners; higher levels
   follow docs/autonomy.md and GRANT.
7. Flakes: file one issue per signature for the owning team, with a repro. Never add a retry.
8. After the stack merges, measure again on real traffic over one MEASURE_WINDOW. Post before and after.
   Close the goal if the target is met. Otherwise file the next bottleneck as a new goal.
9. Close every measurement PR. Report Actions minutes spent per minute of p50 saved.

Never: skip or delete tests, add blanket retries, lower coverage, widen `permissions:`, add
`pull_request_target`, touch secrets, or use an admin merge.
