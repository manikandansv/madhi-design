# Madhi — Context Handoff

**Purpose:** Give Claude Code enough locked context to continue the Madhi build without re-deriving decisions already made. Read this alongside the product and architecture docs — this file is the session brief and working process, not a duplicate of those docs.

**Tagline:** Clear Paths.
**Name:** Madhi (மதி) — Tamil for moon / wisdom / judgment. The UI's signature visual (moon-phase indicators for progress and milestone status) comes directly from this.

**Architecture direction:** Domain-driven design with strong event-driven workflow modeling. Spring Boot remains the API and infrastructure shell, and the business workflow logic is modeled as commands and domain events. Akka is a later runtime option for highly asynchronous or supervision-heavy orchestration, not the default starting point.

---

## Document index

| Document | What it covers |
|----------|---------------|
| `docs/product/Madhi_Product_Writeup_v1.md` | Full feature spec, governing principle, AI modes, guardrails, phase breakdown |
| `docs/architecture/TechnicalArchitecture.md` | All-phases architecture: stack decisions, AI wrapper, data layer, streaming platform |
| `design/hld/HLD_Phase1_Intake_Milestones.md` | Phase 1 functional design: responsibilities, inputs/outputs, key tradeoffs |
| `design/lld/LLD_Phase1_Intake_Milestones.md` | Phase 1 implementation detail: types, function signatures, API contracts, edge cases |
| `design/architecture/madhi-system-architecture.html` | Visual system architecture diagram (all phases, all layers) |
| `design/mockups/index.html` | Static UI mock — Phase 1 intake flow and preview dashboard |

For governing principle, AI mode table, feature guardrails, and phase order: **read the product writeup.** They live there; don't duplicate them here.

---

## What already exists

A static, interactive UI mock (`design/mockups/index.html`) covering the Phase 1 intake flow (5-step form, moon-phase progress) and a preview dashboard (milestone list with moon-phase status, a static allocation-mix teaser). No backend — hardcoded example data only. Built to see the look of the feature set before any logic is written; treat it as a visual reference, not a foundation to build directly on top of without review.

The HLD and LLD for Phase 1 are complete (see index above). No source code exists yet — `src/` is empty.

---

## Working process to keep

- **One decision at a time.** Resolve, confirm, move on — don't queue up multiple milestones' worth of design choices in one pass.
- **Per milestone: HLD (responsibilities, inputs/outputs, key tradeoffs) → confirm → LLD (data structures, function signatures, edge cases) → confirm → code.** Never skip straight to code generation.
- **State tradeoffs plainly with an actual recommendation**, not a neutral list — then let the human confirm or override.
- **Don't build ahead of the current phase.** Phase 1 only, until it's a complete, working slice.
