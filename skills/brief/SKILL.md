---
name: brief
user-invocable: false
description: Turns a short intent, often around 20 words, into a brief that can run unattended. The brief has the user's words verbatim, an outcome owner, an observable done-when, a feedback loop, constraints and neighbors, a budget, an autonomy level, an absence window, stop conditions and a report contract. Runs only on `/lever brief`, when another lever skill delegates, or on an explicit request to write a brief or hand work off ("kick off", "spin up", "hand this to", "while I'm asleep", "I'll be away"). It files issues, so it never runs just because work is happening in parallel.
---

# Brief

The person with the intent should write a short sentence, and the agent should do the structuring. Experts type short kickoffs. What makes their delegation work is the brief compiled from that sentence. It says what done looks like, how the worker will know it is working, what it must not touch, how long it has, who lands the result and what to do while nobody is watching. This skill compiles that brief into [`templates/brief.md`](../../templates/brief.md). It explores the repo to fill fields instead of asking the human.

## When to use

- Handing work to anything that will run without you watching: a child session, a background agent, a coding agent on an issue, a scheduled run, or an automation prompt.
- Fanning out the same intent across N sessions, slices, models or hypotheses.
- Any kickoff longer than one sentence that will run for more than about 30 minutes.

**Do not use** it for a quick interactive edit you will watch, or when a brief for this work already exists. Amend the existing brief instead of writing a second one.

## Steps

