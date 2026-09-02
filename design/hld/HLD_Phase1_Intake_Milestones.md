# HLD — Phase 1: Intake + Milestone Setting

## What this document covers

Responsibilities, inputs/outputs, and key tradeoffs for the six modules that make up Phase 1. This is the design to confirm before moving to LLD. Nothing here is code.

---

## System boundary

```
User
 │
 ▼
┌──────────────────────────────────────────────────┐
│  IntakeForm          (deterministic)              │
│  All structured fields → typed UserProfile        │
│  Free-text goals → raw_goal_text[]                │
└─────────────────┬────────────────────────────────┘
                  │ UserProfile + raw_goal_text[]
                  ▼
┌──────────────────────────────────────────────────┐
│  GoalParser          (AI)                         │
│  Raw text → structured goal objects               │
└─────────────────┬────────────────────────────────┘
                  │ StructuredGoal[]
          ┌───────┴────────┐
          ▼                ▼
┌──────────────────┐  ┌────────────────────────────┐
│  MilestoneRules  │  │  MilestoneSuggester  (AI)  │
│  (deterministic) │  │  Gaps a rule can't catch   │
└────────┬─────────┘  └─────────────┬──────────────┘
         │                          │
         └──────────────┬───────────┘
                        │ PendingMilestone[]
                        ▼
          ┌─────────────────────────────┐
          │  ApprovalQueue              │
          │  User approves / rejects    │
          └──────────────┬──────────────┘
                         │ ConfirmedMilestone[]
                         ▼
          ┌─────────────────────────────┐
          │  PlanStore                  │
          │  Persists the full Phase 1  │
          │  plan as structured state   │
          └─────────────────────────────┘
```

---

## Architecture pattern — DDD + event-driven workflow, with Akka as a later runtime option

Phase 1 should be modeled as a command-driven workflow with explicit domain events. This keeps the business behavior understandable, testable, and ready for async AI processing without prematurely forcing an actor runtime into the design.

### Domain flow

- `SubmitIntake` is the entry command from the frontend.
- an application workflow handles validation and draft-state creation.
- the goal parsing step transforms each raw goal into `StructuredGoal`.
- deterministic milestone rules generate rule-based milestones.
- the AI suggestion pass adds non-obvious milestones specific to the profile.
- the approval workflow records approve / skip decisions and transitions the plan to `ReadyForPlanView`.
- `PlanStore` persists the final `Phase1Plan` after the approval gate is complete.

### Events in the domain

- `IntakeReceived`
- `GoalsParsed`
- `MilestonesGenerated`
- `MilestoneApproved`
- `MilestoneSkipped`
- `PlanPersisted`

### Why this is the right pattern

- The user journey is stateful and multi-step.
- AI calls are asynchronous and may fail or time out.
- Approval is a domain transition, not just a UI button click.
- This keeps business logic in the domain and makes the flow resilient to future event-stream extensions.

This pattern does not replace the Spring Boot API layer; it defines a clear domain boundary. The API remains the external boundary, while workflow coordination sits in the application/domain layer. Akka can be introduced later if the workflow becomes supervision-heavy or actor-based orchestration becomes necessary.

---

## Module 1 — IntakeForm

**Responsibility:** Collect and validate all user inputs across a structured multi-step form. No inference — what the user states is what goes out. The richer the profile collected here, the more signal the rule engine and the AI have to work with in later modules.

**Inputs:** User interactions across a 7-step form (expanded from the 5-step mock to accommodate background and future commitments).

**Proposed form steps:**

| Step | Label | What's collected |
|------|-------|-----------------|
| 1 | About you | Age, education, employment type + details |
| 2 | Family — present | Spouse, children, parents |
| 3 | Commitments — present | All existing financial, personal, and educational commitments. Mandatory — must enter at least one or explicitly select NIL. |
| 4 | Commitments — future | All planned financial, personal, and educational commitments. Mandatory — must enter at least one or explicitly select NIL. |
| 5 | Money | Monthly income, take-home, living expenses; investable capacity derived from Step 3 |
| 6 | What you have | Current investments and insurance (baseline snapshot) |
| 7 | Where you want to go | Free-text goals, one per entry |

---

### Outputs — UserProfile

