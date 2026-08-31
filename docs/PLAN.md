# Control Centre — Plan (draft v0.1)

## 1. Purpose — the gap it fills
Today you have workers/agents that can execute, and a running **Nano Workforce** that
drives PRs and epics to completion. What's missing is the **front half**: turning a raw
idea into a well-formed, approved feature *before* anything gets built — and a single
place to do it. Right now that would live in two places (an idea/spec tool + the
workforce console). The **Control Centre** unifies both:

- **(A) Idea → Feature orchestration** — an instance of Claude runs the pipeline
  *capture → clarify → shape a spec → human approval gate → delegate*.
- **(B) Workforce dashboard** — live `/status`, the Tasks/escalation inbox, delivery
  graphs, and the submit doors — in the **same** UI.

## 2. Principles (non-negotiable)
1. **Nothing is built until a human approves the spec.** The approval gate is the only
   thing that unlocks a delegation (a side-effecting workforce start door).
2. **Claude orchestrates the front half; the Nano Workforce executes the back half.**
3. **Thin cockpit, not a re-implementation.** The Control Centre *calls* the existing
   nano-workforce control API (proven this session: `/status`, `/actions/start/*`,
   the Tasks inbox, `compile-delivery-graph`). It does not re-implement orchestration.
4. **Preview before dispatch** (mirrors the workforce's own rule) — show the plan/graph
   and the side effects before firing a start door.

## 3. The idea → delegation pipeline (the core feature)
Each **Idea** is a first-class record moving through stages:

| Stage | Who drives | Output |
|---|---|---|
| Capture | human | raw idea / problem statement |
| Clarify | Claude (interview) | resolved questions, scope, constraints |
| Shape | Claude | a **feature spec** + acceptance criteria |
| **Approve** | **human (gate)** | approved spec, delivery vehicle chosen |
| Delegate | Claude → workforce | epic (`plan-fanout`) / PR (`convergence-loop`) / delivery graph |
| Track | system | live status pulled back from the workforce |

## 4. Unified UI (one place)
- **Idea board** — kanban of ideas by stage.
- **Feature workspace** — chat with Claude + the evolving spec doc + acceptance criteria.
- **Approval gate** — human reviews the spec, picks the delivery vehicle, approves.
- **Workforce panel** — `/status` PRs in flight, Tasks inbox (answer escalations),
  delivery-graph propose/preview, submit doors.

## 5. Architecture (proposed — reviewer to challenge)
- **Shape:** its own web app that consumes the nano-workforce control API over HTTP.
- **Front-end:** React + Vite + TypeScript.
- **Backend:** Node/TS service that (a) hosts the Claude orchestrator, (b) proxies +
  aggregates the nano-workforce API, (c) persists ideas/specs (SQLite to start).
- **Claude orchestrator:** Claude Agent SDK / Messages API with a system prompt defining
  the feature-creation role and tools (read context, save spec, and — only after
  approval — call the workforce start doors).
- **Workforce integration:** the same endpoints used this session.
- **Approval gate:** a server-enforced rule — a start door can only be called for an
  Idea in `approved` state.

## 6. Data model (first cut)
`Idea { id, title, status, problem, constraints }` ·
`Spec { ideaId, body, acceptanceCriteria[], version }` ·
`Delegation { ideaId, vehicle, prKey|planKey|digest, submittedAt }` · workforce status
is read-through (not owned).

## 7. How it gets built — by the Nano Workforce
1. Repo: `PWFelix/control-centre` (name/visibility TBC).
2. Epic issue describing the build.
3. `plan-fanout` with `baseBranch: epic/control-centre` (auto-created) decomposes it into
   levelized tasks (scaffold → API proxy → Claude orchestrator → idea pipeline →
   dashboard → approval gate → persistence), one PR per task, each converged + merged.

## 8. Immediate steps — the "before building" gate (this session)
1. Finalize this plan with you.
2. Create repo + a **plan-doc PR** (this plan as `docs/PLAN.md`).
3. Submit that PR to `convergence-loop` (**convergeOnly** — no merge) so the workforce
   **`senior:pr-review`** reviewer reviews the plan.
4. I address the review to convergence.
5. **You approve.**
6. *Then* create the epic + `plan-fanout` to actually build it.

## 9. Open questions / prerequisites
- **Repo name + visibility** (default: `PWFelix/control-centre`, private).
- **Reviewer bot provisioning.** The convergence loop reviews *against an automated
  reviewer* (Copilot code review) on the repo. Now provisioned via a repository ruleset
  ("Request pull request review from Copilot") targeting `main` and `epic/*`, so every PR
  is auto-reviewed and the loop has a reviewer to converge against.
- **Fleet daemon.** Confirmed working — a fleet worker leased and completed a
  `senior:pr-review` round on this PR, so `senior:*` task types are being serviced.
- **Claude API key** for the orchestrator (a build-time concern, not needed to plan).
- **Auth model** for the Control Centre — local single-user to start?

## 10. Risks
- The workforce reviewer is tuned for **code** review; reviewing a prose plan may be
  shallow. Mitigation: write the spec with explicit, testable acceptance criteria.
- Bootstrapping tension: an app that *calls* the workforce while being *built by* it —
  keep the first milestone tiny (scaffold + one live `/status` panel) to prove the loop.
