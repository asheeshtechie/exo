# 00 — Design Principles (Ingestion Platform)

This document is the constitution for the ingestion platform. Every later spec — scenario catalog, skills library, surface contract, infra blocks — defers to these principles. When a tension arises, this document wins; later specs must either resolve to the principle or explicitly call out the deviation and justify it.

Principles are deliberately few. Each one carries hard consequences.

---

## P1 — Agent-First Platform Design

**Statement.** The platform is designed with the agent (and a future federation of domain agents) as the primary user. There is one contract; human interfaces are built on top of the same surfaces the agent uses, not separate.

**Why.** When the agent is bolted on later, its surface ends up shaped by whatever the platform happened to expose, not by what operating the platform actually requires. Agent-first inverts that: scenarios drive skills, skills demand surfaces, infra implements them. It also forces dogfooding — if the agent can't drive it, no consumer can.

**Consequences.**

- Scenarios are the primary spec input; skills are the primary unit of capability; infra blocks are substrate.
- No "internal" APIs the agent doesn't have access to. One contract.
- Surfaces stay domain-agnostic where possible, so this contract can later host a fleet (retrieval, generation, metadata, memory) without redesign.
- Spec ordering is fixed: scenarios → skills → surface contract → blocks.

**Anti-patterns.**

- "Build the platform first; we'll add the agent later."
- Two parallel APIs (one for humans, one for agents) with different shapes.
- Designing a block by listing what its tech *can* do rather than what skills demand.

---

## P2 — Dial-able Autonomy with Trust Progression

**Statement.** Autonomy is a knob, not a destination. Every skill operates at an autonomy level (L0–L5) per context (per pipeline / per tenant / per environment), starts low, and is promoted only on explicit evidence and operator action. Humans can always see what the agent is doing, propose alternatives, intervene mid-flight, and demote autonomy at any time.

**Why.** Trust in an agent operating production systems is earned, not declared. Day-1 the agent should be observed; year-3 some skills in some contexts may be fully autonomous. The platform must make this progression explicit, visible, and reversible — otherwise trust is impossible to build and impossible to verify.

**Consequences.**

- Every skill declares a default starting autonomy level and explicit promotion criteria.
- Every action API supports a `propose` mode in addition to `execute`.
- Intervention APIs (pause, veto, revert, override) are first-class and uniformly available.
- Reasoning traces (not just action logs) are emitted and queryable.
- Promotion is operator-driven and evidence-based; demotion is automatic on incident, on confidence drop, or on operator concern.
- Autonomy ladder is locked: **L0 Observed → L1 Suggesting → L2 Batched-approval → L3 Auto-with-notify → L4 Auto-with-audit → L5 Trusted.**

**Anti-patterns.**

- Binary "agent on / agent off."
- Silent auto-promotion based on heuristics.
- "Trust me, the model is good now" without an evidence trail.
- Hidden agent actions (no notification, no log).

---

## P3 — Structured Over Unstructured

**Statement.** Operational signals — failures, remediations, lineage, audit, reasoning — are structured per controlled vocabularies. Free-form text is supplementary, never primary.

**Why.** Agents reason well over structured data and poorly over stderr blobs. Without structure, every diagnosis is a fresh LLM problem; with structure, it becomes a lookup against a pattern library. The same property makes the data queryable, dashboardable, auditable, and improvable. This is the single highest-leverage choice for agent-operability — and one of the most commonly skipped.

**Consequences.**

- Failure-class taxonomy is mandatory; every failed run carries one.
- Remediation-hint vocabulary is mandatory; jobs annotate failures with intended fix.
- Lineage events emit in a fixed schema, not free text.
- Audit log is structured (actor, action, prior state, post state, reason), not narrative.
- Reasoning traces are structured (evidence, alternatives, confidence), not prose blobs.
- New failure modes get added to the vocabulary; never absorbed as `other`.

**Anti-patterns.**

- "We'll just LLM-parse the logs."
- `failure_message` (free text) as the only failure signal.
- Free-form runbooks the agent has to reinterpret each time.
- Audit logs only humans can read.

---

## P4 — Reversibility and Blast Radius are First-Class

**Statement.** Every action declares its reversibility and its blast radius. Defaults bias to reversible and small-blast. Irreversible or wide-blast actions are gated regardless of the actor's autonomy level.

**Why.** Confidence in agent (or human) operation depends on knowing the worst case is bounded and recoverable. If every action could in principle delete production data, no autonomy level is safe. By treating reversibility and blast radius as *declared* properties, the platform can grant autonomy proportionally — and bound mistakes when they happen.

**Consequences.**

- Action specs include `reversibility` and `blast_radius` fields, machine-checkable.
- Reversible actions store their inverse before executing.
- Wide-blast actions require explicit approval — sometimes from humans, even at high autonomy levels.
- Autonomy promotion is gated by these properties: a skill that performs irreversible cross-tenant work doesn't reach L5 by definition.
- "Undo" is a first-class platform operation, not a runbook footnote.

**Anti-patterns.**

- Treating all actions as equivalent for permission purposes.
- Irreversible actions issued without explicit `reversibility=irreversible` declaration.
- Recovery procedures that exist only as runbooks, not as inverse operations.

---

## P5 — Graceful Degradation Under Pressure

**Statement.** The platform handles overload, spikes, noisy neighbors, and partial failures by slowing down and prioritizing — not by crashing, silently falling behind, or failing critical work to serve best-effort work.

**Why.** Ingestion platforms see spikes constantly: backfills, source floods, retry storms, model-upgrade re-embeds, cost-driven catch-ups. The platform that tolerates them gracefully is operable; the one that doesn't generates incidents constantly and erodes trust in both the platform and the agent faster than anything else.

**Consequences.**

- Quotas and namespaces enforced per tenant, per source, per pipeline.
- Fair scheduling across tenants within each priority class.
- Admission control with backpressure from downstream blocks; throttle dispatch rather than queue without bound.
- Event-trigger coalescing and rate limits.
- Adaptive throttling on rising failure rate; auto-quarantine when circuit-breaker conditions are met.
- Backfills governed separately from steady-state with their own quotas.
- Overload signals (admission-rejected, dispatch-throttled, class-capacity-utilization, fair-share-deficit) are first-class and distinct from failure signals.

**Anti-patterns.**

- FIFO queueing under load — one tenant starves others.
- Unbounded queues that defer the failure to "later" with no signal.
- Retries that ignore platform-level retry budget.
- Backfill as just a parameterized normal run.

---

## How to use this document

When writing a scenario, skill, surface, or block spec:

1. Check it against each principle.
2. If the spec satisfies the principle, no action needed.
3. If the spec conflicts with a principle, either rework the spec or — if the conflict is unavoidable — add an explicit "deviation note" with rationale. Deviations are reviewable; silent violations are not.

This document is editable, but changes are versioned and require explicit reasoning. Principles are sticky on purpose.