1. **Quote the ask verbatim** under `## Ask (verbatim)`. Keep typos and casing. Never paraphrase the only copy. When briefs pass through several agents, the quote is what stops intent from drifting.
2. **Explore before asking.** Read the code, open issues and PRs, CI and existing instruments. Write down **what the code actually supports today** as a baseline line. Fill every field you can from the repo. The question budget defaults to 0. If one field truly needs the human, ask a single question that covers all of it.
3. **Outcome and owner.** Write one line for the outcome, and name the person or team who cares about the result. Name one owner per outcome. If ownership is split, write two briefs.
4. **Done-when you can observe.** State it as checks someone other than the author can run: a test passes, CI is green on the final commit, a metric crosses a threshold, or a screenshot or video shows a specific state. Name the [evidence level](../progress-contract/SKILL.md#evidence-levels) that counts as done. Merged is not the same as deployed or measured. "It works" is not a done-when.
5. **Feedback loop.** If the work is iterative, run `find-the-loop` and link or summarize its Loop Card. If there is no baseline, the worker's first step is to measure one.
6. **Constraints and neighbors.** List out-of-scope paths, read-only areas, real security limits, and the non-goals. Give every constraint a **source and expiry**, for example `(source: operator, expires: end of task)`. A constraint with no source isn't real, so drop it. For neighbors, list other lanes, PRs, sessions or people touching the same files, so the worker coordinates instead of colliding.
7. **Budget.** Take the [spend tier](../../docs/spend.md) and budget the router resolved (your tier, capped by the repo's `budgets:`), and which steps get a premium model. Set a timebox, a maximum number of iterations or attempts, a maximum PR size, and API and compute limits. For polling, use the minimum poll interval and API budget from `.github/lever.yml` `budgets:` when present. Also set the budget stop: what to do at 80% of the budget, which is to report and then finish or park.
8. **Autonomy level.** Resolve it with `autonomy-dial`. Record the level, the ceiling and where the level came from.
9. **Absence window and blocked policy.** Say when the human is away, for example "no human until 09:00 PT". Pick a blocked policy:
   - **decide-and-log:** make the call and record the assumption;
   - **park-and-next:** set the item aside and take the next one;
   - **ask-parent:** escalate to the coordinating agent, never the human, while they are absent.

   Human-only questions are surfaced once, with a recommended answer, in the next report. Delivery beats planning: do not produce status or plans in place of work.
10. **Stop conditions.** Write all three:
    - **success:** the done-when passes at the required evidence level;
    - **give-up or re-plan:** the budget is spent, or there has been no meaningful progress for the stuck threshold (default 2 h), in which case write a simpler plan, get a second-model critique and pick one;
    - **escalate:** the one decision only a human can make.
11. **Report contract.** Name who receives reports, the cadence, and the format, following `progress-contract` and [`templates/status-digest.md`](../../templates/status-digest.md). The default is silent unless something changed.
12. **Fan-out ("1 of N").** When there are several workers, write N sibling briefs that share the same Ask, rules, budget and report contract. Each brief adds a line `You are 1 of N. Your slice: <area | model | hypothesis>. Siblings: <slices>.` Slices must not overlap in the files they write. When comparing arms, every arm uses the same Loop Card.
13. **File it.** The brief lives on a GitHub issue in the target repo, so any session can resume it ([`docs/tracking.md`](../../docs/tracking.md)). Title it with the outcome. The body is the brief, then an empty `## Status` section and the `<!-- lever: … -->` marker. **Assign it to the outcome owner** (usually the user) and label it `lever`, creating the label if it's missing. More than one lane: one sub-issue per lane with its "1 of N" brief, also labelled `lever`, linked to the parent with `gh api repos/{o}/{r}/issues/{n}/sub_issues -F sub_issue_id=$(gh api repos/{o}/{r}/issues/{child} --jq .id)` (the child's `id`, not its number). Search `label:lever` first and amend an existing brief for the same outcome instead of filing a second one. No triage permission, or someone else's repo the user hasn't said to file in: keep the brief in the session and say so.
14. **Pick the runner.** Use `brief.runner` from `~/.copilot/lever.yml` if set. Otherwise end with one choice: **here** (this session coordinates now), **cloud** (assign each lane sub-issue to Copilot) or **later** (filed, nobody runs it yet). Then:
    - **here:** run `/lever deliver #<issue>` and keep it alive across turns: in the CLI, `/goal <done-when>` and `/every 30m /lever deliver #<issue>`; in the app, `/goal` and a session automation that runs `/lever deliver #<issue>` every 30 min. Each tick is a deliver sweep; `/lever status` stays read-only. Write `runner=here` and the coordinator into the marker.
    - **cloud:** assign each lane sub-issue to Copilot. Offer `/lever automate` for a paused Copilot Automation that shepherds the resulting PRs at the resolved autonomy level.
    - **later:** stop. Print `/lever deliver #<issue>` as the command to run when ready.

    In interactive mode, show the brief once before filing. In autopilot, file it and go.

## Example (abridged)

> **Ask (verbatim):** "ci on the app is way too slow. make it faster, I'm offline till morning. measured by wall clock"

The brief this produces:

- **Outcome:** PR time-to-green on the critical path. Owner: the app's maintainers.
- **Done when:** p50 TTG improves above the A/A floor across ≥6 paired runs. Evidence level: *measured*.
- **Loop:** the Loop Card from `find-the-loop` (CI row), with a baseline as the first step.
- **Constraints:**
  - Don't widen workflow `permissions:` (source: repo policy).
  - Don't touch `release/**` (source: operator, expires: end of task).
  - Neighbors: an open PR touching the caching steps.
- **Budget:** 10 h. At most 10 parallel arms. 75 API calls per merged PR.
- **Autonomy:** L1 do-not-merge arms, and winners handed off at L2 (ceiling L2: CODEOWNERS required). Absence window until 09:00 PT, decide-and-log.
- **Stop:**
  - **Success:** keep bar met.
  - **Give up:** 3 consecutive rejects per layer, or the budget is spent.
  - **Escalate:** none expected.
- **Report:** a morning digest, silent overnight unless something lands.

## Dials

| Dial | Default | Options |
|---|---|---|
| Question budget | 0. Explore, then ask one batched question only if blocked | 1–3 questions; full interview |
| Brief length | Full template | Terse: Ask, done-when, autonomy, stop |
| Autonomy level | Resolved by `autonomy-dial` | L0–L5 |
| Blocked policy | decide-and-log | park-and-next; ask-parent; ask-human (only when present) |
| Absence window | Asked once if the user signals absence | None; fixed hours; "until done" |
| Stuck threshold | 2 h without meaningful progress → re-plan | 30 min – 8 h |
| Budget stop | Report at 80%, then finish or park | 50–100% |
| Fan-out | 1 | N copies sliced by area, model or hypothesis |
| Report target and cadence | Back to the delegator, on change | Hourly; milestones; daily digest; issue comment |
| Approval | Interactive: show before filing. Autopilot: file and go | Always show; never show |
| Where it lives | An issue assigned to the owner, label `lever`; sub-issues per lane | Session only (no issue) |
| Runner | `brief.runner`, else ask once | here; cloud; later |

## Outputs

An issue labelled `lever` and assigned to the owner, with a filled `templates/brief.md` in its body and one sub-issue per lane. Sibling briefs share one Ask, and each carries its own "1 of N" line. A one-line summary goes back to the user: `#<issue> · <N> lane(s) · <outcome> · done when <check> · L<n> · runner <here|cloud|later>`.

## This skill's own feedback loop

- **Metric:** steering messages per delegated task, and the first-pass rate to done-when. Both should improve.
- **Retro:** when a worker asks a question, stalls, gets steered, or reports "done" and is wrong, name the brief field that was missing or vague. Append a gap line to the brief, and adjust this skill's defaults once the same gap appears three times.
- **Field coverage:** track how often each field is filled. Stop conditions are usually the weakest field, so check them first.

## Failure modes it prevents

- Intent drift when a paraphrase of a paraphrase reaches the third agent.
- "Done" that means "I stopped". Unobservable done-whens.
- Workers interrogating the human about things exploration could answer.
- Collisions with neighboring work. Constraints with no source that pile up and never expire.
- Overnight runs that stop at the first question nobody is awake to answer.
- Unbounded polling and API burn.

## Why this exists

- Across ~1.8k sessions, one developer's own kickoffs had a median of 18 words. Briefs that agents wrote for other agents had a median of 423. Across 418 pairs, the median expansion was 10.5×. The skill is in the compiling, not in long prompts.
- Over nine months, the share of kickoff briefs with acceptance criteria rose from 0% to 43%. Explicit stop conditions remained the weakest field at 14%.
- 67% of agent-written briefs carried the user's words verbatim, which kept intent stable across chains of agents.
- Plans with a baseline and a metric shipped. A +105k-line rewrite with an 8-phase dependency graph and no incremental loop stalled.
- Same-prompt fan-out worked as a method: "you are 1 of 20 parallel triage sessions" across a public issue tracker.
