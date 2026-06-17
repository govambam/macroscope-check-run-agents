---
name: macroscope-check-run-agents
description: >
  Bootstrap Macroscope check run agents for the current repository. Studies the
  project's written conventions (CLAUDE.md, AGENTS.md, cursor rules, CONTRIBUTING,
  review skills) and mines recent merged PRs for recurring human review comments,
  recommends a handful of high-value rules, and generates ready-to-edit
  `.macroscope/check-run-agents/*.md` files on a branch + PR. Use when the user
  wants to create, set up, generate, or discover check run agents for their repo,
  or asks "what should Macroscope check on my PRs?". Run this from inside the
  target repository.
---

# Bootstrap Macroscope Check Run Agents

You help a Macroscope user turn the conventions their team *already follows* into
custom check run agents — the AI agents that run on every PR. You do this by
reading what the repo documents and by mining what human reviewers actually say in
merged PRs, then generating real agent files for the user to review, merge, and
backtest.

**Read `reference/agent-file-format.md` and `reference/good-rule-heuristics.md`
before generating anything.** They are the source of truth for the file format and
for what makes a rule worth automating.

## The one rule you must never break

Macroscope is the *only* thing that runs a check run. **Never** simulate, role-play,
or approximate a check run yourself (e.g. "let me read the agent and check this
diff"). The only validation this skill offers is the **true backtest** in Step 6,
which runs the real Macroscope engine. Do not imply Macroscope evaluated anything
when it hasn't.

## Orient the user first

Before doing anything, tell the user in a couple of sentences what's about to
happen, so the process isn't a black box. Cover the whole arc:

> I'll research this repo — your written conventions (CLAUDE.md, CONTRIBUTING, etc.)
> and the review comments on your recent merged PRs — to find good check run agent
> candidates, then propose them for you to accept or decline. Once you confirm which
> to keep, I'll generate the agent files and open a PR. You can review, edit (or ask
> me to edit), and merge it when you're happy — merging is what activates the agents.
> After that we can stop, or run a sanity-check backtest against one of your past PRs.

Then proceed.

## Preconditions and target confirmation

Confirm the environment, then **confirm the target repo with the user before doing
anything that reads or writes their GitHub**:

```bash
git rev-parse --show-toplevel                                   # in a git repo?
gh auth status                                                  # gh authenticated?
REPO=$(gh repo view --json nameWithOwner --jq .nameWithOwner)
DEFAULT=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name)
echo "Target: $REPO  (default branch: $DEFAULT)"
```

- If you're not in a git repo or `gh` isn't authenticated, stop and say exactly what
  to fix (`cd` into the repo / `gh auth login`).
- **Show `$REPO` and `$DEFAULT` to the user and confirm they're correct before
  continuing.** `gh` resolves the repo from this checkout's *default remote* — in a
  fork (an `origin` plus an `upstream`) that may not be the repo they mean. If it's
  wrong, have them run `gh repo set-default <owner>/<repo>` or reopen the session in
  the right repo.
- **Pin the target for the whole run:** pass `--repo "$REPO"` on every `gh` command
  from here on (and `repos/$REPO/...` on every `gh api` call) so nothing silently
  drifts to another repo if the remotes are ambiguous. The captured `$REPO` and
  `$DEFAULT` are the source of truth for the rest of this skill.

---

## Step 1 — Discover written conventions (static)

Find where rules are already written down. Read each that exists:

- Agent/assistant rule files: `CLAUDE.md`, `AGENTS.md` (root **and** nested), `.cursor/rules/**`, `.cursorrules`, `.github/copilot-instructions.md`
- Human docs: `CONTRIBUTING*.md`, `STYLEGUIDE*.md`, `docs/**` (conventions/standards pages), ADRs (`docs/adr/**`, `doc/adr/**`)
- Review skills: `.claude/skills/**`, `.agents/skills/**` (especially anything review/lint/standards-flavored)
- Process signals: `.github/PULL_REQUEST_TEMPLATE*`, `.github/CODEOWNERS`

Then read what already exists so you don't duplicate **another check run agent** —
that's the only thing worth deduping against:

- `.macroscope/check-run-agents/**` — **existing custom agents**. Anything they cover is off the table.
- The two **built-in** agents are always on: **Correctness** (runtime bugs) and **Approvability** (merge readiness). Do not propose rules that just restate these.

