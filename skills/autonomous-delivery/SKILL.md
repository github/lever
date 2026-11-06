---
name: autonomous-delivery
user-invocable: false
description: Runs long-lived delivery as a coordinator with parallel lanes. Sets authority and pacing dials up front, keeps lanes alive and fed, lands work through a single-writer release train, and reports status only when something changed. Runs only on `/lever deliver`, "/autonomous-delivery" or an explicit request to run multi-hour, overnight or multi-lane delivery ("run this overnight", "keep N streams going", "don't stop until it ships"), or to re-arm an existing run after a restart.
---

# Autonomous delivery (v3)

**Organize the work, keep it moving, and keep improving how it is organized.** This is the coordination skill for [build mode](../../docs/modes.md): large new work where the process is to file issues, let independent sessions work them in parallel, then merge and deploy. There may be no metric to climb, and that's fine. When a lane does have a measurable signal, that lane runs [`find-the-loop`](../find-the-loop/SKILL.md). A coordinator holds your intent, splits it into lanes, and delegates each lane to a child session that owns an outcome. Children may run their own workers. The coordinator optimizes for **verified outcomes per hour**, not agent activity. Quality and throughput are joint goals, and you never buy speed by weakening the outcome. For a delivery mandate, the outcome is **working behavior observed on the real path**, not source code or an open PR.

A skill is not authorization. But existing delegation *is* authority within its scope, and it doesn't need to be requested again at every step.