```
UserProfile {

  // Personal
  age: number
  location: string                   // context only; never used to infer commitments

  // Education & employment
  highest_education: "high_school" | "diploma" | "undergraduate" |
                     "postgraduate" | "doctorate" | "professional_degree"
  field_of_study: string             // broad label: engineering / medicine / commerce /
                                     // law / arts / other
  employment_type: "salaried_private" | "salaried_govt" | "self_employed" |
                   "business_owner" | "professional" | "not_employed"
  years_of_experience: number
  seniority: "entry" | "mid" | "senior" | "leadership"
  employer_benefits: {
    pf_enrolled: boolean             // EPF via employer
    gratuity_eligible: boolean
    pension_scheme: boolean          // NPS govt, OPS legacy, or none
    health_insurance_employer: boolean
  }
  investment_experience: "beginner" | "intermediate" | "experienced"

  // Present family
  marital_status: "single" | "married" | "widowed" | "separated"
  spouse: {
    employed: boolean
    monthly_income: number | null    // null if not employed / not shared
  } | null
  children: Array<{ age: number }>   // one entry per child
  parents_dependent: boolean         // at least one parent financially dependent
  parent_ages: number[]              // ages of dependent parents
  living_situation: "renting" | "own_home_no_emi" | "own_home_with_emi" | "family_home"
                                     // own_home_with_emi → home loan EMI must appear in present_commitments
                                     // own_home_no_emi / family_home → no housing cost line

  // Present commitments — mandatory; NIL must be explicitly confirmed if none
  // Covers financial (EMIs, SIPs, rent, premiums), personal (family support),
  // and educational (ongoing fees, courses) commitments.
  present_commitments_nil: boolean   // true = user explicitly stated no present commitments
  present_commitments: Array<{
    category: "financial" | "personal" | "educational" | "other"
    label: string                    // user's own label: "home loan EMI", "SIP — MF", etc.
    monthly_amount: number
    ends_in_years: string | null     // raw as entered: "3", "10", "ongoing", null
  }>

  // Future commitments — mandatory; NIL must be explicitly confirmed if none
  // Covers planned financial, personal, and educational commitments.
  future_commitments_nil: boolean    // true = user explicitly stated no future commitments
  future_commitments: Array<{
    category: "financial" | "personal" | "educational" | "other"
    label: string                    // "planned marriage", "child's college", "parent care"
    description: string | null        // optional extra detail
    expected_in_years: string | null // raw as entered: "3", "5-6", "someday", null
    estimated_monthly_impact: number | null
  }>

  // Money — monthly
  monthly_gross_income: number
  income_stability: "very_stable" | "moderate" | "variable" | null
                                     // null for salaried/govt; set only for self_employed, business_owner, professional
  monthly_take_home: number          // post-tax, post-PF — actual cash available
  monthly_living_expenses: number    // stated by user; groceries/utilities/discretionary
  // derived (shown to user, correctable)
  monthly_investable_capacity: number  // take_home − living_expenses − sum(present_commitments monthly)

  // Current investments & insurance (baseline)
  current_assets: {
    epf_balance: number | null
    ppf_balance: number | null
    nps_balance: number | null
    mutual_funds_value: number | null
    fixed_deposits: number | null
    direct_equities_value: number | null
    gold_silver_value: number | null
    real_estate_investment: number | null  // NOT primary residence
  }
  current_insurance: {
    term_cover_amount: number | null       // 0 or null = no term policy
    health_cover_amount: number | null     // own/family floater
    has_life_insurance: boolean            // endowment / ULIP / LIC — note: not term
  }

  // Optional structured fields for future-phase math — collected in Step 6
  // All null if not provided; system degrades gracefully but notes the gap in rationale.

  epf_contributions: {
    monthly_employee_contribution: number  // employee's 12% share (or VPF amount)
    monthly_employer_contribution: number  // employer's 12% share
  } | null

  ppf_annual_contribution: number | null   // yearly PPF deposit amount

  home_loan_details: {
    annual_interest_rate: number           // e.g. 8.5 (percentage)
    outstanding_principal: number
    remaining_tenure_months: number
  } | null                                 // null if no home loan in present_commitments
                                           // shown in Step 6 only when living_situation = "own_home_with_emi"

  tax_details: {
    regime: "old" | "new"                  // current preference
    other_80c_investments: number | null   // ELSS, tax-saver FD, etc. not already in commitments
    hra_applicable: boolean                // whether HRA is part of salary structure
  } | null
}
```

