# Check run agent file format

Source: https://docs.macroscope.com/check-run-agents

Each agent is one `.md` file in `.macroscope/check-run-agents/` at the repo root.
The filename (minus `.md`) becomes the default title. Agents run on every PR — on
open, on push, and on manual rerun — alongside Macroscope's two built-in agents
(**Correctness** and **Approvability**).

> **Reserved:** `approvability.md` and `ignore` are reserved and must stay in
> `.macroscope/` root, not in the `check-run-agents/` subfolder.
>
> **Activation:** Macroscope reads agent files from the **default branch**. Changes
> only take effect on new PRs after they're merged.

## Anatomy

```
---
# optional YAML frontmatter (all fields optional; sensible defaults apply)
---
markdown instructions for the agent
```

## Frontmatter schema

| Field | Default | Options / notes |
|-------|---------|-----------------|
| `title` | filename | string, max 60 chars — name shown in the GitHub Checks tab |
| `model` | `claude-opus-4-6` | `claude-opus-4-5/4-6/4-7/4-8`, `claude-sonnet-4-5/4-6`, `gpt-5-2/5-4/5-5` |
| `reasoning` | `low` | `off`, `low`, `medium`, `high` |
| `effort` | `low` | `low`, `medium`, `high` |
| `input` | `full_diff` | `full_diff` (the PR diff) or `code_object` |
| `tools` | `browse_code`, `git_tools`, `github_api_read_only`, `modify_pr` | extras (need connections): `web_tools`, `slack`, `sentry`, `posthog`, `launchdarkly`, `bigquery`, `amplitude`, `gcp_cloud_logging`, `issue_tracking_tools`, `image_gen`, `mcp` |
| `include` | none | glob patterns — only matching changed files are reviewed |
| `exclude` | none | glob patterns — matching files are skipped |
| `conclusion` | `neutral` | `neutral` (advisory) or `failure` (can block) |
| `showToolCalls` | `true` | `true`/`false` — log tool calls on the check run page |
| `waitsFor` | none | check names or `["*"]` — wait for other checks first |
| `waitsForTimeout` | `20` | 1–60 minutes |

### Scoping with include/exclude

- Neither set → all changed files reviewed.
- Only `include` → only matching files.
- Only `exclude` → everything except matches.
- Both → `include` narrows first, then `exclude` carves out exceptions, e.g.
  `include: ["src/**"]` + `exclude: ["src/gen/**"]`.
- Repo-wide exclusions in `.macroscope/ignore` apply additively.

## Writing the body (instructions)

The body is plain markdown telling the agent what to do. Macroscope's guidance:

- **Be specific.** "Flag any exported function over 50 lines without a doc comment"
  beats "review for quality".
- **Define severity levels** so output is consistently triaged.
- **Use headings** to organize multiple concerns within one agent.
- **Reference concrete paths** where rules live.
- **Tell it what *not* to flag.**
- **Give the agent permission to do nothing** — explicitly, so it doesn't
  manufacture findings on a clean PR. This is the single most important line for
  keeping the agent trustworthy.

The agent formats findings however you instruct (tables, emoji severity, checklists,
grouped output). Results surface in the check run details, as inline PR comments,
and/or as a top-level PR comment.

## Conventions this skill uses for generated agents

To keep generated agents readable, this skill writes each rule as a `###` section
with:

- a **severity** marker — 🔴 Must fix / 🟡 Should fix / 🟢 Nit
- a **What to flag** description (the "flag X when Y")
- a **Don't flag** line where useful

**No provenance in the agent file.** Do *not* write "Source", "seen in #1234", or any
citation into the agent body. Macroscope reads the body as instructions at runtime, so
provenance is noise the agent might try to act on. Provenance is surfaced to the user
elsewhere — in the pre-pick proposal and in the PR description (see `SKILL.md`
Steps 4–5) — never in the `.md` the agent runs.

See `examples/` for two complete agents in this shape.