You may also read linter/formatter/type-checker/CI configs (`.eslintrc*`,
`eslint.config.*`, `.prettierrc*`, `tsconfig*.json`, `ruff.toml`/`pyproject.toml`,
`.golangci*`, `.rubocop.yml`, `.github/workflows/**`) — but **as a source of
conventions, not an exclusion filter.** Do *not* drop a candidate just because a
linter might catch it: if a violation reaches review, the linter didn't catch it
(disabled inline, never configured, or it can't make the semantic call), and an agent
that catches the miss is valuable. Overlap with linters is fine — overlap with another
**check run agent** is the only thing to avoid. See `reference/good-rule-heuristics.md`
test 3.

Collect each discovered rule with its **provenance**: the **file path** *and* the
specific **line or short quote** that states the convention (e.g. `CONTRIBUTING.md` →
"every endpoint must be registered in the router"). You'll show this to the user so
they can see exactly where in their repo the rule came from — capture enough to be
convincing, not just a filename.

## Step 2 — Mine merged PRs (behavioral)

The strongest signal is what reviewers repeat. Pull recent merged PRs and their
**human** review comments:

```bash
# Recent merged PRs
gh pr list --repo "$REPO" --state merged --limit 30 --json number,title,author,mergedAt,url

# For each PR you deep-fetch, pull inline review comments + review summaries
gh api --paginate "repos/$REPO/pulls/<number>/comments" --jq '.[] | {user:.user.login, body:.body, path:.path}'
gh pr view <number> --repo "$REPO" --json reviews --jq '.reviews[] | {user:.author.login, state:.state, body:.body}'
```

**Mind the API cost.** Deep-fetching comments is (at least) one API call *per PR*.
Don't fan out across hundreds of PRs — cap the deep-fetch to roughly the **20–30 most
recent merged PRs**. If the repo is much larger, say so and sample the recent window
rather than the whole history.

Then:

- **Filter out bots.** Exclude `*[bot]`, `macroscopeapp`, `dependabot`, `codecov`, `github-actions`, and any CI/automation accounts. You want **humans**.
- **Cluster into themes.** Group comments that express the same underlying rule
  ("always add a changeset", "don't log PII", "new endpoints need a test",
  "use the `Result` type not exceptions").
- **Keep conventions, drop bug reports.** A comment pointing out a *specific bug in
  this PR* ("this will null-deref if `user` is nil", "off-by-one here") is **not** a
  check run agent — it's a one-off correctness finding, and Macroscope's built-in
  **Correctness** agent already covers that whole class. Only keep a comment if it
  expresses a **generalizable, repeatable rule** that would apply to future PRs the
  reviewer hasn't seen yet. If you can't restate it as "on any PR, flag X when Y"
  without naming this PR's specifics, it's a bug, not a rule — drop it. See the
  "Conventions, not bugs" section in `reference/good-rule-heuristics.md`.
- **Recurrence is the signal.** A theme raised across **multiple PRs / multiple
  reviewers** is a strong candidate. A one-off nit is not — drop it.
- Keep, per candidate, its **provenance**: the **PR numbers**, the **reviewer(s)**,
  and a **short excerpt** of a representative comment (e.g. #1841, @alice — "we always
  register new endpoints in the router"). This is what you'll show the user to justify
  the rule.

If the repo has few merged PRs or comments are sparse, say so plainly and lean on
Step 1's written conventions.

## Step 3 — Rank and group

Merge Step 1 + Step 2 candidates, dedupe, and score each against the rubric in
`reference/good-rule-heuristics.md` (objective & diff-checkable, recurring, not
duplicating another check run agent, high value). Drop anything weak.

Take the strongest candidates, **capped at 6**. This is a ceiling, not a target —
**do not pad to reach it.** If only three rules clear the bar, propose three. A
short, high-confidence list is the goal; weak suggestions teach the team to ignore
the agent. Quality over count, every time.

Group the survivors into a **small number of agents** (2–4), bundling related rules
under one agent the way Macroscope's own examples do (e.g. one "API conventions"
agent, one "Testing" agent) — not one agent per rule. Infer `include`/`exclude`
globs from where each rule applies.

## Step 4 — Interactive accept/decline menu

Present the shortlist to the user as concise rule descriptions — **not** the full
agent files. For each rule show three things: **what it flags**, its **severity**, and
**why you're suggesting it** — i.e. where in *their* repo you found the convention.
Make the "why" specific and verifiable: name the file and quote the line for written
conventions, and cite the PR(s), reviewer(s), and a short comment excerpt for mined
ones. The user should be able to think "yes, that's our rule, and I can see where it
came from." Example:

> **Require a doc comment on every exported function** — 🟡 Should fix
> *Why:* `CONTRIBUTING.md` says "all exported functions need a doc comment", and
> reviewers asked for it in #1234 (@alice) and #1310 (@bob).

This provenance is for the user's decision only — it does **not** go into the
generated agent files (see Step 5).

Use `AskUserQuestion` to let them accept/decline. The tool allows **max 4 options
per question** (multi-select within those 4), so present candidates in batches of ≤4
across one or two questions; instruct the user to check the ones to keep. Keep it to
one or two quick screens.

After selection, let the user reshape conversationally ("merge these two", "make
that a nit", "scope it to `api/`"). Only the accepted rules proceed.

Once they've confirmed, tell them what's next before you start writing files — e.g.
"Got it. I'll write these as agent files and open a PR you can review." Then go.

## Step 5 — Generate on an isolated branch and open a PR

Write the accepted rules into `.macroscope/check-run-agents/*.md` following
`reference/agent-file-format.md` exactly:

- Frontmatter: `title`; `model: claude-opus-4-6`; **`reasoning: high`**;
  **`effort: high`**; `input: full_diff`; and inferred `include`/`exclude`. Default to
  **high reasoning and high effort** — these agents need multi-step tracing (e.g. "this
  `assert` lets the test keep running, and the next line dereferences a value that can
  be nil → panic"), which low/medium effort skips. Don't lower them unless the user
  asks.
- **Leave `conclusion` unset** so it defaults to **`neutral`** (advisory). Freshly
  generated agents must not be able to **block** anyone's PR before the team trusts
  them — never set `conclusion: failure` here. Tell the user the agents ship advisory
  and they can flip one to blocking later.
- Body: specific "flag X when Y" instructions, explicit severity levels, and a
  **"permission to do nothing"** clause to suppress false positives.
- **Every agent must post findings as inline review comments.** Without this,
  Macroscope dumps findings into the check-run summary (the Checks tab), where they're
  far less visible and actionable. End each agent body with this output-format
  instruction (verbatim or close to it):

  > For each finding, **post an inline review comment on the exact offending line**
  > (file + line), with the severity emoji and a one-sentence explanation of the
  > problem and the fix. After the inline comments, post one top-level PR comment that
  > lists each finding as a single line. If the diff is clean, post a single top-level
  > comment "All clear." and add no inline comments. Never invent findings to fill space.
- **No provenance in the agent body.** Don't write "Source", "seen in #1234", or any
  citation into the `.md` — Macroscope reads the body as instructions at runtime, so a
  citation is noise it might act on. Provenance belongs in the proposal (Step 4) and
  the PR description (below), not in the file the agent runs.

**Do not disturb the user's working state.** They may have uncommitted work or be on
a feature branch — never stash their changes, switch their checkout, or branch off
their current branch (that would pull their unmerged commits into this PR). Instead
build in an **isolated git worktree** based on the up-to-date default branch, so the
PR contains *only* the agent files:

```bash
git fetch origin "$DEFAULT"
BRANCH=macroscope/check-run-agents
WT=$(mktemp -d)
git worktree add -b "$BRANCH" "$WT" "origin/$DEFAULT"
mkdir -p "$WT/.macroscope/check-run-agents"
# ... write the agent files into "$WT/.macroscope/check-run-agents/" ...
git -C "$WT" add .macroscope/check-run-agents
git -C "$WT" commit -m "Add Macroscope check run agents"
git -C "$WT" push -u origin "$BRANCH"
gh pr create --repo "$REPO" --base "$DEFAULT" --head "$BRANCH" \
  --title "Add Macroscope check run agents" \
  --body-file "$WT/pr-body.md"     # compose this first (see below); never commit it
```

**Compose the PR description with the provenance** that you kept out of the agent
files — so the "why" is preserved for whoever reviews or revisits the PR. Write it to
a temp file (e.g. `$WT/pr-body.md`, which is **not** inside `.macroscope/` so it's
never staged/committed) and pass it via `--body-file`. Include:
- a one-line intro noting the agents are **advisory (neutral)** and won't block PRs;
- a **"Why these rules"** section: one bullet per rule with where it came from — the
  convention file + quoted line, and/or the PR(s), reviewer(s), and comment excerpt;
- a closing line: review the files, then merge to activate.

Notes:
- If `$BRANCH` already exists from a prior run, use a fresh name (e.g. add a `-2`
  suffix) or, only with the user's ok, delete the stale branch first.
- **Keep the worktree (`$WT`) alive for the rest of the session** — it's the scratch
  space for "Propose changes". Remove it (`git worktree remove --force "$WT"`) only at
  a terminal step (merge and finish, reject and close, or after the backtest). The
  user's original checkout is never touched.

Once the PR is open, give the user the PR link, note they can always view the diff or
edit the files directly in their editor, then present an **interactive menu**
(`AskUserQuestion`) of what to do next. **Take no action until they pick one — and
never merge unless they explicitly choose a merge option here** (or tell you to
directly). Present these four choices:

1. **Backtest the agents** — sanity-check the agents against a real past PR using the
   live Macroscope engine. Handle merge state (a backtest needs the agents live on the
   default branch):
   - Check whether the PR is merged: `gh pr view <N> --json state,mergedAt`.
   - If already merged → go to **Step 6**.
   - If not merged → tell the user it must merge first, and ask whether you should
     merge it now or they'd rather do it themselves. Merge only on their go (see the
     merge command below), then go to **Step 6**.
2. **Merge and finish** — merge the PR and end. Confirm the merge succeeded, remind
   them the agents are now live (advisory) on new PRs, remove the worktree
   (`git worktree remove --force "$WT"`), and stop.
3. **Propose changes** — ask what they want changed, edit the files **in the worktree
   (`$WT`)**, then `git -C "$WT" commit` and `git -C "$WT" push` (the PR updates
   automatically). Summarize what changed, then **re-present this same menu**.
4. **Reject and close** — close the PR without merging
   (`gh pr close <N> --repo "$REPO"`), offer to delete the branch, remove the worktree
   (`git worktree remove --force "$WT"`), and end.

When merging (choices 1 or 2), use an explicit, non-interactive method that matches
the repo's convention — default to squash: `gh pr merge <N> --repo "$REPO" --squash`.
Never run a bare interactive `gh pr merge`.

## Step 6 — Backtest (validate against a real PR)

The agents must already be live on the default branch (the menu in Step 5 ensures
this). The backtest runs the **real Macroscope engine** — you never simulate a check
run yourself.

**First, choose which PR to backtest.** Ask the user (`AskUserQuestion`, two options):

1. **Let Claude pick one** — find a good candidate automatically.
2. **I'll paste a URL** — the user provides a specific merged-PR URL.

If Claude picks: find a recently **merged** PR that will actually exercise the agents
— prefer one that touches **multiple files** and has a **substantial diff (~500+
lines changed)**, ideally one whose history inspired a generated rule:

```bash
gh pr list --repo "$REPO" --state merged --limit 50 \
  --json number,title,additions,deletions,changedFiles,mergedAt,url \
  --jq 'map(select((.additions + .deletions) >= 500 and .changedFiles >= 2))
        | sort_by(.mergedAt) | reverse | .[0:5]'
```

Show the top candidate(s) and confirm before running. If none clear the bar, say so
and either fall back to the largest available PR or ask the user to paste a URL.

**Before running, set expectations honestly:**
- The backtest only produces findings if the **Macroscope GitHub app is installed on
  `$REPO`**. If the recreated PR shows no Macroscope checks, that's the cause — the
  app isn't installed — not a problem with the agents.
- The backtest uses `pr-backtest.sh`, which **ships bundled with this skill** (a
  vendored copy of [pr-backtest-script](https://github.com/govambam/pr-backtest-script))
  — there is **no runtime download**. It uses the user's **write access** to push two
  branches and open a new PR in their repo, but all its own work happens in a
  disposable temp clone, so their checkout is never touched. Offer to show the script
  (it's in this skill's `scripts/` folder) before running.

Locate the bundled script (works for both plugin and manual-copy installs) and run it:

```bash
SCRIPT=""
for c in \
  "${CLAUDE_PLUGIN_ROOT:-}/skills/macroscope-check-run-agents/scripts/pr-backtest.sh" \
  "$HOME/.claude/skills/macroscope-check-run-agents/scripts/pr-backtest.sh" \
  ".claude/skills/macroscope-check-run-agents/scripts/pr-backtest.sh"; do
  [ -n "$c" ] && [ -f "$c" ] && { SCRIPT="$c"; break; }
done
[ -z "$SCRIPT" ] && echo "Bundled pr-backtest.sh not found — check the skill is installed correctly." >&2
bash "$SCRIPT" <PR_URL>
```

(If you already know this skill's absolute install path, just run
`scripts/pr-backtest.sh` from there directly — the loop above is only a fallback.)

It prints the new `[backtest] …` PR's URL. Point the user to that PR's **Checks tab**
to read the **actual** Macroscope findings.

**Afterwards, clean up.** The backtest leaves a `[backtest] …` PR and its branches in
the repo. Once the user has read the findings, offer to close that PR and delete its
branches so the repo isn't littered.

If an agent misfires (false positive, too vague, missed something), offer to adjust
the agent file — that loops back through generate → PR → merge → backtest. Otherwise,
confirm the agents look good, remove the worktree if it's still around, and finish.

---

## Style notes

- Be concrete and evidence-backed. Every proposed rule traces to a file or PR.
- Prefer fewer, sharper agents over many noisy ones. A check run agent that cries
  wolf gets ignored.
- Never invent conventions the repo doesn't actually hold. If a category is empty,
  say so.