**Key tradeoffs and locked decisions:**

1. *How much of current_assets to collect.* Every field is optional (null = not stated). Collecting all of them risks overwhelming the user. Present as a single "what you currently have" screen with clearly optional fields and a "skip for now" path — the system functions without them, but milestone rationale will note the gap.

2. *Monthly investable capacity — ask vs. derive.* Derived only: `take_home − living_expenses − sum(present_commitments monthly_amount)`. Show the number, let the user correct it. Aspirational answers to a direct question produce a plan built on fiction.

3. *Present and future commitments are mandatory — decided.* The form cannot proceed past Step 3 or Step 4 without either (a) at least one commitment entered, or (b) an explicit "NIL" confirmation. Blank is not valid. This is enforced in form validation. The `present_commitments_nil` and `future_commitments_nil` flags record what the user stated.

4. *Future commitment timeline — raw input — decided.* `expected_in_years` is stored as the user typed it ("3", "5–6 years", "someday"). No conversion to absolute year at save time. The AI uses it as context; Phase 2's feasibility math will handle the interpretation when it is built.

5. *Self-employed income stability — structured select — decided.* A three-option radio: Very stable / Moderate swings / Highly variable. Shown only when `employment_type` is `self_employed`, `business_owner`, or `professional`. Stored as `income_stability` on the profile; passed to the AI suggester so it can calibrate rationale for income risk.

---

### Outputs — raw_goal_text[]

One free-text entry per goal. Separate entry fields (one at a time, with "add another goal" button) rather than a single box for all goals — prevents multi-goal parsing complexity. What the user types is preserved verbatim; GoalParser transforms it.

---

## Module 2 — GoalParser

**Responsibility:** Call the AI with a single raw goal text and return a structured goal object. Run once per raw_goal_text entry. This is a bounded extraction task — the AI pulls out what's stated, not what it thinks the person should want. It has access to UserProfile for disambiguation (age helps interpret "retire early"; having children helps interpret "education fund").

**Input:** one `raw_goal_text` string + `UserProfile`

**Output:**
```
StructuredGoal {
  id: string                         // uuid, generated here
  type: GoalType
  target_amount: number | null       // null if not stated
  timeframe_years: number | null     // null if vague ("someday", "when I can")
  description: string                // AI's one-line parse summary
  source_text: string                // the raw text that produced this, unchanged
  parse_confidence: "high" | "medium" | "low"
}

GoalType: "home_purchase" | "retirement" | "child_education" | "child_marriage" |
          "travel" | "emergency_fund" | "vehicle" | "own_wedding" |
          "parent_care_fund" | "business_startup" | "other"
```

**parse_confidence:**
- `high` — type, amount, and timeframe all cleanly extracted
- `medium` — one field inferred or ambiguous (e.g., amount not stated but type and timeline clear)
- `low` — major gap; proceed only after the user clarifies

**Key tradeoffs:**

1. *Structured output vs. free-form response.* Recommendation: structured output via tool use / response schema. Fragile parsing of free-form JSON is avoidable.

2. *What to do on low-confidence parse — decided: inline.* When `parse_confidence: "low"`, a clarification prompt appears immediately, before the user can proceed past that goal entry. The user sees what the parser extracted and is asked for the missing field. They cannot advance until the gap is resolved. A milestone plan built on an unclear goal is unreliable from the start.

3. *Model.* This is extraction against a bounded schema — Claude Haiku is sufficient and fast. Reserve Sonnet for MilestoneSuggester, which needs judgment.

---

## Module 3 — MilestoneRuleEngine

**Responsibility:** Apply a deterministic checklist against the full profile and parsed goals. Produce flagged milestones for clear, unambiguous gaps. No AI. Runs before the AI call and its output is passed to MilestoneSuggester so the AI doesn't repeat what's already been caught.

**Inputs:** `UserProfile` + `StructuredGoal[]`

**Output:** `PendingMilestone[]` (source: "deterministic")

