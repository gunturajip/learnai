# devplan.README.md

> **README for:** [`devplan.md`](./devplan.md) — the LearnAI implementation plan.
> **Status:** Sidecar index for the plan. Read this first; the plan is long.
> **Last updated:** 2026-06-13

---

## What this is

`devplan.md` is a ~1,000-line implementation plan for **LearnAI**, a personalized learning plan generator. It's derived from the product brief in [`idea2.txt`](./idea2.txt) and converted into a developer-executable plan: phased tasks, file paths, schema, API design, test strategy, and ranked risks.

**Read this README first** to understand the plan's shape. Read the devplan when you're about to start a phase.

---

## Source documents

| Doc | Purpose | Status |
|---|---|---|
| [`idea.txt`](./idea.txt) | Original product idea — **BacktestAI** (trading backtest app, the in-flight sibling project) | Active, in development |
| [`idea2.txt`](./idea2.txt) | Second product idea — **LearnAI** (learning plan generator, this plan's source) | Concept, not yet started |
| [`devplan.md`](./devplan.md) | Implementation plan for LearnAI | **DRAFT** — awaiting D1–D15 decisions |
| [`CLAUDE.md`](./CLAUDE.md) | Project-wide guidance for AI assistants (rank 1 patterns, run commands, plan file index) | Stable |
| [`docs/superpowers/plans/`](./docs/superpowers/plans/) | Implementation plans for BacktestAI (the active project) | Multiple in-flight |

**This plan reuses the rank 1 architectural patterns from `CLAUDE.md` directly** — the two products share tech stack and conventions, just different domains.

---

## Plan structure at a glance

| § | Section | Read it when… |
|---|---|---|
| 1–2 | Executive summary + Goals/Non-Goals | You need a 30-second elevator pitch or scope boundaries |
| 3 | Open Decisions (D1–D15) | **You need to make a decision** before Phase 0 starts |
| 4 | High-Level Architecture | You need to understand service topology or are writing the VPS deploy config |
| 5 | Repository Structure | You're scaffolding a new module or wondering where a file lives |
| 6 | Database Schema | You're writing migrations, queries, or RLS policies |
| 7 | API Design | You're building a route handler or writing a typed client |
| 8 | Frontend Structure | You're adding a page, component, or hook |
| 9 | AI Prompt Architecture | You're iterating on Claude prompts or the 3-pass pipeline |
| 10 | Phased Implementation | **You are about to start work** — find your phase, find your task |
| 11 | Testing Strategy | You're writing tests or auditing coverage |
| 12 | Security | You're implementing auth, RLS, or rate limits |
| 13 | Deployment | You're pushing to the VPS or wiring CI/CD |
| 14 | Risks & Mitigations | You're making an architectural call and want a sanity check |
| 15 | Open Questions (Q1–Q6) | You hit a decision the plan intentionally deferred |
| 16 | Definition of Done | You're closing out a phase PR |
| 17 | References | You need an external doc (FSRS, Stripe, Notion, Anki) |

---

## Phase status

| Phase | Scope | Duration | Status |
|---|---|---|---|
| **Phase 0** | Project setup (repo, Docker, CI, rank 1 patterns) | 3–5 days | ⛔ Blocked on D1–D15 |
| **Phase 1** | Public marketing site + Auth (Supabase) | Weeks 1–3 | ⛔ Blocked on Phase 0 |
| **Phase 2** | Dashboard + Plan CRUD + topic tree + seed sessions | Weeks 4–6 | ⛔ Blocked on Phase 1 |
| **Phase 3** | AI plan generation (3-pass Claude pipeline, Celery, WebSocket) | Weeks 7–10 | ⛔ Blocked on Phase 2 |
| **Phase 4** | Session review list + 10-metric engine + 3 charts | Weeks 11–13 | ⛔ Blocked on Phase 3 |
| **Phase 5** | Stripe + exports (PDF/CSV/Notion/Anki/ICS) + Google Calendar + polish | Weeks 14–16 | ⛔ Blocked on Phase 4 |
| **V1 (V2 product)** | Plan A/B comparison, walk-forward, Monte Carlo | +6–8 wks | 🔒 Scoped in `idea2.txt`, not yet detailed |
| **V2 (V3 product)** | Portfolio learning, AI validation, adaptive reschedule | TBD | 🔒 Scoped in `idea2.txt`, not yet detailed |
| **V3 (V4 product)** | Public API, team workspaces, marketplace, PostHog | TBD | 🔒 Scoped in `idea2.txt`, not yet detailed |

Legend: ⛔ Blocked · 🟡 In progress · ✅ Done · 🔒 Scoped, not yet planned

---

## Open Decisions (must resolve before Phase 0)

The plan is blocked on **15 decisions** (D1–D15) that the author intentionally left open. They're listed in `devplan.md` §3. The high-impact ones:

| # | Question | Recommended answer | Needs owner? |
|---|---|---|---|
| **D1** | Repo location | `D:\Freelance\fxrnd_kodaepik\learnai\` (sibling to `backtestai/`) | **Yes — your call** |
| D2 | Shared infra with backtestai? | No — separate VPS, shared patterns | No |
| D3 | Spaced-repetition algorithm | **FSRS v4** (open-source, modern) | No |
| D4 | Topic tree source | User-defined for MVP, PDF upload in Phase 5 | No |
| D5 | AI model | Claude Sonnet 4.6 (quality) + Haiku 4.5 (cheap pass 1) | No |
| D10 | Payments | Stripe (3 tiers: Free / Pro / Enterprise) | No |
| D15 | FSRS library | `py-fsrs` package or port directly | No |

**Action:** review §3 of the devplan, lock in D1 (and D2 if you disagree), then Phase 0 is unblocked.

---

## How to use this plan

**If you're a developer starting Phase 0:**
1. Read §3 (Open Decisions) and §5 (Repository Structure) — 15 min.
2. Confirm D1 (repo location) with the project owner.
3. Work through Phase 0 tasks (0.1–0.13) in order.
4. Run the Phase 0 exit criteria checklist from §10 before declaring done.

**If you're a developer starting Phase N (N ≥ 1):**
1. Verify the previous phase's exit criteria are met (check git log, ask).
2. Read §10's Phase N block. Each task is atomic and includes the file paths and tests.
3. Don't skip tasks. Tasks are ordered so that later tasks depend on earlier ones.

**If you're reviewing the plan:**
1. Read §1 (Executive Summary) — 2 min.
2. Skim §10 (Phased Plan) to see the full sequence.
3. Read §14 (Risks & Mitigations) — that's where the author's uncertainty lives.
4. Comment on §3 (Open Decisions) — these are the unblockers.

**If you're a stakeholder evaluating the product:**
1. Read `idea2.txt` (the product brief) first.
2. Read §1 of the devplan (Executive Summary) for tech and scope confirmation.
3. Read §10 phase durations to see the time-to-MVP (16 weeks).
4. Don't read §6 or §7 unless you want SQL or endpoint specs.

---

## How to update this plan

The plan is a **living document** during execution, not a fixed spec. Update it when:

- A task is completed → mark with ✅ in the phase table above
- A new risk is discovered → add to §14
- A decision is resolved → strike through D-N in §3 and move the answer to a "Decisions" subsection
- A phase is closed → update §16 (Definition of Done) with the actual completion date
- The scope shifts → update §2 (Goals/Non-Goals) and the affected phase

**Versioning:** the plan has no version number; git history is the version history. Tag the commit when a phase closes (`git tag phase-0-complete`).

**PR template for plan updates:**

```markdown
## Plan update

**Which section(s):** §
**Reason:** (what changed in reality that the plan needs to reflect)
**Tasks affected:** (re-numbered, added, removed, or unchanged)
**Approval:** (which stakeholder signed off, or "self-approved for typo")
```

---

## Approval workflow

| Change | Approver |
|---|---|
| Typo, formatting, broken link | Self-approved, merge directly |
| Task re-numbering, wording tightening | Self-approved, merge directly |
| New risk added to §14 | Self-approved, note in PR |
| New task added to a phase | Project owner approval (you) |
| Phase scope change (add/remove) | Project owner approval |
| §3 decision resolved | Project owner approval |
| §2 Goals/Non-Goals change | Project owner approval |
| New phase created | Project owner approval |

---

## Related reading order

If you're new to the project, read in this order:

1. **[`CLAUDE.md`](./CLAUDE.md)** — project-wide patterns and run commands (10 min)
2. **[`idea2.txt`](./idea2.txt)** — what we're building and why (20 min)
3. **This README** (5 min)
4. **[`devplan.md`](./devplan.md) §1, §3, §10** — executive summary, open decisions, phased plan (30 min)
5. **[`devplan.md`](./devplan.md) §6, §7, §8** — schema, API, frontend (only when starting implementation)
6. **[`devplan.md`](./devplan.md) §14** — risks (before making any architectural call)

Total onboarding: ~65 minutes.

---

## Status banner

> **🛑 DRAFT — do not start coding.**
>
> Phase 0 cannot begin until **D1 (repo location)** is resolved by the project owner. D2–D15 have recommended defaults but should be confirmed if the project owner has a preference.
>
> Once D1 is locked, Phase 0 tasks 0.1–0.13 can run as a 3–5 day spike to validate the stack end-to-end.
