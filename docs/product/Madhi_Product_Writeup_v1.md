# Madhi

**Tagline:** Clear Paths.
*(Madhi — மதி — moon, wisdom, judgment. The product shows you where you stand and what paths are open; it doesn't choose for you.)*

**One-liner:** Personal finance portfolio management with AI capability to help you reach your own milestones — not an advisor that tells you what to do, a system that shows you where you are, what's possible, and why.

---

## 1. Who it's for

**India-first.** EPF, PPF, NPS, 80C/80D/Section 24b, the old vs. new tax regime, and the Account Aggregator framework are all structural to this product. Global applicability is not a goal for this build — the tax and retirement structures are India-specific and the product doesn't abstract over them.

Anyone from a first-time earner to someone twenty years into their career. The product's core shape is simple: understand who you are and where you're starting from, understand where you want to go, and show the paths between the two — including ones you didn't ask about but could reach. What differs between a fresher and someone twenty years in isn't the product — it's the inputs, not an assumed profile.

## 2. Governing principle

Same discipline as everything else built this way: **every AI-driven feature must answer "what breaks if we remove this."** Where the answer is "nothing, it's just math," it stays deterministic. AI is used only for language understanding (parsing what someone says about their goals) and judgment that a fixed rule can't capture (why a specific alert happened). Every AI output is a suggestion the person approves — never an instruction they follow.

**Three AI capability modes are used across phases — chosen by what the task actually requires:**

| Mode | What it does | Where it applies |
|------|-------------|-----------------|
| **Deterministic** | Pure arithmetic or rule lookup — no AI | Feasibility math, rule engine, tax gap, variance |
| **Single-call AI** | One prompt → structured response | Goal parsing, milestone suggestion, allocation narration |
| **Agentic** | Multi-step: observes intermediate result → decides next action → loops until complete | Goal clarification, PDF statement parsing, alert root-cause investigation |

The governing principle applies to all three modes. A multi-step loop doesn't make something more "AI" — it makes it more capable where a single call genuinely isn't enough. Agentic patterns are introduced only when the task has a variable number of steps that can't be predetermined.

## 3. Core features

### 3.1 Intake
Structured facts (age, income, dependents, existing commitments, location/background) — plain form, no AI. Location is collected as context only, not as a basis for assuming commitments — those are still asked directly, since inferring them from geography risks stereotyping rather than accuracy. Free-text goals (e.g. a home purchase, a target retirement age, a recurring trip) — AI parses this into structured goal objects (type, amount, timeframe). Prior investment experience is asked directly, not inferred — it's used later to set how technical the explanations are, nothing else.

### 3.2 Milestone setting
Deterministic checklist catches the obvious gaps (no emergency fund mentioned). AI suggests milestones specific to what the person actually said that a fixed checklist wouldn't catch (e.g. flagging a coverage gap tied to a stated family commitment). Nothing is added to the plan without explicit approval.

### 3.3 Path & tradeoffs
Feasibility is a formula: given monthly investment capacity, a goal amount, and a timeline, can it be hit at a reasonable rate of return. Where it can't, the engine computes real alternatives (longer timeline, smaller target, higher monthly contribution) — it never picks one and calls it better.

The same engine also runs in the other direction: where someone has surplus capacity beyond their stated goals, it checks that surplus against a standard set of common milestones (emergency fund, earlier retirement, an additional goal) and surfaces the ones that are actually feasible — "you could also reach this." Screening against those templates is deterministic; deciding which one is worth surfacing, and why, is AI's job. Same rule as everywhere else: it's shown as an option to consider, never added to the plan without approval.

AI's job throughout is narrating what each option means, in language matched to the person's stated experience level.

**Multi-goal constrained optimization** extends this engine to the harder case: when multiple goals compete for one shared monthly surplus and it isn't enough to fund all of them on their stated timelines, something has to give. A constrained optimization solver — a deterministic algorithm, not an AI call — takes each goal's amount, timeline, and a priority weight, and computes how to split limited capacity across them. Two objective modes: minimize total delay across all goals while respecting priority order, or fully protect the highest-priority goal first and delay the others proportionally. AI has three specific, narrow jobs in this capability: convert a fuzzy priority statement ("the house matters more than the trip, but don't push retirement past 55") into structured weights the solver can use; narrate what the solver decided and why, in plain language matched to the person's experience level; and when even the minimum version of every goal doesn't fit (infeasible), lay out the tradeoff options clearly — never silently picking one. The human decides. **What breaks if AI is removed:** nothing structurally — the solver still runs with equal weights by default and produces correct output. It is generic rather than personalized to stated priorities, but the math is sound. This follows the same deterministic-first pattern as every other capability in this product.

**Inflation-adjusted goal amounts** are built into this engine — not optional. When someone states a goal amount (e.g. ₹80 lakhs for a home in 7 years), the feasibility math works against the inflation-adjusted figure (~₹1.2 crore at 6% inflation), not the stated number. Both are shown — "you said ₹80L; in 7 years that's approximately ₹1.2 crore in today's purchasing power" — so the person understands what they're actually planning for.

**EPF and PPF passive corpus projection** is part of the retirement feasibility calculation. For salaried users with EPF, the system projects the corpus at retirement age given current balance, monthly contribution (12% of basic), and the current guaranteed rate. PPF is similarly projected if a balance was provided. This right-sizes how much additional investment retirement actually requires — without it, the plan over-states the gap and produces unnecessarily alarming numbers.

### 3.4 Portfolio allocation guidance
Generic asset-class buckets only — equity / debt / NPS / PPF / REIT / gold-silver ETF / mutual funds as categories, never specific funds or tickers. Allocation percentages and contribution limits (PPF, NPS) follow standard, well-established financial-planning rules — not an AI decision. Direct equities are recommended as an amount only, explicitly left to the person's own analysis and discretion. AI explains *why* a given mix fits the person's timeline and risk profile — it doesn't choose the mix.

**Loan prepayment vs. invest tradeoff** is surfaced here for anyone with an active home loan and investable surplus. The math is deterministic: effective home loan cost after the Section 24b interest deduction, compared against expected post-tax return on equivalent investment in the suggested allocation. The system already has the loan EMI, tenure, and monthly investable capacity from intake. AI narrates what each option means for the person's specific timeline — it doesn't recommend one over the other.

**Preference elicitation for allocation** addresses a gap in the age-based formula: two people with identical age, income, and timeline are treated as equivalent even when their actual risk tolerance differs. The primary mechanism is a scored risk-tolerance questionnaire — this mirrors what SEBI requires registered advisors to use for risk profiling, and it stands alone as sufficient without any AI involvement. AI has three specific, narrow jobs: an optional open-ended follow-up question for cases the fixed questionnaire doesn't capture well, reusing the same goal-clarification-loop pattern already built in Phase 1 rather than building a new mechanism; flagging stated contradictions (someone scoring as high-risk-tolerant in the questionnaire but writing "I really can't afford to lose this" in a goal description) back to the person — never silently overriding the score; and narrating why the resulting allocation follows from their specific profile. **What breaks if AI is removed:** nothing — the questionnaire is fully functional and produces a valid risk profile on its own, the same instrument a SEBI-registered advisor uses before recommending an allocation.

**Tax planning gap flagging** is also surfaced here, since 80C-eligible instruments (ELSS, PPF, NPS, home loan principal) are categories within the allocation itself. The system computes how much of the ₹1.5L 80C limit is being used, whether 80D (health insurance premium) is claimed, and whether the home loan interest deduction under Section 24b is being applied. It also runs an old vs. new tax regime comparison given the person's income and deductions. All of this is deterministic arithmetic — no AI. Framed as "you may be leaving ₹X unclaimed," not tax advice.

**RAG-grounded regulatory explanation (Phase 3 portfolio extension — not required for the core build).** The tax-gap arithmetic above is and remains 100% deterministic. This extension addresses only the AI narration that explains *why* a given gap matters. 80C sub-limits, 80D premium caps, and Section 24b conditions change periodically with Finance Act amendments. Narration that relies on the model's training data goes stale between training cutoffs and actual rule changes. This extension grounds the explanatory narration in a small, maintained corpus of current rule text and circulars, retrieved at prompt time before generating the explanation. The tax calculation is untouched — only the explanatory layer is grounded this way. What breaks if this extension is not built: the narration still runs, using trained-in knowledge, which is correct for stable rules but may lag on recent amendments. This is a portfolio extension demonstrating retrieval-grounded generation; it is not a claim that the product gives tax advice.

### 3.5 Statement parsing
User-provided CSV or PDF only — no bank or investment-account integration in this phase. CSV is deterministic parsing. PDF statements (starting with one format) are a genuine AI use case — extracting structured transaction data from documents that aren't cleanly tabular.

**Input validation — deterministic engineering discipline, not an AI capability.** Every uploaded file is validated before any processing begins. File size is checked against a hard limit; files exceeding it are rejected with a plain explanation, not silently failed. Accepted formats are PDF and CSV only — anything else is rejected at the boundary with a clear message. Password-protected or unreadable PDFs are detected and rejected before they reach the parsing agent. The principle throughout is reject-and-explain: the user receives a specific reason for every failure, never a generic error.

**PII masking before any AI call — deterministic, server-side.** Bank and investment statements contain real PII: account numbers, customer names, IFSC codes, and sometimes addresses. None of this reaches the language model. Before statement content is sent to the PDF parsing agent, a deterministic masking step replaces account numbers (e.g., XXXX1234), customer names, and other sensitive identifiers with redacted placeholders. Only transaction-level data — date, amount, merchant description — reaches the model. The masking is string replacement; it does not require an AI call and does not depend on model judgment. This is a hard architectural constraint, not a best-effort measure.

**Output validation — deterministic, part of the existing agentic pipeline.** After the agent extracts and categorizes transactions, extracted totals are reconciled against the statement's own summary figures (opening balance, closing balance, total debits, total credits). If they do not match within a defined tolerance, the extraction is flagged as anomalous rather than silently accepted. This is deterministic arithmetic — no AI — and is the validation step already documented in the agentic pipeline (extract → categorize → validate against summary → flag anomalies). It is not a separate feature; it is the fourth step of the same loop.

### 3.6 Tracking, variance & alerts
Transaction categorization is a deterministic lookup for the common cases (merchant name/code → category); AI only classifies what the lookup can't resolve. Variance against the plan is arithmetic. When an alert fires, AI investigates *why* — checking spending, income, and whether the goal itself changed — and narrates the actual cause instead of just the number.

### 3.7 Adaptive milestones
When a goal changes, the same feasibility engine (3.3) recomputes the path and the same narration capability (3.3) explains what changed. Not a new capability — reuse of what already exists.

### 3.8 Life event timeline
A single visual showing every goal and future commitment plotted on a forward timeline — planned marriage, child's education, parent care, home purchase, retirement. No new data is collected; everything comes from intake. The value is comprehension: people don't naturally hold five competing future obligations in their head simultaneously. Seeing them together on one axis reveals conflicts (a home down payment and a planned marriage in the same 3-year window, for instance) before feasibility math even runs. Deterministic display — no AI.

### 3.9 Tax planning gap flagging (time-sensitive mode)
In addition to the static gap view in Phase 3, this surfaces as a time-sensitive alert in tracking once statement data is available: "3 months left in the financial year — ₹45,000 of your 80C limit is unclaimed." The alert is calendar-aware (fires in Q3/Q4 of the financial year) and uses actual invested amounts from parsed statements rather than stated figures from intake. Deterministic throughout.

## 4. Explicit guardrails

- **No bank/investment-account integration.** Real integration means India's Account Aggregator framework — licensing, consent flows, RBI security requirements. Out of scope; user-provided data stands in.
- **No specific fund, stock, or product recommendations.** Category-level allocation only. Direct equities are amount-only, fully the user's own decision.
- **Framed as educational/informational throughout**, not financial advice — with a disclaimer that means something, not boilerplate.
- **Every AI suggestion requires human approval** before it affects the plan.

## 5. Future state (beyond this build)

Listed here as real possibilities, not commitments — several would need serious groundwork before they're responsible to build:

- **Live account integration** (Account Aggregator or direct bank/broker APIs) — replacing user-fed CSV/PDF. Needs real compliance work first; a sandbox environment (e.g. Setu's AA sandbox) is a reasonable first step, not the real thing.
- **Multi-goal constrained optimization** *(Phase 2)* — when multiple goals compete for one shared monthly surplus and it isn't enough to fund all of them on their stated timelines, a deterministic solver computes how to split limited capacity across them given priority weights. AI converts fuzzy priority statements into weights the solver can use, narrates what the solver decided in plain language, and when the problem is infeasible, explains the tradeoff options — never silently resolving it. What breaks if AI is removed: nothing structurally — the solver runs with equal weights by default, correct output, generic narration.
- **Preference elicitation for allocation** *(Phase 3)* — a scored risk-tolerance questionnaire (SEBI-aligned, the standard instrument in registered-advisor practice) replaces the purely age-based allocation formula, producing a risk profile that varies meaningfully between two people with identical demographics. AI provides an optional follow-up question for edge cases the questionnaire doesn't capture (reusing the Phase 1 goal-clarification pattern), flags stated contradictions the person introduced in their own words, and narrates why the resulting allocation fits their profile. What breaks if AI is removed: nothing — the questionnaire is fully functional alone.
- **Broader PDF statement support** — beyond the single format built first.
- **Deeper spending intelligence** — patterns and habits beyond simple category variance.
- **Licensed advisory tier** — genuinely personalized fund/product recommendations. This crosses into regulated financial-advisor territory and would need actual licensing, not just more AI — it's named here so it's not confused with anything in the current build.
- **RAG-grounded regulatory explanation** *(Phase 3 portfolio extension)* — grounds AI narration for tax-gap explanations in a maintained corpus of current rule text and circulars rather than relying on trained-in knowledge. The tax arithmetic is deterministic and unchanged. Only the explanatory narration retrieves from the corpus. Demonstrates retrieval-grounded generation in a low-stakes context (narration quality, not calculation correctness).
- **LLM-as-judge narration evaluation** *(portfolio extension, applies Phase 2 onward)* — extends the evaluation harness to score generated narration (feasibility explanations, allocation rationale, root-cause explanations) against a rubric rather than by exact string match. Narration quality is subjective and cannot be graded by comparison to a reference answer; a separate model call scores each generated output for accuracy to the underlying numbers, clarity, and tone appropriateness for the stated experience level. This is an evaluation tool — it is not part of the product itself and does not affect what users see.

## 6. Phase-wise development milestones

Small first, same discipline as the rest of this work — each phase should be a complete, working slice before the next begins.

**Phase 1 — Intake + milestone setting.**
Structured facts (education, employment, present + future commitments, current assets), AI-parsed free-text goals, deterministic + AI-suggested milestones, human approval.
- *Added:* **Life event timeline** in the plan view — goals and future commitments plotted on a single forward axis, so conflicts are visible before math runs. No new data collection; deterministic display only.
- *Agentic:* **Goal clarification loop** — when a goal description is ambiguous, the system asks one targeted follow-up question and re-parses with the answer rather than blocking or failing. This is a 2-turn loop — the simplest form of agency — and the entry point for the agentic pattern in the product.

**Phase 2 — Feasibility & path.**
The "you are here → here's a path" engine: feasibility math, alternative-path generation, AI narration in plain language matched to experience level.
- *Added:* **Inflation-adjusted goal amounts** — feasibility math works against the real future cost, not the today's-money figure stated at intake. Both numbers shown to the user. Non-negotiable; without this, the plan optimises for the wrong target.
- *Added:* **EPF / PPF passive corpus projection** — retirement feasibility calculation includes the corpus already building passively through EPF and PPF, based on balance and contribution from intake. Right-sizes how much additional saving retirement actually requires.
- *Added:* **Multi-goal constrained optimization** — when multiple goals compete for the same monthly surplus and capacity is insufficient for all of them on their stated timelines, a deterministic constrained-optimization solver computes how to allocate limited capacity given priority weights. Two objective modes: minimize total delay across all goals, or fully protect the highest-priority goal and delay the rest proportionally. AI's role is strictly: convert a fuzzy priority statement into structured weights the solver can use; narrate what the solver decided and why in plain language; and when the problem is infeasible (no feasible split exists even at minimum goal sizes), lay out the tradeoff options — never silently resolving it. Without AI: solver runs with equal weights, produces correct output, generic narration. Deterministic core, same split as everywhere else in this product.

**Phase 3 — Portfolio allocation guidance.**
Category-level allocation rules (equity/debt/NPS/PPF/REIT/gold-silver ETF/MF), tenure and contribution-limit logic, AI explanation of the *why*.
- *Added:* **Loan prepayment vs. invest tradeoff** — for users with an active home loan and surplus capacity, the system computes effective loan cost after Section 24b deduction vs. expected post-tax investment return, and presents what each path means. Deterministic math; AI narrates the implications.
- *Added:* **Tax planning gap flagging** — computes how much of the ₹1.5L 80C limit is used, whether 80D is claimed, whether Section 24b interest deduction applies, and runs an old vs. new tax regime comparison. All deterministic. Surfaced as "you may be leaving ₹X unclaimed," not tax advice.
- *Added:* **Preference elicitation for allocation** — a scored risk-tolerance questionnaire (SEBI-aligned, the standard instrument in registered-advisor practice) is introduced here, producing a risk profile that refines the age-based allocation formula for anyone who completes it. The questionnaire stands alone and is sufficient without AI. AI's role is strictly: an optional open-ended follow-up question for edge cases the fixed questionnaire doesn't capture well, reusing the Phase 1 goal-clarification-loop pattern rather than building a new mechanism; flagging contradictions between the questionnaire score and the person's own stated goals (e.g., high-risk score but "I really can't afford to lose this") back to the person — never silently overriding the score; narrating why the resulting allocation follows from their specific profile. Without AI: the questionnaire produces a valid, complete risk profile — the same instrument a SEBI-registered advisor uses.

**Phase 4 — Statement parsing.**
CSV parsing (deterministic), then PDF parsing for one statement format (AI), feeding structured transaction data into the system.
- *Deterministic:* **Input validation and PII masking** — every uploaded file is validated for size, format, and readability before any processing. Account numbers, customer names, and other sensitive identifiers are masked server-side before statement content reaches the parsing agent. Only transaction-level data (date, amount, description) is sent to the model. Reject-and-explain on every failure — no silent errors.
- *Agentic:* **PDF parsing pipeline** — extracting transactions from a PDF is not a single-pass task. The agent identifies the document format, extracts raw rows, categorizes each transaction, validates totals against the statement summary, and flags anomalies — each step informed by the previous. This multi-step structure is what makes it genuinely agentic rather than a single prompt call. The final validation step (totals reconciliation) is deterministic arithmetic, not an AI call.

**Phase 5 — Tracking, variance & alerts.**
Deterministic categorization + AI for ambiguous merchants, variance math, AI root-cause narration when an alert fires.
- *Added:* **Tax planning gap — time-sensitive mode** — once statement data is available, the tax gap alert becomes calendar-aware: fires in Q3/Q4 of the financial year with actual invested amounts from parsed statements, not just intake figures. "3 months left — ₹45,000 of your 80C limit is unclaimed."
- *Agentic:* **Alert root-cause investigation** — when a variance alert fires, the system investigates step by step: check spending trend, check income change, check whether the goal itself was modified. It forms a conclusion only after checking all three, then narrates the actual cause. A single prompt can't do this reliably because the right question to ask next depends on what the previous check found.

**Phase 6 — Adaptive milestones.**
Recompute and re-narrate when a goal changes, reusing Phase 2 and Phase 1 capabilities rather than adding new ones.

**Portfolio extension (labeled, not required):** an evaluation harness checking whether parsed goals and allocation explanations hold up against a held-out set of known-correct cases — proving "the AI is probably right" is measured, not assumed.
- *Added:* **LLM-as-judge for narration quality** *(applies Phase 2 onward)* — the existing harness grades goal parsing and milestone suggestions against known-correct answers using exact or near-exact match. Plain-language narration (feasibility explanations, allocation rationale, root-cause explanations) cannot be graded this way — quality is not binary-correct. This extension adds a separate model call that scores generated narration against a rubric: accuracy to the underlying numbers, clarity, and appropriate tone for the person's stated experience level. The judge model and the product model are separate; the judge's scores accumulate as quality signals over time. This is an evaluation tool only — it does not run in the product path and does not affect what users see.
- *Added:* **RAG-grounded regulatory explanation** *(Phase 3 portfolio extension)* — grounds AI narration for tax-gap explanations in a maintained corpus of current Indian tax rule text and Finance Act circulars, retrieved at prompt time. The tax arithmetic is deterministic and unchanged. This demonstrates retrieval-grounded generation in a low-stakes, measurable context; the harness can verify that the retrieved passage actually supports the generated explanation.