**Rules for Phase 1:**

| Rule | Condition | Flagged milestone |
|------|-----------|------------------|
| Emergency fund | No goal of type `emergency_fund` | Build 6-month expense buffer |
| Term insurance — with dependents | current_insurance.term_cover_amount is null or 0, AND (children.length > 0 OR parents_dependent OR spouse != null) | Consider term life cover |
| Health coverage | current_insurance.health_cover_amount is null or 0 | Secure health insurance |
| Retirement goal | No goal of type `retirement` | Set a retirement target |
| Child education | children.length > 0 AND no goal of type `child_education` | Fund for child's education |
| Investable capacity gap | monthly_investable_capacity < 0 | Commitments exceed income — flag immediately |
This list is intentionally narrow. The AI's job is the long tail; the rule engine handles the obvious, universal cases only. No new rules without a specific reason.

---

## Module 4 — MilestoneSuggester

**Responsibility:** Call the AI with the full parsed context and ask it to identify non-obvious milestones specific to this person's profile and stated goals — things a fixed rule wouldn't catch. This is the only judgment call in Phase 1.

**Inputs:**
- `UserProfile`
- `StructuredGoal[]`
- `PendingMilestone[]` from MilestoneRuleEngine (so AI doesn't repeat)

**Output:** `PendingMilestone[]` (source: "ai_suggested"), each with:
```
PendingMilestone {
  id: string
  source: "deterministic" | "ai_suggested"
  title: string
  rationale: string   // specific to this person — not generic; language matches experience level
  status: "pending"
}
```

**What the AI has to work with (and therefore can actually use):**

With the richer intake, the AI now has real signal:
- A self-employed person with irregular income → flag income stabilisation or business continuity
- A professional degree holder early in career → likely high student loan or bonding period, flag accordingly
- Parents dependent + no parent care fund goal → surfaceable
- Government employee on OPS → pension secured, no need to flag retirement separately; flag a different gap instead
- Future child planned in 3 years + child education goal not yet set → time-sensitive
- Spouse employed + joint home goal → home loan EMI sizing and dual-income risk (one income goes away)

**Prompt contract (what the AI is asked to do — not the actual prompt text):**
- Review this person's profile, goals, and already-flagged gaps. Identify up to 3 milestones that are specific to their situation and that a fixed rule would not have caught. Explain each in language matched to their investment experience level. Do not repeat what the deterministic rules already flagged (list provided). Rank by relevance.

**Key tradeoffs:**

1. *Cap at 3 suggestions.* Hard cap in the prompt, not a UI filter. The approval step should be manageable — up to 9 milestones maximum (6 rule-based when all conditions fire + 3 AI), but in practice fewer: three of the six rules are conditional (term insurance fires only with dependents; child education fires only with children; capacity gap fires only when capacity is negative).

2. *Run in parallel with rule engine?* The rule engine is synchronous and near-instant. The AI call needs the rule engine's output (to avoid repetition), so the sequence is: rule engine completes → AI call starts (with rule engine output included). In practice the AI call starts ~0ms after intake submission; the rule engine output is ready before the first token comes back.

3. *Rationale quality.* The rationale field is mandatory and must be specific — "consider this because your income is irregular and you have dependent parents" not "consider this for your financial security." Weak rationale gets rejected; strong rationale earns approval. This is enforced in the prompt, not in validation.

---

## Module 5 — ApprovalQueue

**Responsibility:** Present all pending milestones to the user for explicit approve/reject. Nothing enters the confirmed plan without a user action. This is a pure UI-layer module — state transitions only, no logic or AI.

**Input:** `PendingMilestone[]` (combined: deterministic + AI-suggested)

**Output:** `ConfirmedMilestone[]` (approved, with source preserved) + `SkippedMilestone[]` (recorded, not discarded)

**Behavior:**
- All milestones shown together, labeled by source.
- Approve/reject are explicit — no default-approve.
- Rationale always visible (not behind a toggle). The user needs to know why before deciding.
- Skipped milestones are stored — they may matter in Phase 6 (adaptive milestones) and knowing what was explicitly declined prevents re-surfacing it.
- **Approval gate:** the "See my plan" navigation is blocked until every card has been either approved or skipped. No card can remain in `pending` state. This is the product guardrail — nothing enters the plan without an explicit decision.

Note on the will/nomination milestone: this was a deterministic rule but was dropped because no "estate planning indicator" field exists in the profile. It is instead surfaced as an AI-suggested milestone when `age >= 35` — the AI has access to age and can include it in its up-to-3 suggestions when relevant.

**Key tradeoff:** *All on one screen vs. one at a time.* With visible rationale per card, showing them together lets the user see the full picture before approving anything. One-at-a-time is better for high-stakes decisions with long explanations — these are short, specific, and comparable to each other. Show them together.

---

## Module 6 — PlanStore

**Responsibility:** Persist the complete Phase 1 output as structured state so Phase 2 can pick it up without re-running intake.

**What it stores:**
```
Phase1Plan {
  profile: UserProfile
  goals: StructuredGoal[]
  milestones: ConfirmedMilestone[]
  skipped_milestones: SkippedMilestone[]
  created_at: ISO8601 string
  version: "1"
}
```

**Life event timeline:** the plan view renders goals and future commitments on a single forward-time axis — no new data collection required, everything is already in `Phase1Plan`. This is a display rendering of `goals[].timeframe_years` and `future_commitments[].expected_in_years`, not a separate module. It belongs in the Phase 1 plan screen and makes goal conflicts visible (e.g., home purchase and planned marriage overlapping in the same 3-year window) before Phase 2's feasibility math runs.

**Phase 1 persistence:** localStorage only. No backend, no auth, no sync. This keeps Phase 1 self-contained and avoids adding auth complexity before the core product logic exists. The tradeoff (plan lost on browser clear) is acceptable for Phase 1.

**Note on the wireframe prototype:** commitment NIL enforcement (Screens 3–4) is intentionally not wired — the prototype is for flow validation, not acceptance testing, and mandatory-field blocking would make it unnavigable for review. Approval gating (Screen 8 → 9 blocked until all cards decided) and the life event timeline (Screen 9) are both wired in the mock. This HLD is the source of truth for NIL validation behavior.

**Key tradeoff:** *localStorage vs. thin backend with UUID-keyed plan.* Recommendation: localStorage for now with a clean export-to-JSON path, so data isn't trapped when we add persistence later. Add the backend in Phase 3 or 4 when the plan has enough depth that device-local is a real problem.

---

## Stack recommendation

**Frontend:** React + TypeScript. The index.html mock is a visual reference only — Phase 1 has multi-step form state, async AI call sequencing, and an approval flow. Vanilla JS becomes a maintenance problem here; React gives explicit state and component isolation.

**AI calls:** Claude API (Haiku for GoalParser, Sonnet for MilestoneSuggester), routed through the Spring Boot backend. API keys must never live in client code. A Java + Spring Boot service with two endpoints (`POST /api/parse-goal`, `POST /api/suggest-milestones`) is all Phase 1 needs — not a full backend, just a key relay.

**No other infrastructure in Phase 1.** No database, no auth, no deployment pipeline. Run locally.

---

## What Phase 1 explicitly does not include

- Feasibility math (Phase 2)
- Allocation guidance (Phase 3)
- Statement parsing (Phase 4)
- Variance tracking (Phase 5)
- Adaptive milestone recomputation (Phase 6)
- Multi-device sync or user accounts

---

## Decisions locked (from open questions)

| # | Question | Decision |
|---|----------|----------|
| 1 | Goal clarification flow | Inline — blocks progress past that goal until resolved |
| 2 | Self-employed income variance | Structured select: Very stable / Moderate swings / Highly variable |
| 3 | Future commitment timeline | Raw input stored as entered; AI interprets contextually |
| 4 | Commitments mandatory? | Yes — both present and future. Blank is invalid; NIL must be explicitly stated |

## Minimum required fields before the form can submit

All of these must be filled; everything else is optional:

- `age`
- `employment_type`
- `monthly_take_home`
- `monthly_living_expenses`
- `present_commitments` resolved (at least one entry OR `present_commitments_nil: true`)
- `future_commitments` resolved (at least one entry OR `future_commitments_nil: true`)
- At least one goal entered and parsed with confidence `medium` or above
