---
name: prompt-mirror
user-invocable: false
description: Mine your own local Copilot session history and return a scorecard of how you direct agents, plus three suggestions illustrated with your own before/after messages. It reads your private history, so it runs only on `/lever mirror` or an explicit ask ("how am I prompting?", "coach me", "mine my sessions", "what do I keep repeating?", a weekly self-retro).
---

# prompt-mirror

A coach that reads only your own session history. The extraction runs on your machine. It separates what you typed from the far larger volume of agent, autopilot and scheduled traffic, scores how you direct agents (length, steering, polling, repeated corrections, missing done-when and stop conditions, stuck loops, fan-out), and returns three concrete suggestions using your own messages as the before. It may draft instruction or skill changes; it never applies them. Applying is [`steer-once`](../steer-once/SKILL.md)'s job.

## When to use

- On demand: "how am I prompting", "coach me", "what do I keep correcting", "self-retro".
- Weekly, as a **local** session automation on your own machine.
- After a run where you typed "still broken", "status?" or "why did you stop" more than twice.

Do not use it to evaluate someone else, to rank teammates, or from a cloud automation. Cloud runners cannot and should not read local session state.

## Privacy rules (non-negotiable defaults; each is also a dial)

1. **Extraction is local. Analysis goes through your model.** The script reads `~/.copilot` (or `$COPILOT_HOME`) and writes to `~/.copilot/prompt-mirror/<date>/` with `0700`/`0600` permissions, and nothing it writes is uploaded. But any redacted sample Copilot reads to score you goes to your configured model service, same as anything else in chat. So before reading message text, ask once: "Read redacted samples (they go to your model service), or numbers only?" Numbers only means `--no-text`: build the scorecard from counts, and skip the sample reads in steps 3 and 5 and the before/after quotes. Never paste history into an issue, PR, gist, cloud agent or automation.
2. **Quote sparingly.** At most one quote per suggestion, at most 25 words, redacted, and only from the "before" side.
3. **Never quote** messages flagged `heated` or `sensitive` (profanity, HR, performance reviews, compensation, health, legal). Use aggregate counts only.
4. **Redact before display:** secrets and tokens, emails, URLs, home paths, `@handles`, UUIDs and long hex, plus every term in the user's `--redact-terms` file (people, codenames, internal hosts). Never show session IDs beyond an 8-character prefix, and never in anything shared.
5. Anything leaving the machine (for example a steer-once PR) is re-shown to the user first, as the exact text that will be published.

## Where the data lives

| Source | Format | Use |
|---|---|---|
| `~/.copilot/session-state/<id>/events.jsonl` | One JSON event per line: `{type, id, parentId, timestamp, data, agentId?}` | High confidence. Has source tags and delivery mode. |
| `~/.copilot/session-store.db` | SQLite. `sessions(id, cwd, repository, branch, ...)`, `turns(session_id, turn_index, user_message, assistant_response, timestamp)` | Medium confidence. Backfills sessions whose `events.jsonl` is gone. No source tag. |

How to tell your words from machine traffic in `events.jsonl` (`type == "user.message"`):

| Signal | Meaning |
|---|---|
| `data.source` absent (older builds: `"user"`) | Candidate typed message |
| `data.source == "agent-<id>"` | Cross-session message from another agent |
| `data.source == "autopilot"` or `data.isAutopilotContinuation` | Autopilot continuation |
| `data.source == "system"` | Scheduled wake or automation prompt |
| top-level `agentId` present | Prompt to a subagent |
| no source, but `data.transformedContent` contains `<cross_session_message` | Kickoff brief written by a coordinator for a child session |
| `data.content` is `[ALL CAPS REQUEST] ...`, or "Continue from where you left off." | App-generated template or resume |
| `data.delivery` = `idle` / `steering` / `queued` | Sent while the agent was idle, mid-turn, or queued |

Also: `abort` events with `data.reason == "user_initiated"`; `session.start` → `data.context.cwd` / `repository`. Use `data.content` (what was typed), not `transformedContent` (adds injected context). Store rows need heuristics: strip injected `<system_*>`, `<skill-context>` and similar blocks, drop envelopes and templates, and drop long third-person briefs ("User request verbatim: ..."). Drop sessions whose cwd is under `/tmp` or `/var/folders`. A 60+ character prompt whose opening recurs in 4+ sessions is usually an eval harness or pasted fan-out, so drop it when the source is uncertain. If it's clearly you typing it (the same morning prompt every day), keep it: that's an automation candidate, not noise. If the format has drifted, inspect key names on a few events (never print content) and update the filters before scoring.

## Steps

1. **Set the dials** (table below) in one line back to the user: window, scope, tone, output. Ask only if the user named a scope you cannot infer.
2. **Extract.** Run the bundled stdlib script:
   ```bash
   python3 <this-skill-dir>/scripts/mine_sessions.py --since 30d [--repo owner/name] [--redact-terms ~/.config/prompt-mirror/terms.txt]
   ```
   It writes `stats.json`, `messages.jsonl` and `messages.csv` (redacted text, flags, correction classes). Use `--no-text` for numbers only: no message text and no text-derived phrase stats. If an agent tool can query the session store directly, apply the same filters.
