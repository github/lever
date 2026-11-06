# Spend dial

How many tokens and how much compute you are willing to spend is a choice, like autonomy. The tiers use the same names as Copilot Fusion: `standard`, `high` and `max`. A `standard` setup can still close loops. A `max` setup buys parallelism and judgment, not correctness for free.

| | `standard` | `high` (default) | `max` |
|---|---|---|---|
| Model tier | Fusion Standard or standard models everywhere | Fusion High or a premium model, only for judgment: diagnosis, planning, risky review | Fusion Max or a premium model by default |
| Parallel lanes | 1–2 | 3–5 | 5–20+ |
| Review | On request | One reviewer per PR, on events | Multi-model panel on risky diffs, re-review on push |
| Paired runs per arm | 6 (fewer arms, not fewer runs) | 6–8 | 6–12 |
| Triggers | Events with filters only | Events plus an hourly sweep | Events plus a 15-min sweep |
| Budget stop | Hard stop at 80% | Report at 80%, then finish or park | Report at 80% |

## Order of magnitude

Costs multiply. Estimate before you pick:

- **Model tier:** a premium model costs about **10–15× per call** compared with a standard one. In practice, cost follows the model more than anything else.
- **Lanes:** N parallel lanes cost about **N×**. Coordination overhead grows as well; watch for lanes that only relay messages.
- **Triggers:** an unfiltered per-push trigger can mean **~2–3× runs per PR** with no added signal. Triggers decide cost.
- **Review panels:** k reviewing models cost about **k×** per review.
- **Polling:** status checks and polling can quietly be the largest line item. Prefer events; a no-change wake should end silently and cheaply.

## Where spend pays off

- **Spend on judgment:** diagnosis, planning a fan-out, reviewing a risky change, deciding whether a measured win is real.
- **Save on routine:** classification, labeling, digests, no-op checks. A run that correctly does nothing should cost almost nothing.
- **Measure value per spend:** report useful outputs per dollar, or for a CI loop, Actions minutes spent per minute saved. Raise spend where that ratio is good, and cut it where it isn't.

`standard` buys fewer arms, not weaker evidence.

## Setting it

Your tier is personal, like your autonomy dial. It lives in `~/.copilot/lever.yml` ([example](../config/personal.example.yml)) and you change it with `/lever spend <tier>`:

```yaml
spend:
  tier: high                # standard | high | max
  repos:
    you/side-project: max
```

The repo caps it. `.github/lever.yml` ([example](../config/autonomy.example.yml)) can set a `spend.tier` for people with no dial, a `daily_cap_usd` and `budgets:` (API calls per merged PR, max parallel arms, poll interval):

```yaml
spend:
  tier: high                # default for people with no dial
  premium_for: [diagnosis, planning, risky_review]
  daily_cap_usd: 50         # hard stop
budgets:
  github_api_calls_per_merged_pr: 75
  max_parallel_arms: 10
```

Precedence: your tier (or one run's kickoff answer or automation PARAMETERS), capped by the repo's `budgets:` and `daily_cap_usd`. The `/lever` router resolves this once, before every verb, and hands the skill one budget. A skill's own defaults (lanes, arms, sweeps) only apply inside it. Runs with no end time (`unattended`, a refill loop) need a finite budget from one of these places, or they don't start.
