# What makes a good check run agent rule

A check run agent runs on every PR. A bad rule — vague, subjective, or a duplicate of
another check run agent — trains the team to ignore the agent. Score every candidate
on the four tests below. A rule should clearly pass all four before you propose it.

## 1. Objective & diff-checkable

Can an LLM reviewer actually decide this from the PR diff plus browsing the
codebase? Favor rules with a checkable trigger.

- **Good:** "New HTTP route handlers must register an entry in `routes.ts`."
- **Good:** "Any new public function in `sdk/` needs a doc comment."
- **Bad:** "Code should be well-architected." (subjective, no trigger)
- **Bad:** "Make sure the feature is what the customer wanted." (not in the diff)

Rephrase soft conventions into a concrete trigger, or drop them.

## 2. Recurring / worth automating

Automate things that come up again and again — not one-offs.

- Strong: a theme raised across **multiple merged PRs by multiple reviewers**, or
  stated emphatically in `CONTRIBUTING.md`/`CLAUDE.md`.
- Weak: a single reviewer's one-time nit on one PR.

Recurrence is the clearest evidence the team actually cares and that an agent will
earn its keep.

## 3. Not duplicating another check run agent

The only thing worth deduping against is **another check run agent** — an existing
custom agent in `.macroscope/check-run-agents/*`, or the built-in **Correctness**
(runtime bugs) and **Approvability** (merge readiness). If a rule just restates what
one of those already does, drop it.

**Overlap with linters, formatters, type-checkers, and CI is fine — often valuable.**
Do *not* drop a rule because a linter *could* catch it. A linter only helps when it's
actually configured, enabled, and reliable for that case — in practice rules get
disabled inline, are never set up, or miss the semantic call entirely. If a PR reaches
review with the violation still in it, the linter *didn't* catch it, and an agent that
does is a real save, not noise. Read linter configs to *understand* the team's
conventions (a custom lint rule is a codified convention worth encoding) — never as a
reason to exclude a candidate.

- Keep: "prefer `require` over `assert` when a later line dereferences the value" —
  a linter might flag some assert/require swaps, but it won't reason about the
  nil-then-deref panic, and a miss ships a flaky test.
- Drop: a rule that just re-runs the built-in Correctness bug hunt, or repeats a
  sibling custom agent already in the repo.

## Conventions, not bugs (the most important filter when mining PRs)

A check run agent encodes a **standing convention** — a rule that applies to every
future PR. It is **not** a place to park individual bug findings. This matters most
in Step 2, because reviewer comments are a mix of both:

- **A bug report** points at a defect *in the PR under review* — "this null-derefs",
  "this leaks a file handle", "wrong boundary here". It's specific to that diff and
  doesn't generalize. Macroscope's built-in **Correctness** agent already hunts this
  entire class on every PR. **Do not turn these into check run agents.**
- **A convention** states a rule the team holds *independent of any one PR* — "we
  always add a changeset", "endpoints get registered in the router", "no PII in
  logs". It applies to PRs the reviewer hasn't even seen yet.

The test: can you restate the comment as **"on any PR, flag X when Y"** without
referring to this PR's specific code? If yes, it's a convention — keep it. If you
can only describe it by pointing at this PR's lines, it's a bug — drop it.

When unsure, drop it. A missed convention costs nothing; a bug masquerading as a
convention produces a vague agent that fires randomly and erodes trust.

## 4. High value when violated

Prefer rules where a miss actually hurts — correctness, security, data integrity,
API/contract stability, on-call burden — over cosmetic preferences. Cosmetic rules
can still ship as 🟢 Nit but shouldn't crowd out the important ones.

---

## Grouping into agents

Don't make one agent per rule. Bundle related rules into a **small number of agents
(aim for 2–4)** organized by domain, matching how Macroscope's own examples
consolidate concerns:

- `api-conventions.md` — routing, error handling, versioning, public surface
- `testing.md` — coverage expectations, test placement, fixtures
- `frontend.md` — accessibility, event tracking, component patterns
- `security.md` — secrets, PII logging, authz checks

Scope each agent with `include`/`exclude` globs so it only runs on relevant files.

## The phrasing checklist for each generated rule

- States a concrete **trigger** ("flag X when Y").
- Carries a **severity** (🔴 / 🟡 / 🟢).
- Says what **not** to flag where false positives are likely.
- Ends the agent with **explicit permission to report nothing** on a clean PR.
- Has **provenance tracked for the user** (source file + line/quote, or PR # +
  reviewer + comment excerpt) — shown in the proposal and PR description, **not**
  written into the agent file.

## Anti-patterns — do not propose

- Restating **another check run agent** — an existing custom agent, or the built-in
  Correctness / Approvability. (Overlapping with a *linter* is fine; overlapping with
  another *agent* is the duplication to avoid.)
- A one-off bug finding from a PR comment dressed up as a rule (see "Conventions,
  not bugs" above) — the Correctness agent already owns this.
- Subjective taste with no checkable trigger.
- A rule invented by you that the repo doesn't actually hold. If you can't cite a
  source for it, don't propose it.
