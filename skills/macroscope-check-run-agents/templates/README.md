# Macroscope check run agent templates

Curated, ready-to-use check run agents Macroscope recommends as starting points. The
skill can offer one of these when a repo has no strong idioms of its own to mine, or
add one alongside mined rules when it covers something valuable the repo doesn't
already check.

> **Status: DRAFT / vendored.** These are not yet published in the Macroscope app or
> docs. Vendored 2026-06-18 from `prassoai/back` PR #12314 (branch
> `jon/add-check-run-templates`), file
> `targets/app/src/modules/code-reviews/CheckRunAgentTemplates/data.ts`. When the
> templates ship in the docs, repoint to `docs.macroscope.com` and re-sync this folder
> + the stamp. UI-only fields from the source (`iconName`, `color`) are intentionally
> dropped; each template here is a directly-usable `.md` agent file.
>
> **Not yet wired into the skill.** The agent files exist, but `SKILL.md` does not yet
> reference or recommend them — that logic is still to be designed.

## The templates

| File | What it checks | `input` | `conclusion` | Integration / notable tools |
|------|----------------|---------|--------------|-----------------------------|
| `ticket-requirements.md` | PR actually implements its linked Jira/Linear tickets | `full_diff` | `neutral` | **needs `issue_tracking_tools`** (Jira/Linear connected) |
| `language-idioms.md` | Go/TS idioms: error wrapping, naming, prefer shared utilities | `full_diff` | `neutral` | defaults only |
| `architecture-standards.md` | Service boundaries, dependency direction, public-API compatibility | `code_object` | `neutral` | defaults (no `modify_pr`/`git_tools` in source) |
| `security-review.md` | Hardcoded secrets, input validation, unsafe deserialization/SSRF | `full_diff` | **`failure`** | defaults |
| `accessibility.md` | a11y in UI changes: reachable controls, alt text, labeled inputs | `full_diff` | `neutral` | defaults |
| `guardrails.md` | Stray `console.log`/`print`, untracked TODO/FIXME, dead code | `full_diff` | **`failure`** | low reasoning/effort (cheap) |

## Things to weigh when we design the recommend logic

- **Two templates ship `conclusion: failure` (blocking)** — `security-review` and
  `guardrails`. That differs from the skill's own generated-agent default
  (`neutral`/advisory). Decide whether to preserve the templates' blocking default or
  downgrade to advisory on first add.
- **`ticket-requirements` only works with an issue-tracker integration** connected to
  the customer's Macroscope; otherwise it no-ops. Gate recommending it on that.
- **Several templates omit `modify_pr`/`git_tools`** that the source author left out
  (e.g. `architecture-standards`, `accessibility`). If we adopt the skill's
  "inline comments require `modify_pr`" rule, we may need to reconcile.
- **Applicability varies by repo** — accessibility only matters for UIs; language-idioms
  is language-specific; architecture-standards suits multi-service/monorepos. The
  recommend logic should be signal-driven, not one-size-fits-all.
