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
> + the stamp. Two deliberate changes from source: UI-only fields (`iconName`, `color`)
> are dropped, and each template's `tools:` is normalized to include `modify_pr` (so
> findings post as inline comments, not just the check-run summary). Each template here
> is a directly-usable `.md` agent file.
>
> **Wired into the skill.** `SKILL.md` assesses template applicability in Step 2, offers
> fitting templates in the Step 3 menu (labeled "Macroscope template"), and copies
> accepted ones **verbatim** in Step 4.

## The templates

| File | What it checks | `input` | `conclusion` | Integration / notable tools |
|------|----------------|---------|--------------|-----------------------------|
| `ticket-requirements.md` | PR actually implements its linked Jira/Linear tickets | `full_diff` | `neutral` | **needs `issue_tracking_tools`** (Jira/Linear connected) |
| `language-idioms.md` | Go/TS idioms: error wrapping, naming, prefer shared utilities | `full_diff` | `neutral` | defaults only |
| `architecture-standards.md` | Service boundaries, dependency direction, public-API compatibility | `code_object` | `neutral` | defaults (no `modify_pr`/`git_tools` in source) |
| `security-review.md` | Hardcoded secrets, input validation, unsafe deserialization/SSRF | `full_diff` | **`failure`** | defaults |
| `accessibility.md` | a11y in UI changes: reachable controls, alt text, labeled inputs | `full_diff` | `neutral` | defaults |
| `guardrails.md` | Stray `console.log`/`print`, untracked TODO/FIXME, dead code | `full_diff` | **`failure`** | low reasoning/effort (cheap) |

## Design decisions (wired into `SKILL.md`)

- **Blocking preserved.** `security-review` and `guardrails` keep `conclusion: failure`
  — they ship blocking on purpose. The Step 4 menu flags this so the user opts in
  knowingly. (Mined agents the skill *authors* still default to advisory `neutral`.)
- **Tools normalized for inline posting.** Every template's `tools:` includes
  `modify_pr` (the one deviation from source), so findings post as inline comments, not
  only the check-run summary.
- **Added verbatim.** Accepted templates are copied unchanged — no glob tailoring, no
  body edits.
- **Same cap.** Templates count toward the skill's ≤6-candidate / 2–4-agent budget.
- **Signal-driven and deduped.** A template is offered only when a repo signal fits
  (language, structure, UI presence, ticket usage) and isn't already covered by a mined
  rule or an existing agent. `ticket-requirements` is gated on an issue-tracker
  integration being connected.