3. **Sanity-check the extraction before scoring.** Typed count and sessions plausible? `machine_per_typed` usually ≫ 1 for heavy agent users. If the user allowed samples, read 20 random redacted messages: if any read like an agent brief or report, tighten filters and re-run. Never score unfiltered history; you would be measuring your agents.
4. **Build the scorecard** from `stats.json`, compared with the previous run if one exists:

   | Metric | Field | Healthy direction |
   |---|---|---|
   | Median / p90 typed length | `typed.words` | Context, not a target |
   | Mid-turn steering share | `delivery.steering_pct` | Context; high is fine if outcomes are stated |
   | Status-ask rate | `rates_pct_of_typed.status_ask` | Down; each ask is a poll you ran by hand |
   | Repeated corrections | `correction_classes`, `repeated_correction_phrases` (≥3 sessions) | Down per class |
   | Kickoffs missing done-when | `100 − kickoffs.has_done_when_pct` | Down |
   | Kickoffs missing a stop condition | `100 − kickoffs.has_stop_condition_pct` | Down |
   | Stuck without measurement (loop smells) | `loop_smells`, `rates_pct_of_typed.stuck` vs `rates_pct_of_typed.measurement` | Fewer sessions |
   | Fan-out share | `rates_pct_of_typed.fanout`, `kickoffs.fanout_pct` | Context |
   | Same prompt typed again and again | `automation_candidates` (rows flagged `repeated_prompt`) | Each one is an `/lever automate` candidate |
   | Agent concessions ("You're right…") | `assistant.concessions_per_100` | A concession without a durable fix predicts a repeat |
   | Machine messages per typed message | `machine_per_typed` | Context: how much work follows each message |

5. **Validate each headline signal by reading.** The classifiers are regexes. For every metric you will cite, open at least 5 flagged messages and discard false positives; report the corrected number and say it was hand-checked.
6. **Pick three suggestions,** ranked by frequency × cost (time lost, sessions affected). Each one has:
   - the signal and its number (for example `status asks: 14% of typed messages, 212 in 9 sessions`);
   - **before:** one of your own messages (rules above);
   - **after:** a rewrite, or the brief field or skill that removes the need for the message;
   - the lever piece that fixes it, for example status polling → [`progress-contract`](../progress-contract/SKILL.md); no done-when/stop → [`brief`](../brief/SKILL.md); stuck ×2 with no measurement → [`find-the-loop`](../find-the-loop/SKILL.md); same correction in ≥3 sessions → [`steer-once`](../steer-once/SKILL.md); same prompt typed daily → [`automate-it`](../automate-it/SKILL.md);
   - the metric that should move next run.
7. **Optionally draft** instruction or skill text for the top repeated correction, as a diff in the report. Do not write to any instruction, skill or config file. Offer `steer-once` to land it.
8. **Save** `report.md` and `suggestions.json` (suggestion, metric, baseline, date) next to the extraction. Show the report in chat.
9. **On the next run,** load the last `suggestions.json` and report each metric's change. Call a change real only if it exceeds the week-to-week spread of the previous 4 weeks (`weekly`); otherwise say "within noise".

## Dials

| Dial | Default | Options |
|---|---|---|
| Window | 30 days | any `--since/--until` |
| Scope | all local sessions | `--repo` substring |
| Redaction | strict + user term file | strict; `--keep-heated` / `--keep-sensitive` for the user's eyes only |
| Quoting | ≤1 quote per suggestion, ≤25 words | counts only (`--no-text`) |
| Text sent to the model | ask once before reading samples | numbers only (`--no-text`); redacted samples |
| Tone | direct, specific, no scolding | gentle; blunt |
| Output | report + drafted diffs | report only |
| Comparison | your own trend only | none; opt-in anonymous team aggregate (counts only, never text) |
| Cadence | on demand | weekly local session automation |
| Retention | keep last 8 runs | delete after report (`rm -rf ~/.copilot/prompt-mirror`) |

## Outputs

- `~/.copilot/prompt-mirror/<date>/stats.json`, `messages.jsonl`, `messages.csv`: local, redacted, private.
- `report.md`: scorecard table, three suggestions with before/after, drafted diffs, trend versus last run.
- `suggestions.json`: what was suggested and which metric should move.

## This skill's own feedback loop

Every suggestion names a metric and a baseline. Next run, the skill reports the fraction of last run's suggestions whose metric moved beyond noise. If that falls below one in three across two runs, the suggestions are generic; ground them harder in the user's repeated phrases and loop smells. Secondary: user status asks per day and repeated-correction rate, trending down.

## Failure modes it prevents

- Measuring your agents instead of you (machine traffic is often 15–20× the typed volume).
- Generic prompting advice; every suggestion carries your own before.
- Polish-anxiety: length and typos are context, not faults. Outcome, evidence and boundaries matter.
- Private text leaking into a shared artifact.
- Trusting a regex rate you never read.

## Why this exists

- This suite was built this way: one heavy user's ~9.5k typed messages across ~1.8k sessions were mined to find what to encode. Separating typed messages from agent traffic was the first and hardest step; in one month each typed message produced about 21 machine messages.
- Kickoffs had a median of 18 words. Only 6% stated a done-when, and stop conditions were the weakest field of agent-written briefs at 14%.
- As autonomy grew, status and ETA asks rose from about 2% to 13% of typed messages: polling done by hand. 40% of messages landed mid-turn.
- When agents were stuck, the effective move was "build a feedback loop" (23 explicit rescues), not more instructions. The mirror flags "stuck twice, no measurement" so you make that move sooner.
