# Control Centre — Plan (v0.2)

> **Changelog v0.1 → v0.2.** Revised after a design review (verdict:
> APPROVE-WITH-CHANGES). This version hardens the approval gate into a real
> enforcement boundary, adds rejection/re-approval/amendment loops, closes the
> "convergence ≠ acceptance criteria" gap, thickens the data model with an audit
> trail + idempotency, promotes auth/secrets/kill-switch from open questions to
> prerequisites, and records the standalone-vs-Nano-Urban decision instead of
> defaulting by omission.

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
1. **Nothing is built until a human approves the spec** — and approval is enforced at the
   **backend→workforce boundary**, not merely as a status flag (see §5).
2. **Claude orchestrates the front half; the Nano Workforce executes the back half.**
3. **Thin cockpit, not a re-implementation.** The Control Centre *calls* the existing
   nano-workforce control API (proven this session: `/status`, `/actions/start/*`,
   the Tasks inbox, `compile-delivery-graph`). It does not re-implement orchestration.
4. **Preview before dispatch** (mirrors the workforce's own rule) — show the plan/graph
   and the side effects, then require an explicit human **confirm** carrying a single-use
   token before any start door fires.
5. **Claude can request, never fire.** The orchestrator has no tool that performs a
   side-effecting delegation on its own; it can only *propose* one for a human to confirm.

## 3. The idea → delegation pipeline (the core feature)
Each **Idea** is a first-class record with an explicit state machine — including the
rejection and amendment paths, not just the happy path:

| Stage | Who drives | Output |
|---|---|---|
| Capture | human | raw idea / problem statement |
| Clarify | Claude (interview) | resolved questions, scope, constraints (transcript persisted) |
| Shape | Claude | a **feature spec** + testable acceptance criteria (a new immutable `specVersion`) |
| **Approve** | **human (gate)** | an `Approval` bound to the exact `specVersion`, delivery vehicle chosen |
| Delegate | human confirm → workforce | epic (`plan-fanout`) / PR (`convergence-loop`) / delivery graph |
| Track | system + human | live status pulled back; **acceptance criteria verified at completion** |

**State machine (with the paths v0.1 missed):**
```
capturing → clarifying → shaping → in_review
   in_review ──approve──►  approved ──confirm delegate──► delegated → tracking → done
   in_review ──changes-requested──► shaping         (reason recorded)
   in_review ──reject──► rejected                   (terminal, reason recorded)
   approved  ──spec edited──► in_review             (approval auto-invalidated; re-approval required)
   tracking  ──workforce feedback / build wrong──► shaping   (amend spec → new version → re-approve)
```
- **Editing a spec after approval invalidates the approval.** A delegation can only be
  confirmed for an `Approval` whose `specVersion` equals the spec's current head.
- **Workforce feedback closes the loop back into the spec** — an escalation or a build
  that reveals the spec was wrong routes to `shaping` (amend → new version → re-approve),
  rather than dead-ending in the dashboard.

## 4. Unified UI (one place)
- **Idea board** — kanban of ideas by stage (incl. `changes-requested` / `rejected`).
- **Feature workspace** — chat with Claude + the evolving, versioned spec + acceptance
  criteria + the clarify transcript.
- **Approval gate** — human reviews the spec, picks a vehicle, approves (or requests
  changes / rejects with a reason). Approval is stamped to the current `specVersion`.
- **Delegation confirm** — preview the resolved side effects, then a single-use
  **Confirm & dispatch** action (the only thing that fires a start door).
- **Workforce panel** — `/status` PRs in flight, Tasks inbox (answer escalations),
  delivery-graph propose/preview, submit doors.

## 5. Architecture

### 5.1 Shape — Decision D1 (recorded, to confirm)
The cockpit is a **thin app that calls the workforce control API over HTTP** (Principle 3).
**Open decision D1: Nano Urban app vs standalone Node/React service.** A standalone app
means re-building auth, secret handling, deployment, and a workforce API client — exactly
the surfaces this review flags as risky. A **Nano Urban app** likely inherits auth,
secrets, and deployment and sits closer to the workforce, removing bootstrapping work.
**Recommendation:** evaluate Nano Urban *first*; adopt it if it can host the React cockpit
+ the Claude orchestrator; fall back to a standalone service only if it can't. **Record the
outcome here before building** — do not default by omission.

### 5.2 Components (independent of D1)
- **Front-end:** React + Vite + TypeScript.
- **Backend:** hosts the Claude orchestrator, proxies + aggregates the nano-workforce API,
  persists ideas/specs/approvals/audit. Storage: SQLite to start (data only — **never
  secrets**).
- **Claude orchestrator:** Claude Agent SDK / Messages API. **Stateless per turn** —
  rehydrates from the persisted Idea/Spec/transcript each turn rather than holding an
  unbounded live thread (context = cost); context is capped/curated. Its tools read
  context and save spec drafts; it can **propose** a delegation but has **no tool that
  fires a start door**.

### 5.3 The approval gate — an enforcement boundary (BLOCKER fix)
The gate is enforced in **one place, server-side, atomically** at the backend→workforce
edge — not inside the agent loop and not as a mere status read:

1. Claude (or a human) can only create a **delegation request** for an Idea.
2. A human hits **Confirm & dispatch**; the server issues a **single-use confirm token**.
3. The server executes the start door **iff** `idea.status === approved` **AND** the
   `Approval.specVersion` equals the spec's current head **AND** the confirm token is
   valid and unused **AND** the global `delegationEnabled` kill-switch is on.
4. The idempotency key is **persisted before** the workforce call, so a retry/double-click
   never double-dispatches (ties to §6).

Because the orchestrator's context ingests untrusted text (escalation bodies, PR content),
this boundary is also the **prompt-injection containment**: even a fully subverted agent
cannot cross it, because it holds no capability to.

### 5.4 Live workforce status
Poll `/status` + the Tasks inbox on an interval to start (note staleness / rate-limit); a
**reconciliation job** maps external keys (`prKey`/`planKey`/`digest`) back to
`Delegation` rows and repairs drift. SSE/websocket push is a later optimization, not v1.

## 6. Data model (v0.2 — auditable + idempotent)
Timestamps (`createdAt`/`updatedAt`) and `owner` on every entity.
- `Idea { id, title, status, problem, constraints, owner, createdAt, updatedAt }`
- `Spec { id, ideaId, version, body, acceptanceCriteria[], createdAt }` — **immutable per
  version**; edits create a new version, never mutate one.
- `ClarifyTurn { id, ideaId, role, content, createdAt }` — the spec's provenance.
- `Approval { id, ideaId, specVersion, approvedBy, approvedAt, vehicle, rationale }` — the
  audit record for a button that spends money and mutates a repo.
- `Delegation { id, ideaId, specVersion, vehicle, status, idempotencyKey,
  prKey|planKey|digest, requestedAt, dispatchedAt }`.
- `AuditEvent { id, ideaId, actor, type, payload, at }` — **append-only** log of every
  state transition, approval, and dispatch.
- Workforce status is **read-through** (not owned), linked via the external keys above.

## 7. Acceptance criteria — flow in, verify out (new)
The whole point of the front half is that delivered work matches what was approved, so
close the loop:
- **In:** at delegation, the approved acceptance criteria are written into the epic
  issue / PR body / delivery-graph node prompts the workforce receives.
- **Out:** at the **Track** stage the Delegation is not `done` until its acceptance
  criteria are checked against the merged result (human-verified is acceptable for v1;
  automated checks later). Convergence/merge alone ≠ acceptance met.

## 8. How it gets built — by the Nano Workforce
Repo: `PWFelix/control-centre` (private). Epic issue → `plan-fanout` on
`baseBranch: epic/control-centre` (auto-created), decomposed into levelized tasks.
**Build order (corrected):**
1. Scaffold (app shell)
2. Read-only workforce API proxy (`/status` panel) — the tiny first milestone that proves
   the loop
3. **Persistence** (before the pipeline needs it)
4. Claude orchestrator (stateless per turn)
5. Idea pipeline + spec versioning + clarify transcript
6. Dashboard (Tasks inbox, delivery graphs)
7. **Gate + delegation door — built LAST, hand-reviewed & human-merged (not auto-merged),
   shipped behind `delegationEnabled=false` until the gate is proven.**

Each task → one PR → convergence-loop → merge. Acceptance criteria (from §7) go into each
task's issue body.

## 9. Prerequisites (promoted from "open questions")
- **Auth + secret handling.** The app can spend money, open/merge PRs, and mutate a repo.
  Even single-user: bind to localhost or put behind auth; keep the Claude API key +
  workforce credentials in **env / a secret store, never SQLite**.
- **Kill-switch.** A global `delegationEnabled` flag, **default off** until the gate is
  proven — no delegation can fire while off.
- **Budget guard.** A ceiling / alert on orchestrator + Copilot-review spend.
- **Reviewer bot.** Copilot code review (needs a paid Copilot plan) provisioned via a
  repository ruleset targeting `main` + `epic/*`, so every build PR has a reviewer to
  converge against.
- **Fleet daemon.** Confirmed working — a fleet worker leased and completed a
  `senior:pr-review` round on the plan PR, so `senior:*` task types are serviced.
- **Decision D1** (§5.1) recorded before building.

## 10. Risks
- **Prompt injection via untrusted context** (escalation/PR text enters the orchestrator).
  Mitigated by §5.3 — Claude holds no fire-the-door capability.
- **Double-fire / non-idempotent dispatch** → duplicate epics, real cost. Mitigated by the
  confirm token + persisted idempotency key (§5.3, §6).
- **Stale approval after a spec edit** → delegating a changed spec. Mitigated by
  version-bound approvals (§3).
- **Cost runaway** (unbounded orchestrator context, review spend). Mitigated by stateless
  turns + budget guard (§5.2, §9).
- **Convergence ≠ acceptance.** Mitigated by §7.
- **Bootstrapping tension** — an app that *calls* the workforce while being *built by* it.
  Keep the first milestone tiny (scaffold + read-only `/status`) to prove the loop.
