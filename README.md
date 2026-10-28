# lever

**Feedback loops for coding agents.** A Copilot plugin.

![/lever <verb> <your words>: the verb picks the playbook, the rest is you talking normally](docs/examples/try.png)

```bash
copilot plugin marketplace add github/lever
copilot plugin install lever@lever
```

In the Copilot app, open these links. Each one asks you to confirm, then opens a pre-filled form:

1. [Add the marketplace](ghapp://plugins/marketplace/add?source=github%2Flever) (`github/lever`)
2. [Install the plugin](ghapp://plugins/install?source=lever%40lever) (`lever@lever`)

Or do it by hand: **Customize → + Add → Marketplace** with `github/lever`, then **+ Add → Plugin** with `lever@lever`.

Start a new session and try one that only reads:

```text
/lever loop our test suite feels slow. just the card, don't change anything
```

You get a Loop Card back: the metric, how to measure it, how noisy it is, and what counts as a win. Nothing gets pushed, filed or merged.

## Why

Agents get really good once they can check their own work. Give one a number, a way to measure it and a bar for "better", and it'll grind on that for hours without you. Without that, you're the loop: pasting screenshots, eyeballing scroll jank, hitting rerun on CI at midnight.

Setting up the loop is the hard part. What do you measure? How noisy is it? What's a real win and what's a lucky run? When do you stop? Most people skip it, which is fair, because it's annoying. lever asks you those questions, writes the answers down, and then lets agents iterate against it.

It comes from how some of the most AI-forward engineers at GitHub work day to day: CI speedups, a five-day crate split run from three terminals, weeks of [unattended delivery](https://gist.github.com/EvanBoyle/8c1a97f682d92dab8c09d0e1ea73f4a0), and the automations that took over the chores. We mined our own sessions for what kept working, then put most of the effort into making it one command. Same workflow, no PhD required.

## What it looks like

![/lever routes a plain-words ask to the right verb](docs/examples/lever.png)

![/lever loop turns "it feels slow" into a Loop Card](docs/examples/loop.png)

![/lever autonomy shows and sets your dial](docs/examples/autonomy.png)

More in the **[examples gallery](docs/examples.md)**: experiment arms, a brief, the three-terminal refactor, the PR shepherd, cross-model review, an automation, and the verbs that tune lever to you.

<details><summary>Copy-pasteable examples</summary>

```text
/lever our CI is slow and I'm tired of hitting rerun
/lever loop the editor feels janky when I scroll big files
/lever arms try the 6 ideas on the card
/lever brief add offline mode to the editor. ship it by tomorrow
/lever refactor src/core is 400k lines. cut it in half
/lever shepherd keep my stack green till it lands
/lever review this PR
/lever automate make the stack shepherd an automation for this repo
/lever mirror last 30 days
/lever steer push immediately, then verify
/lever autonomy agents can merge docs PRs in this repo, nothing else
```

</details>

## Commands

```text
/lever <what's bugging you>      not sure? describe it. lever picks the verb.

BUILD   make something new
  brief <ask>                    turn ~20 words into a brief an agent can run unattended
  deliver [brief|issues]         run lanes in parallel and land the work
  refactor <scope> <goal>        huge mechanical change: producer, packager, shepherd
  backlog                        keep enough ready issues in the pool
  shepherd [pr|stack]            keep PRs green until they land
  land [prs]                     merge plan for a stack or a batch
  status                         progress you never have to ask for

LOOP    make something measurably better
  loop <problem>                 find the feedback loop first: a Loop Card
  arms [ideas]                   one agent per idea, against a control
  review [pr|plan]               several models, blind; only agreed findings

RUN     make it happen without you
  automate <task>                turn a chore into a paused Copilot Automation
  health                         why did it stall? one fix, or "no change"

YOU     tune lever to how you work
  mirror [30d]                   a scorecard of how you prompt, from your sessions
  steer <correction>             say it once, never again
  autonomy [L0-L5|<words>]       who merges. see or set your dial
  spend [standard|high|max]      how much model to throw at it
  setup                          first run: set your dials
  help [verb]                    this list, or one verb in detail
```

Each verb is a skill in [`skills/`](skills/). The `/` menu shows only `/lever`. Skills that only read and advise, like `loop`, can still load on their own when your ask fits. Anything that pushes, files, opens PRs or schedules runs waits for `/lever <verb>` or an explicit ask.

## Your dials

- **[Autonomy](docs/autonomy.md)**, L0 to L5: who merges. L0 only proposes. L2 gets PRs green and you merge. L5 lands, releases and watches. It's your setting, saved in `~/.copilot/lever.yml`, and your repo's branch protection still wins. Start at L2 and turn it up as you trust it.
- **[Spend](docs/spend.md)**, `standard` · `high` · `max`: model tier, parallel lanes, review depth and how often things run. A repo can cap it in `.github/lever.yml`.

`/lever setup` asks three questions and writes both ([example](config/personal.example.yml)). For a team, copy [`config/autonomy.example.yml`](config/autonomy.example.yml) to `.github/lever.yml`.

Work lever starts is tracked as an issue assigned to you with the [`lever` label](docs/tracking.md), so `/lever status` can list everything in flight, needs-you first.

## Other ways to install

**Per repo**, so the Copilot coding agent in the cloud gets the skills too:

```bash
mkdir -p .github/skills
cp -r path/to/lever/skills/* .github/skills/
cp -r path/to/lever/{docs,templates,config,automations} .github/
```

Skills link to `../../docs/...`, so those four folders have to sit next to `skills/`.

**Hacking on lever:** clone it and run `copilot plugin marketplace add ./lever`, then `copilot plugin install lever@lever`. It loads straight from your checkout, so edits show up in your next session.

**Automations:** the recipes in [`automations/`](automations/) install through the GitHub MCP server with `/lever automate`. Each one starts paused, dry-runs on past events, and only then gets turned on. The Copilot app has the automation tools built in; in Copilot CLI, start with `copilot --enable-all-github-mcp-tools`.

## Make it yours

These skills fit how we work. Yours will be different. `/lever mirror` reads your own session history and shows you what you keep repeating, and `/lever steer` turns those repeated corrections into rules that stick. Fork it and bend it.

## Status

Early. Issues and PRs welcome.

## License

MIT. See [LICENSE](LICENSE).
