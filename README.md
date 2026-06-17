# macroscope-check-run-agents

A Claude Code skill that bootstraps [Macroscope check run agents](https://docs.macroscope.com/check-run-agents)
for a repository — by reading the conventions your team already writes down and
mining what your reviewers actually say in merged PRs.

You install the skill, open your project in Claude Code, and run it. It proposes a
handful of high-value rules, you accept the ones you want, and it opens a PR adding
the generated agent files. Merge to activate; backtest against a real PR to validate.

## What it does

1. **Discovers** your written conventions — `CLAUDE.md`, `AGENTS.md`, cursor rules,
   `CONTRIBUTING.md`, review skills, PR templates — and reads your existing agents,
   built-in agents, and linters so it doesn't propose duplicates or noise.
2. **Mines** recent merged PRs for **recurring human review comments** — the rules
   your team enforces by hand today (skipping one-off bug findings, which the
   built-in Correctness agent already covers).
3. **Recommends** up to 6 of the strongest candidates as one-line descriptions and
   lets you accept/decline each in an interactive menu. Fewer if the repo doesn't
   hold 6 genuine conventions — it won't pad the list with weak ideas.
4. **Generates** the approved rules as `.macroscope/check-run-agents/*.md` files on a
   new branch (built in an isolated worktree, so your working tree is never touched)
   and **opens a PR** — your review surface. Generated agents are advisory
   (`neutral`) and can't block PRs.
5. You **review, edit, and merge** the PR. Macroscope loads agents from the default
   branch, so merging is what activates them.
6. **Backtests** (optional): the skill recreates a real past PR as a fresh one so your
   now-live Macroscope agents evaluate it for real. You read the actual check-run
   output and iterate.

> The skill never simulates a check run itself. Macroscope is the only thing that
> ever runs one — validation happens through the real backtest in step 6.

## Requirements

- [Claude Code](https://claude.com/claude-code)
- [`gh`](https://cli.github.com/) authenticated (`gh auth login`)
- The Macroscope GitHub app installed on the repo (so agents actually run on PRs,
  and for the backtest)
- For backtesting: **write access** to the repo (the backtest opens a new PR)

## Install

### Option A — as a plugin (recommended)

In Claude Code:

```
/plugin marketplace add govambam/macroscope-check-run-agents
/plugin install macroscope-check-run-agents
```

This auto-discovers the skill and keeps it updated (`/plugin update`). Because the
plugin doesn't pin a `version`, you receive new commits automatically on update.

### Option B — manual copy

Copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/govambam/macroscope-check-run-agents
cp -R macroscope-check-run-agents/skills/macroscope-check-run-agents ~/.claude/skills/      # user-scoped (every repo)
# or, project-scoped:
cp -R macroscope-check-run-agents/skills/macroscope-check-run-agents <your-repo>/.claude/skills/
```

## Use

Open your repository in Claude Code and ask:

> Set up Macroscope check run agents for this repo

or run `/macroscope-check-run-agents`. The skill confirms the target repo, walks the
steps above, and stays in your control — nothing is committed without your go, and it
only merges if you explicitly choose to.

## Backtesting

The optional validation step recreates an existing PR as a fresh one so your live
Macroscope agents evaluate it as new, then points you to the new PR's Checks tab for
the real findings. It uses `pr-backtest.sh`, which is **bundled with this skill** at
`skills/macroscope-check-run-agents/scripts/pr-backtest.sh` — a vendored snapshot of
[pr-backtest-script](https://github.com/govambam/pr-backtest-script). Nothing is
downloaded at runtime, so everything that runs is in this one repo for you to read
before installing. The script does all its work in a disposable temp clone and never
touches your checkout; it does push two branches and open a PR (using your `gh` write
access).

## Repository layout

```
.claude-plugin/
  marketplace.json                 # makes this repo an installable marketplace
  plugin.json                      # the plugin manifest
skills/
  macroscope-check-run-agents/
    SKILL.md                       # the workflow Claude follows
    reference/
      agent-file-format.md         # frontmatter schema + body conventions
      good-rule-heuristics.md      # what makes a rule worth automating
      examples/                    # two exemplar agents
    scripts/
      pr-backtest.sh               # bundled backtest script (vendored snapshot)
```

## License

MIT — see [LICENSE](LICENSE).