Version 3 builds on the [public autonomous-delivery skill, v2.1](https://gist.github.com/EvanBoyle/8c1a97f682d92dab8c09d0e1ea73f4a0) (MIT). It keeps the adaptive orchestrator-worker pattern and adds the rules that weeks of real unattended runs forced into existence. Dials are set up front. Stalled children are restarted automatically. The watchdog metric is time since last merge. Wakes with nothing new stay silent. A stuck lane gets re-planned after two hours. Releases have one writer. The backlog never runs dry. Constraints that agents invent come with a source and an expiry.

Worked examples, templates and the full list of changes since v2.1 are in [`reference.md`](reference.md).

## When to use

- The work spans hours or days, or must keep running while you're away.
- There are more than about three independent units of work, or you said "parallelize", "lanes", "orchestrator", "keep N going".
- You're resuming a run after a restart, sleep or outage. Re-invoking re-arms it with the stored dials.

**Don't use it for:**
- A small or tightly coupled change. One agent is the right organization; just do it.
- Driving one PR to landed. Use [`pr-shepherd`](../pr-shepherd/SKILL.md).
- Turning an intent into a brief. Use [`brief`](../brief/SKILL.md) first and hand the result to this skill.

## Steps

### 0. Start from intent and current state (every session, and after long gaps)

Long-running coordinators fall behind. At session start and after any long gap:
- Refresh from the remote default branch and **re-read this skill** and the repo's runbooks, so you don't act on stale rules.
- Read the live state before acting: open issues and PRs, their discussion, existing owners, and what is actually deployed.
- Separate requirements from assumptions. An old checkout, a historical receipt or an idle session does not change the user's delegation.

### 1. Setup interview (once)

Ask only for the dials the prompt didn't already answer, in **one** message that takes under a minute. Offer a preset. Store the answers in the ledger header ([`templates/ledger.md`](../../templates/ledger.md)). Echo the contract back in one line, for example: `Overnight preset · L2 (your dial; ceiling L4) · standard spend · 3–5 lanes · refill when idle · sweep 30m · report 08:30 PT`. When re-invoked, read the stored dials and continue without asking again.

| Dial | `pair` (you're here) | `overnight` (you're away until a set time) | `unattended` (ongoing, no end time) |
|---|---|---|---|
| Autonomy | your dial (unchanged) | your dial (unchanged) | your dial (unchanged) |
| [Spend](../../docs/spend.md) | your tier | your tier; offer `max` for a deadline push | your tier, premium for judgment only |
| Lanes (min–max), capped by your tier | 1–3 | 3–5 | 5–8 |
| Idle policy | wait for events | refill from the backlog | refill from the backlog |
| Liveness sweep | off (events only) | every 30 min | every 15 min |
| Stall threshold (time since last merge) | report only | 2 h → act | 45 min in active hours → act |
| Stuck rule | 2 h → re-plan and ask | 2 h → re-plan with a rubber duck | 2 h → re-plan with a rubber duck |
| Status | at milestones | once, at your wake time | daily digest plus milestones |
| Human-only questions | ask now | queue for the morning report | queue for the daily digest |
| Budget stop | none | report at 80% | hard stop at 80%; won't start without a finite budget |
| Self-edit of this skill | off | propose | propose |

Presets change pacing, parallelism and spend. They never change autonomy. Lane counts and sweeps stay inside your spend tier's caps ([`docs/spend.md`](../../docs/spend.md)), and a preset never switches your tier without asking.

The autonomy level is resolved per [`docs/autonomy.md`](../../docs/autonomy.md): your latest explicit word (dial file, kickoff or chat), capped by `.github/lever.yml` and GitHub. If the prompt already said "then merge" or "hand off to the owning team", that is the answer. Don't ask again. Silence is not a grant. Without an explicit merge grant from you, work at L2 at most.

`unattended` has no end time, so it needs a finite budget before it starts: `budgets:` in `.github/lever.yml`, a cap in your spend dial, or a number you give in the setup answer. No budget, no start. Say so in one line.

**Owner directive format.** When the operator changes direction mid-run, record it as: `outcomes (priority, label rule, cap) · workers: N · merge authority: <who> · release owner: <who> · tracking: <issue> · until: <condition>`. A newer directive supersedes older ones and any repo doc that conflicts with it, except `.github/lever.yml`, rulesets and CODEOWNERS, which change only by PR. Fix a conflicting doc in a follow-up PR rather than obeying it.

### 2. Plan lanes, not documents

1. Turn the goal into outcomes, each with an observable done-when and a feedback loop (use [`find-the-loop`](../find-the-loop/SKILL.md) for any lane without one).
2. Build the dependency graph. Group units with no mutual dependency into a batch, and mark a batch ready only when its inputs exist.
3. Give each outcome **one owner** and a human-friendly milestone label (never a bare PR number or SHA). Treat shared files, APIs, fixtures and quotas as dependencies, even when the features sound independent.
4. Set the merge order up front: foundation first, leaves last, smallest reviewable PRs. Use [`landing-plan`](../landing-plan/SKILL.md) for batches.
5. A markdown plan is enough. Append superseding direction instead of rewriting history. Reconcile the graph after each batch.

**Fit the organization to the work.** A small fix or one tightly coupled investigation stays with one worker. Independent outcomes get separate owners. A large domain can get its own orchestrator when its local decisions would overload you. `max_lanes` is a cap, not a quota. Don't create agents to review each other just to satisfy process, and collapse layers that only relay messages.

**Parallel by default.** When the user raises a new outcome mid-stream, start a new dedicated owner for it right away and keep current work moving. Don't queue independent investigations, fixes or experiments behind a release or another lane unless they really share a resource. The user should never have to say "in parallel".

**Claim before you write.** Find or create a tracking issue for each outcome, labelled `lever` and assigned to the outcome owner, with lanes as sub-issues ([`docs/tracking.md`](../../docs/tracking.md)). Given `#issue`, resume from it: the brief, the lanes and the status section are the state. Write yourself into its marker as coordinator. If the marker names another coordinator, stay read-only and say who's driving. Take over only on an explicit handoff, or a stale marker per [`docs/tracking.md`](../../docs/tracking.md), and re-read the marker right before each write. If it isn't yours anymore, back off. Check assignees, linked PRs and active sessions before claiming. Assignment is not an atomic lock: if two owners claim the same thing, reconcile instead of starting a second writer.

### 3. Delegate with an envelope

Each child gets a standalone brief ([`templates/brief.md`](../../templates/brief.md); see the envelope in [`reference.md`](reference.md)) with the user's words verbatim, the outcome and its done-when, the feedback loop, the files it owns, its slice of the shared budget, and the inherited autonomy level. Its **escalation target is the parent, not the human**, and it runs in autopilot, never waiting on a human prompt.

**Every constraint carries provenance and an expiry:**

```
constraint | source (operator | repo | platform | agent:<lane>) | set | expires | why
```

- Operator, repo and platform constraints hold until they expire. Descendants inherit them and can only narrow them.
- An agent-sourced constraint applies only inside the subtree that created it. It expires when that assignment ends, or after `agent_constraint_ttl`. It never becomes a project-wide rule.
- A constraint with no source is not real. Drop it.

"Narrow, never widen" on its own drives authority toward zero down a three-level tree, because every layer adds a gate and nobody clears them.

**Handoff message.** A worker finishing a PR sends one line: `READY <pr> <head-sha>`. The receiver **re-runs the gate itself** on that head: open, not draft, mergeable, every required check green on that exact SHA, zero unresolved threads, and diff reviewed. It merges with an expected-head guard. A worker's claim is evidence level *claimed*, never *green*.

Model choice: give the coordinator strong reasoning, give well-scoped implementation fast capable workers, and save the strongest reasoning for hard diagnosis and stuck lanes. Respect your stated model preferences.

### 3a. Each lane runs the shortest evidence-driven loop

Start from a concrete hypothesis. Inspect the evidence, make the smallest coherent change, run focused checks and the required CI, land at the resolved level, exercise the real path, inspect the result, and iterate.

- **Keep verification proportionate.** For behavior changes, run focused tests, the required CI and a bounded real-path smoke test. For docs-only changes, check sources, links and consistency, plus required CI; don't redeploy. Don't make exhaustive scans, unchanged reruns or new verification machinery a prerequisite. Broaden only when the evidence demands it, and say what each check does and doesn't prove.
- **Check the diagnosis against raw evidence** before acting on it, and confirm a measurement script counts what it claims to.
- **Reproduce off production** before spending live attempts. A default-off flag in source may never have been turned on in the live system, so read the live configuration.
- **Treat a deploy as an experiment.** After a release or a flag flip, compare load, latency and error rates against the pre-change baseline right away. Roll back or adjust quickly when they regress.

### 4. Run the loop: events first, a sweep as backstop, silent when nothing changed

Handle child completions and GitHub events as they arrive: reconcile, unblock, route each merged SHA to the release owner, and start the next ready lane. When the sweep fires (`sweep_interval`, or a scheduled `/lever deliver #<issue>`), check the marker is still yours, then:

1. **Liveness, recursively.** Find every descendant that is interrupted or idle with unfinished work and **rekick it automatically**. The rekick message carries what changed and what's expected, never a bare "continue". The user should never have to ask for this. If a child doesn't move after two rekicks, apply the takeover protocol (step 9). If your host can resume children natively, use that; if it can't, file the gap as a product bug rather than writing more prompt. Judge liveness by the last turn or tool timestamp, not by a busy flag. A session can show busy with no turn running. **If a deploy or restart takes down the host your sessions run on, resuming yourself and your children is the last step of that deploy**, not something the human notices later.
2. **Watchdog metric: time since last merge.** Compare it to `stall_threshold`. If it's over, name the binding constraint (empty backlog, CI, review, quota, permission, a stuck lane) and act on it.
3. **Green but not landed.** Hand each such PR to `pr-shepherd`.
4. **Lane count below the minimum.** Refill (step 6).
5. **Budget burn.** Check API calls, CI minutes and credits against their caps.

Keep one short current-state checkpoint per lane: owner, outcome, evidence, blocker, next action. Store historical receipts separately; they are evidence, not current restrictions. Don't re-read unchanged ledgers, and don't ask owners for generic status when no decision depends on it. Busy doesn't prove progress, silence doesn't prove a stall, and idle doesn't mean complete or safe to archive.

If nothing changed since the last watermark, end the wake silently: no message, and no task invented to justify the wake. Never sleep or poll inside a turn. Give the waiting agent other work instead.

### 5. Land and release on a train

Per-PR landing belongs to `pr-shepherd`, at the resolved level. **After a merge, release and the next implementation run in parallel**: hand the exact merged SHA to the release owner and pick up the next ready work immediately. Don't serialize the team behind deployment, and don't create a second deployer. At L5, **one release-owner lane is the only writer of releases**. Other lanes send it merged SHAs. The release owner:
- ships the latest default-branch head whose **post-merge CI** is green (PR CI isn't enough), batching everything merged since the last release, not one release per PR;
- records any change between the selected target and a newer head, rather than silently swapping targets;
- deploys;
- runs a post-deploy check against a baseline;
- rolls back on regression.

Never redeploy unchanged source just to look busy, and never waive CI or compatibility checks to simulate continuous deployment. Use expand → migrate → contract for format changes. Use maintenance windows only for breaking changes or data migrations. Report merge→production latency.

**Pick the release target once per train.** Late retargets ("final target update", "revert", "ship whichever goes green first") cost a full re-plan each. Batch everything merged since the last release, and change target only for a regression.

### 6. Never idle (when `idle_policy = refill`)

"Nothing to do" is not a valid end state for an ongoing mandate. Keep `min_lanes` busy. When the ready pool runs low, draw from the standing backlog you agreed in setup:
- failing or flaky tests;
- audits against a reference;
- tool-failure and error-log triage;
- performance regressions;
- a backlog re-plan.

Run standing lanes **on change**, not on a timer. Revisit affected user journeys after meaningful UI or contract changes and releases, comparing them against agreed reference behavior: loading, error and empty states, accessibility, responsive layout. When you triage tool or error failures, keep the observation window and its limits: failure-only samples can't give you a failure rate, and missing logs don't prove success. Never replay a real mutation to test a hypothesis. Reuse existing findings, and don't re-audit unchanged surfaces.

You can also file new, independently mergeable work with [`keep-the-pool-full`](../keep-the-pool-full/SKILL.md). Refilling counts as a change, so it isn't a silent wake. For finite work with `idle_policy = wait`, stop when the outcome is verified.

### 7. Stuck rule: two hours without meaningful progress means re-plan

If a lane has gone `stuck_threshold` without meaningful progress:
1. Stop polishing the blocked approach.
2. Do a back-of-the-envelope check: at the current rate, when does this finish? If the answer is absurd, the plan is wrong.
3. Write a **simpler** recovery plan and rubber-duck it with a different strong model ([`cross-model-review`](../cross-model-review/SKILL.md)).
4. Pick one, record why in the ledger, and continue. Don't spend another two hours on the same approach.

### 8. Ask once, completely, then keep working

A question that truly needs a human states:
- what is blocked, and why;
- the exact URL or command;
- your recommended answer;
- what happens if the answer is no.

Do the work yourself through APIs and tools whenever a route exists. Never make manual testing, a human sign-in or a settings click a prerequisite when there's an automated way. If a setting really is UI-only, give the exact URL and paste-ready values once, then continue.

Ask it once. In `overnight` or `unattended` mode, put it in the next status report instead of interrupting. Keep working on everything that doesn't depend on the answer. An automatic "user unavailable" reply is not new evidence, so don't ask again. Never ask for permission the level already grants. Never invent an approval step, an exact-GO phrase, a lock or a preflight.

### 9. Takeover protocol (interrupted or orphaned work)

1. Find the work: the branch, PR, child session, or ledger entry.
2. Read the **live** state: head SHA, CI on that head, open threads, and any external write that timed out (its outcome is unknown, not failed).
3. Claim it explicitly in the ledger and on the PR or issue, so there is exactly one writer.
4. Continue from that evidence, not from memory or a stale plan. Never reset or rewrite another owner's ref. Preserve their work.
5. Stop or archive the previous owner once the handoff is confirmed.

When a worker fails permanently, start **exactly one** replacement per issue. It continues the same branch and PR after reading the predecessor's handoff. Never start a fresh branch, and never start two replacements.

### 10. Report through the progress contract

Use [`progress-contract`](../progress-contract/SKILL.md) and [`templates/status-digest.md`](../../templates/status-digest.md).
- Put the dials in force at the top.
- Report each milestone label with its state, ETA in your timezone, and evidence level (claimed < pushed < green < merged < deployed < measured).
- Show metrics as `x → y (Δz%)`, with merged count, time since last merge, and the previous ETA vs. actual.
- List human-only items.

Before you go offline, or at the end of the absence window:
- push all branches;
- put status and next steps in draft PR or issue bodies ([`templates/handoff.md`](../../templates/handoff.md));
- make sure every active lane has an issue.

### 11. Reflect, and remove the process you invented

At meaningful points (after a batch lands, after a stall, at each digest), ask whether this organization is producing verified outcomes faster. Then:
- **Remove the process you invented.** List the active constraints. Drop agent-sourced ones that have expired or have no source and that block the goal. Say so in the next status.
- Compare ETA vs. actual, and stall minutes. Make one workflow change at a time (split, merge or re-scope lanes; change concurrency or model tier) and keep it or revert it on evidence.
- When a correction repeats, route it to a durable layer with [`steer-once`](../steer-once/SKILL.md). Don't just apologise and carry on.
- When several variants of a prompt, definition or config are plausible, compare them as paired arms against a control ([`experiment-arms`](../experiment-arms/SKILL.md)), not one before/after change at a time. Search on replayed or seeded tasks, then confirm the winner on live work.
- **Evolve this skill through review.** After meaningful lessons, revise this SKILL.md through an issue and a reviewed PR, with the evidence in the PR. Update or remove the rule; don't append incident history. Keep SHAs, session IDs and live state out of it. Retire guidance that no longer earns its cost.

### 12. Move lanes to the cloud, one at a time

The long-term direction is for repeatable lanes (review, CI repair, issue repro, status, next-work pickup) to run as Copilot Automations, while the local coordinator moves toward supervision and exceptions. You don't need a hierarchy or cross-session messaging first. Start with independent jobs that read durable inputs (issues, PRs, artifacts) and write durable outputs (reviews, labels, PRs). Use [`automate-it`](../automate-it/SKILL.md) for each one.

Move **one bounded lane at a time**, under explicit ownership:
1. Establish what the hosted run can actually do: tools, permissions, trigger, durable checkpoint, acceptance evidence, and the escalation path.
2. Qualify it on observed hosted outputs. Local and hosted workers must never compete for the same issue, PR, queue or release.
3. Record the cutover and stop the local predecessor's writes. Keep release as one serialized lane across both places.
4. Declare it migrated only after it has produced outputs **and** kept picking up next work from events, with no local prompting, over an agreed observation window. One good run isn't a migration.

Rules:
- One improver with a global view owns changes to automation definitions. Other lanes propose; they don't rewrite themselves.
- A sent message isn't proof the next owner acted. Confirm delivery *and* consumption before depending on a handoff.
- Treat automation-wide stalls (lanes idle, triggers unmatched, schedules skipped) as delivery outages, not background noise.
- Judge the move by **valuable PRs landed through it**, not by the number of lanes or runs.

## Dials

| Dial | Default | Options |
|---|---|---|
| `preset` | `pair` | `pair`, `overnight`, `unattended`, custom |
| `autonomy` | Resolved per [`docs/autonomy.md`](../../docs/autonomy.md) | L0–L5, capped by the repo ceiling |
| `min_lanes` / `max_lanes` | 1 / 3 | integers; depth ≤ 3 |
| `idle_policy` | `wait` | `wait`, `refill`, `stop` |
| `standing_backlog` | none | list of recurring lane types |
| `sweep_interval` | off | minutes |
| `stall_threshold` | 2 h, report only | duration; report or act |
| `stuck_threshold` | 2 h | duration; escalation models |
| `agent_constraint_ttl` | end of assignment | duration |
| `absence_window` | none | until a time, N hours, or indefinite |
| `status` | milestones; your timezone | cadence, horizon, channel |
| `human_questions` | ask now | ask now, queue for the next report |
| `budget` | `budgets:` in your repo's `.github/lever.yml`, then your spend tier | API calls per merged PR, CI minutes, credits; stop at a percentage. Required for `unattended` |
| `release` | none | none, train (single writer), per merge |
| `self_edit` | off | off, propose, commit (through the level) |

## Outputs

- **Ledger:** the dials header, current state first, owners, decisions with their constraint provenance, and evidence links. Rotate it when it grows.
- **GitHub artifacts** a person can resume from: an issue per lane, PRs with status in the body, and flake issues.
- **Status digests** (silent when unchanged) and a morning or end-of-run report.

## This skill's own feedback loop

- **Watchdog:** time since last merge, and stall minutes per night. Target: no window over `stall_threshold` without a named cause.
- **Throughput:** merges per hour, and merge→production median.
- **Human load:** rekick requests (target 0), "why did you stop" nudges (target 0), and status asks as a share of your messages.
- **Forecast quality:** ETA error per digest.
- **Bureaucracy:** gate and waiting language in agent messages.

## Failure modes it prevents

- Zero merges overnight because nobody noticed a stall.
- You restarting children by hand.
- Coordinators that stop at a milestone.
- Polling chatter on every wake.
- Two hours polishing a dead approach.
- Concurrent releases from several lanes.
- Authority eroded by inherited gates nobody remembers adding.
- Questions to an absent human that block every lane.

## Boundaries that adaptation never changes

- **Authority:** a skill grants no permissions. Act within the resolved level, the host's rules, and the budget. A planning request is not permission to merge, deploy, spend money or set up a recurring schedule.
- **Data:** don't send private code, credentials, logs or conversations anywhere unauthorized. Never borrow credentials from another session. Idle doesn't mean disposable, so preserve checkouts and saved work.
- **Effects:** keep one owner for each shared mutable resource. Before retrying a write that timed out, reconcile what actually happened.
- **Evidence:** match each claim to the evidence level you actually reached. Report what failed and what you couldn't check.

## Why this exists

- What's possible: a single coordinator can run for **days to weeks**, steering **10s to 100s** of child sessions through **1,000s** of cross-session messages and scheduled wakeups, with a human typing a **low single-digit %** of the turns. At that scale, status or ETA requests become the largest category of what the human types, which is why `progress-contract` exists.
- The human was the watchdog. Morning messages read "no merged commits in 19 hours" and "nothing merged for 8 hours", and there were at least 7 "rekick your children" requests. Hours since last merge was computed by hand for two weeks before it became a sweep.
- Agents invented approval gates, preflights and locks, and then waited on them. After about 200 of the user's own corrections were compressed into this skill's rules, "why did you stop" nudges went from 27 to 0, and gate language in agent messages from 31% to 3%.
- Merge authority changed four times in two weeks, depending on repo and task. That's why it is a dial.
- "If we get stuck for more than two hours, create and rubber-duck a plan" came out of a migration that dragged on for two days.
