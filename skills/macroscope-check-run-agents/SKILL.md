---
name: macroscope-check-run-agents
description: >
  Bootstrap Macroscope check run agents for the current repository. Studies the
  project's written conventions (CLAUDE.md, AGENTS.md, cursor rules, CONTRIBUTING,
  review skills) plus Macroscope's curated templates, recommends a handful of
  high-value rules, and generates ready-to-edit
  `.macroscope/check-run-agents/*.md` files on a branch + PR. Use when the user
  wants to create, set up, generate, or discover check run agents for their repo,
  or asks "what should Macroscope check on my PRs?". Run this from inside the
  target repository.
---

# Bootstrap Macroscope Check Run Agents

You help a Macroscope user turn the conventions their team *already follows* into
custom check run agents — the AI agents that run on every PR. You do this by
reading what the repo documents — its rule files, contributor docs, and review skills —
then generating real agent files for the user to review, merge, and backtest. When the
repo has thin conventions to mine — or whenever a vetted check would
add value the repo doesn't already have — you can also offer a **Macroscope template**
(curated starting points bundled in `templates/`) instead of inventing a rule.

**Read `reference/agent-file-format.md` and `reference/good-rule-heuristics.md`
before generating anything.** They are the source of truth for the file format and
for what makes a rule worth automating.

## Orient the user first

Before doing anything, tell the user in a couple of sentences what's about to
happen, so the process isn't a black box. Cover the whole arc:

> I'll research this repo — your written conventions (CLAUDE.md, CONTRIBUTING, cursor
> rules, review skills, etc.) — to find good check run agent candidates, then propose
> them for you to accept or decline. Once you confirm which
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

This is the primary signal, so make it fast *and* thorough — but read it from **the
default branch the agents will actually run against (`origin/$DEFAULT`), not the current
working tree.** The user is often on a feature branch, or their checkout is behind the
remote; `git ls-files` and the Read tool would then show the wrong fileset — e.g. miss a
`CONTRIBUTING.md` that's on `main` but not on their branch. The agents run against the
default branch, so that's the only fileset that matters. Fetch it, list candidate files
from it in one sweep, then dump them all in one pass:

```bash
git fetch -q origin "$DEFAULT"
FILES=$(git ls-tree -r --name-only "origin/$DEFAULT" \
  | grep -iE '(^|/)(CLAUDE|AGENTS|CONTRIBUTING|STYLEGUIDE|copilot-instructions)\.md$|\.cursorrules$|(^|/)\.cursor/rules/|(^|/)\.github/(PULL_REQUEST_TEMPLATE|CODEOWNERS)|(^|/)(\.claude|\.agents)/skills/|(^|/)\.macroscope/check-run-agents/')
echo "$FILES"
for f in $FILES; do printf '\n===== %s =====\n' "$f"; git show "origin/$DEFAULT:$f"; done
```

That single pass reads everything from the right ref — faster than per-file Read calls
*and* correct regardless of the current branch. (Only if there's no `origin`/no network,
fall back to the working tree — and **say so**, since discovery then reflects the current
branch, which may not be what ships.)

Prioritize what you read by signal:

- **High signal — read fully:** agent/assistant rule files (`CLAUDE.md`, `AGENTS.md`,
  root **and** nested; `.cursor/rules/**`, `.cursorrules`, `.github/copilot-instructions.md`),
  `CONTRIBUTING*.md`, `STYLEGUIDE*.md`, and review skills (`.claude/skills/**`,
  `.agents/skills/**` — especially review/lint/standards-flavored).
- **Process signals — skim:** `.github/PULL_REQUEST_TEMPLATE*`, `.github/CODEOWNERS`.
- **`docs/**` and ADRs (`docs/adr/**`, `doc/adr/**`) — sample, don't crawl.** They're
  large and mostly low-yield (and not in the sweep above). Only reach for them — via
  `git show "origin/$DEFAULT:<path>"` — if the high-signal files are thin, and even then
  read just the pages whose path/name signals conventions or standards. **Never read an
  entire docs tree** — it's the biggest hidden time sink in this step.

The sweep also surfaces existing **check run agents** (`.macroscope/check-run-agents/**`),
which is the only thing worth deduping against:

- `.macroscope/check-run-agents/**` — **existing custom agents**. Anything they cover is off the table.
- The two **built-in** agents are always on: **Correctness** (runtime bugs) and **Approvability** (merge readiness). Do not propose rules that just restate these.

You may also read linter/formatter/type-checker/CI configs (`.eslintrc*`,
`eslint.config.*`, `.prettierrc*`, `tsconfig*.json`, `ruff.toml`/`pyproject.toml`,
`.golangci*`, `.rubocop.yml`, `.github/workflows/**`) — again from `origin/$DEFAULT`
(`git show`), and **as a source of
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

## Step 2 — Rank and group

Take the Step 1 candidates, dedupe, and score each against the rubric in
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

### Add applicable Macroscope templates

The skill bundles a set of **Macroscope-recommended templates** in `templates/` (see
`templates/README.md` for the catalog) — vetted check run agents that make a good
starting point. Offer them when they fit, using signals you already gathered:

- **ticket-requirements** — the team links work to tickets. Check cheap *local* signals:
  the PR template asking for a ticket link (seen in Step 1), or `PROJ-123`-style IDs in
  branch/commit names (`git log --oneline -30`, `git branch -a`). Needs the team's
  issue-tracker integration connected; flag that dependency in the proposal.
- **language-idioms** — the repo is mostly Go and/or TypeScript.
- **architecture-standards** — multiple services/packages, or a monorepo with real
  module boundaries.
- **accessibility** — the repo has a frontend (JSX/TSX/HTML/components).
- **security-review** — broadly applicable; favor it where the repo handles untrusted
  input, auth, or a web surface.
- **guardrails** — broadly applicable and cheap.

Two triggers, mirroring how you weigh discovered rules:

- **Thin or no discovered conventions** → lead with the most applicable templates rather
  than padding with weak, invented rules.
- **Solid discovered conventions** → still offer the top 1–2 applicable templates that
  cover something the discovered rules and existing agents *don't*.

**Dedupe like any agent.** A template *is* a check run agent — skip it if a discovered rule,
an existing `.macroscope/check-run-agents/*`, or a built-in already covers it (e.g.
don't offer `security-review` to a repo that already has a security agent).

Templates count toward the **same caps** as discovered rules (≤6 candidates, 2–4 agents) —
don't blow the budget just because templates are easy to add. Don't tailor them: they
are added **verbatim** (see Step 4).

## Step 3 — Interactive accept/decline menu

Present the shortlist to the user as concise rule descriptions — **not** the full
agent files. For each rule show three things: **what it flags**, its **severity**, and
**why you're suggesting it** — i.e. where in *their* repo you found the convention.
Make the "why" specific and verifiable: name the file and quote the line where the
convention is written. The user should be able to think "yes, that's our rule, and I can
see where it came from." Example:

> **Require a doc comment on every exported function** — 🟡 Should fix
> *Why:* `CONTRIBUTING.md` says "all exported functions need a doc comment".

This provenance is for the user's decision only — it does **not** go into the
generated agent files (see Step 4).

If a candidate rule would depend on an **integration** (e.g. it needs Sentry, a ticket
tracker, or Slack — see Step 4's "opt into integrations"), say so on that rule's line,
so the user knows it only does anything once that integration is connected to their
Macroscope.

**Macroscope templates appear in this same menu, labeled "Macroscope template."** Their
"why" is not a repo citation — it's "a recommended check that applies to your repo
because <signal>" (the applicability reason from Step 2). Two call-outs to make on a
template's line: (a) `ticket-requirements` only works once the issue-tracker
integration is connected; (b) a template that ships **blocking** (`conclusion: failure`
— `security-review`, `guardrails`) *can fail a PR*, unlike the advisory rules you
author — say so explicitly so the user opts in knowingly.

Use `AskUserQuestion` to let them accept/decline. The tool allows **max 4 options
per question** (multi-select within those 4), so present candidates in batches of ≤4
across one or two questions; instruct the user to check the ones to keep. Keep it to
one or two quick screens.

After selection, let the user reshape conversationally ("merge these two", "make
that a nit", "scope it to `api/`"). Only the accepted rules proceed.

Once they've confirmed, tell them what's next before you start writing files — e.g.
"Got it. I'll write these as agent files and open a PR you can review." Then go.

## Step 4 — Generate on an isolated branch and open a PR

Write the accepted rules into `.macroscope/check-run-agents/*.md` following
`reference/agent-file-format.md` exactly:

- Frontmatter: `title`; `model: claude-opus-4-6`; **`reasoning: high`**;
  **`effort: high`**; `input: full_diff`; and inferred `include`/`exclude`. Default to
  **high reasoning and high effort** — these agents need multi-step tracing (e.g. "this
  `assert` lets the test keep running, and the next line dereferences a value that can
  be nil → panic"), which low/medium effort skips. Don't lower them unless the user
  asks. Keep `input: full_diff` as the default; for a strict per-unit rule (e.g. "never
  log PII in *any* changed file") `input: code_object` runs the agent once per changed
  object and can be more reliable — use it only when a rule clearly needs that per-file
  isolation.
- **Leave `conclusion` unset** so it defaults to **`neutral`** (advisory). Freshly
  generated agents must not be able to **block** anyone's PR before the team trusts
  them — never set `conclusion: failure` here. Tell the user the agents ship advisory
  and they can flip one to blocking later.
- Body: specific "flag X when Y" instructions, explicit severity levels, and a
  **"permission to do nothing"** clause to suppress false positives.
- **Every agent must post findings as inline review comments — which requires the
  `modify_pr` tool.** Inline posting is a `modify_pr` capability: an agent whose
  `tools:` list omits `modify_pr` physically *cannot* post inline and silently falls
  back to the check-run summary (the Checks tab), where findings are far less visible —
  exactly the "issues in the summary, nothing inline" failure. `modify_pr` is in the
  default tool set, so the safe default is to **omit `tools:` entirely**. If you do set
  `tools:` (see "opt into integrations" below), the list **overrides** the defaults —
  you must re-list `modify_pr` and any other defaults you still want. End each agent
  body with this exact output block (use it verbatim so every generated agent is
  consistent):

  > ## Output
  >
  > For each finding, **post an inline review comment on the exact offending line**
  > (file + line), with the severity emoji and a one-sentence explanation of the problem
  > and the fix. After the inline comments, post one top-level PR comment that lists each
  > finding on a single line. If nothing here applies, post a single top-level comment
  > "All clear." and add no inline comments. Never invent findings to fill space.
- **No provenance in the agent body.** Don't write "Source", "seen in #1234", or any
  citation into the `.md` — Macroscope reads the body as instructions at runtime, so a
  citation is noise it might act on. Provenance belongs in the proposal (Step 3) and
  the PR description (below), not in the file the agent runs.
- **Opt into integrations when a rule needs state beyond the diff.** The default tools
  only see code. If an accepted rule needs external state — unresolved Sentry errors on
  a touched file, the status of a referenced ticket, a Slack ping on a 🔴 finding — add
  the matching tool from the table in `reference/agent-file-format.md` (`sentry`,
  `issue_tracking_tools`, `slack`, …) to `tools:`, and **re-list the defaults you still
  need (especially `modify_pr`)** since `tools:` overrides them. The tool silently
  no-ops if that integration isn't connected, so flag the dependency to the user in the
  proposal (Step 3). `reference/examples/observability.md` shows the pattern.
- **For a long, already-written convention, you may reference it instead of restating
  it.** If `CLAUDE.md`/`AGENTS.md` documents a rule at length, the agent body can point
  to that file ("follow the API-error conventions in `CLAUDE.md`") to keep the two in
  sync — but keep a concrete "flag X when Y" trigger inline so the agent still has
  something checkable. Default to inlining short rules.

**Accepted Macroscope templates are copied verbatim — not regenerated.** For each
accepted template, copy its file from this skill's `templates/<id>.md` into
`.macroscope/check-run-agents/` **unchanged**. Resolve the bundled path the same way
Step 5 resolves the backtest script (try `${CLAUDE_PLUGIN_ROOT}/skills/macroscope-check-run-agents/templates/`,
`$HOME/.claude/skills/macroscope-check-run-agents/templates/`, then
`.claude/skills/macroscope-check-run-agents/templates/`). Keep the template's
frontmatter as-is — **including a `conclusion: failure`** where it sets one (those ship
blocking on purpose) — and its own body/output handling (templates carry their own
inline-comment instructions; do **not** append the generated-agent output block). The
"leave `conclusion` neutral", "high reasoning/effort", "add the output block", and
"infer globs" rules above apply to the **discovered-convention** agents you author — *not* to templates.
In the PR description, a template's "why" line reads "Macroscope-recommended template —
applies because <signal>", not a repo file/PR citation.

**Do not disturb the user's working state.** They may have uncommitted work or be on
a feature branch — never stash their changes, switch their checkout, or branch off
their current branch (that would pull their unmerged commits into this PR). Instead
build in an **isolated git worktree** based on the up-to-date default branch, so the
PR contains *only* the agent files:

```bash
git fetch origin "$DEFAULT"
BRANCH=macroscope/check-run-agents
mkdir -p "$HOME/.cache/macroscope-check-run-agents"
WT=$(mktemp -d "$HOME/.cache/macroscope-check-run-agents/wt.XXXXXX")  # predictable parent (so writes here can be pre-approved); mktemp keeps the dir unique + fresh
git worktree add -b "$BRANCH" "$WT" "origin/$DEFAULT"
mkdir -p "$WT/.macroscope/check-run-agents"
# ... write the agent files into "$WT/.macroscope/check-run-agents/" ...
git -C "$WT" add .macroscope/check-run-agents
git -C "$WT" commit -m "Add Macroscope check run agents"
git -C "$WT" push -u origin "$BRANCH"
gh pr create --repo "$REPO" --base "$DEFAULT" --head "$BRANCH" \
  --title "Add Macroscope check run agents" \
  --body-file - <<'MACROSCOPE_PR_BODY'
<the full PR description goes here, verbatim — compose it per "the PR body" below>
MACROSCOPE_PR_BODY
```

**Compose the PR description with the provenance** that you kept out of the agent
files — so the "why" is preserved for whoever reviews or revisits the PR. **Stream it
straight to `gh` over stdin** — no temp file, nothing to stage or clean up, and no extra
write prompt. `--body-file -` reads the body from stdin (confirmed in `gh pr create
--help`), and a **quoted** heredoc (`<<'MACROSCOPE_PR_BODY'`) passes every byte verbatim,
so backticks, `$`, and emoji in the body are never expanded or executed. Two rules make
this bulletproof, and you must follow both:
- the closing `MACROSCOPE_PR_BODY` sits at the **start of its own line** — column 0, no
  leading spaces or tabs — or the heredoc won't terminate;
- **no line of the body may be exactly `MACROSCOPE_PR_BODY`** (it won't be — the token is
  deliberately unique; never write that string into the description).

The body to compose:
- a one-line intro noting the agents are **advisory (neutral)** and won't block PRs;
- a **"Why these rules"** section: one bullet per rule with where it came from — the
  convention file + quoted line (or, for a template, the repo signal that makes it apply);
- a closing line: review the files, then merge to activate.

**Then verify the PR actually got created.** On success `gh pr create` prints the new
PR's URL and exits 0; on failure (branch already has a PR, auth lapsed, network) it exits
non-zero and prints the error. Confirm you got a URL — if the command errored, surface the
exact error and stop here. Do **not** advance to the menu below as if the PR exists.

Notes:
- The worktree lives under a **fixed parent** (`~/.cache/macroscope-check-run-agents/`)
  rather than a random `mktemp` location, so a user who trusts this flow can pre-approve
  the agent-file writes once with a permission rule (e.g.
  `Write(//<home>/.cache/macroscope-check-run-agents/**)` and the matching `Read`/`Edit`)
  and never be prompted per file. The PR is still the review gate. This is opt-in; without
  the rule the writes simply prompt as normal.
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
   - If already merged → go to **Step 5**.
   - If not merged → tell the user it must merge first, and ask whether you should
     merge it now or they'd rather do it themselves. Merge only on their go (see the
     merge command below), then go to **Step 5**.
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

## Step 5 — Backtest (validate against a real PR)

The agents must already be live on the default branch (the menu in Step 4 ensures
this). Validation only happens through this real backtest, because the agents only
execute inside Macroscope's engine on a real PR — so never present a simulated or
hand-reasoned evaluation as if Macroscope produced it.

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
