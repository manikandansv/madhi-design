# HLD — Phase 2 Addition: Multi-Goal Constrained Optimization

## What this document covers

Design of the multi-goal constrained optimization capability that extends Phase 2's feasibility engine (section 3.3 of the product spec). The base feasibility engine (single-goal: given amount, timeline, capacity → can this goal be hit, and if not, what are the alternatives) is a Phase 2 prerequisite and is not re-specified here. This document covers the extension that handles the harder case: multiple goals, one shared surplus, insufficient capacity to fund all of them on their stated timelines.

Phase 1 scope is unchanged. This capability does not begin implementation until Phase 2.

---

## Problem statement

The single-goal feasibility engine answers "can this goal be hit, and if not, what gives?" for one goal at a time. This breaks down when a user has three goals competing for the same ₹30,000/month investable surplus:

- Home purchase: ₹80L in 5 years → requires ₹18,000/month
- Child's education: ₹40L in 10 years → requires ₹9,500/month
- Retirement at 55 (20 years): ₹2.5 crore → requires ₹7,000/month

Total required: ₹34,500/month. Available: ₹30,000/month. Running each feasibility check independently produces three "almost feasible" results that are individually plausible but collectively impossible — the user doesn't see the conflict. This module makes the conflict explicit and computes what can actually be done.

---

## System boundary

```
StructuredGoal[] + UserProfileSnapshot
          │
          ▼
┌─────────────────────────────────────────┐
│  PriorityWeightExtractor   (AI)         │
│  Fuzzy priority statement               │
│  → structured weight vector             │
└────────────────┬────────────────────────┘
                 │ PriorityWeights (or equal weights if no statement given)
                 ▼
┌─────────────────────────────────────────┐
│  MultiGoalSolver           (deterministic)
│  Constrained optimization               │
│  Goal amounts × timelines × weights     │
│  → AllocationPlan or InfeasibilityResult│
└────────────────┬────────────────────────┘
         ┌───────┴────────┐
         ▼                ▼
┌────────────────┐  ┌──────────────────────────┐
│  SolverNarrator│  │  InfeasibilityExplainer  │
│  (AI)          │  │  (AI)                    │
│  Plain-language│  │  Tradeoff options, never  │
│  explanation   │  │  a silent choice          │
└────────┬───────┘  └────────────┬─────────────┘
         └──────────────┬────────┘
                        ▼
              AllocationResult
              shown in plan view
              (human approval required)
```

---

## Module 1 — PriorityWeightExtractor

**Responsibility:** Convert a fuzzy natural-language priority statement into a structured weight vector the solver can use. This is a bounded extraction task — the same class of problem as goal parsing in Phase 1.

**When it runs:** Only if the user provides an explicit priority statement alongside their goals. If no statement is given, the solver receives equal weights by default — this is the correct fallback and requires no AI call.

**Input:**
```
priority_statement: string | null
goals: StructuredGoal[]
```

**Output:**
```
PriorityWeights {
  weights: Map<goal_id, number>   // normalized to sum 1.0
  confidence: "high" | "medium" | "low"
  interpretation: string          // AI's reading of what was stated, shown to user for confirmation
}
```

**Example:** "The house matters more than the trip, but don't push retirement past 55" →
```json
{
  "weights": { "home_purchase": 0.50, "retirement": 0.35, "travel": 0.15 },
  "confidence": "medium",
  "interpretation": "Home purchase treated as highest priority. Retirement protected with a hard constraint at age 55. Travel lowest priority and may be delayed significantly."
}
```

The interpretation is shown to the user before the solver runs. If they disagree, they can adjust. The solver does not run until the weights are confirmed.

**Model:** Claude Haiku. This is extraction against a bounded schema — same class as GoalParser. Reserve Sonnet for tasks requiring genuine judgment.

**What breaks if this module is removed:** Nothing structurally. The solver runs with equal weights — correct, fully functional, generic output. Priority elicitation is a quality-of-output enhancement, not a structural dependency.

---

## Module 2 — MultiGoalSolver

**Responsibility:** Given each goal's amount, timeline, and priority weight, and a total monthly capacity constraint, compute an allocation plan that satisfies as many goals as possible on or near their stated timelines. This is a deterministic constrained optimization problem. It is not an AI call.

**Inputs:**
```
goals: Array<{
  id: string
  target_amount: number           // inflation-adjusted (Phase 2 feasibility engine handles this)
  timeframe_months: number
  priority_weight: number         // from PriorityWeightExtractor, or 1/n if not provided
  minimum_amount: number | null   // optional floor (e.g. "at least 30L for the home, not the full 80L")
}>
monthly_capacity: number
expected_return_rate: number      // conservative annualized rate, from allocation engine
```

**Objective modes (user selects, or default applied):**

| Mode | Objective | Default |
|------|-----------|---------|
| `minimize_total_delay` | Find the allocation that minimizes sum of delays across all goals, weighted by priority | Yes |
| `protect_highest_priority` | Fully fund the highest-priority goal on timeline; delay others with remaining capacity | No |

**Output (feasible case):**
```
AllocationPlan {
  feasible: true
  allocations: Array<{
    goal_id: string
    monthly_contribution: number
    projected_achievement_months: number
    delay_vs_stated_months: number      // 0 = on time, positive = delayed
  }>
  objective_mode: "minimize_total_delay" | "protect_highest_priority"
}
```

**Output (infeasible case):** Triggered when even at minimum goal sizes and maximum delay tolerance, capacity is insufficient.
```
InfeasibilityResult {
  feasible: false
  shortfall_per_month: number
  options: Array<{
    description: string        // e.g. "Drop travel goal entirely — frees ₹4,200/month"
    freed_capacity: number
    remaining_conflict: boolean
  }>
}
```

**Algorithm:** This is a linear programming problem. The implementation can use a standard LP solver (Apache Commons Math, or a purpose-written greedy algorithm for the common cases). The specific solver implementation is an LLD decision — what matters at HLD level is that this is deterministic computation, not an AI call, and its output is reproducible given the same inputs.

**Key tradeoff — solver complexity vs. correctness:** A greedy algorithm (fund in priority order until capacity is exhausted) is fast and produces the `protect_highest_priority` result correctly. It under-serves `minimize_total_delay` for non-trivial cases. An LP solver handles both correctly. Start with the greedy approach for the MVP; upgrade to LP if the `minimize_total_delay` objective is validated as important to users in practice.

---

## Module 3 — SolverNarrator

**Responsibility:** Convert the solver's AllocationPlan into a plain-language explanation of what was decided and why, matched to the person's stated investment experience level.

**Input:** `AllocationPlan` + `UserProfile.investment_experience`

**Output:** A short narrative (3–5 sentences) explaining the tradeoffs the solver made — not just the numbers. Example: "Your home purchase stays on track for 5 years. Retirement savings are slightly reduced but still reach your target by 55. Your travel fund is pushed out by 14 months — the cost of keeping the other two goals on schedule."

**Model:** Claude Haiku. Short, structured narrative. Not a judgment call.

**What breaks if removed:** The AllocationPlan numbers are still shown. The user can read the delay figures. Narration is a quality-of-explanation enhancement, not a structural requirement.

---

## Module 4 — InfeasibilityExplainer

**Responsibility:** When the solver returns `InfeasibilityResult`, explain the tradeoff options in plain language. Crucially: present options, never choose one. The human decides which goal to defer, reduce, or drop.

**Input:** `InfeasibilityResult` + `StructuredGoal[]` + `UserProfile`

**Output:** A plain-language summary of why the problem has no feasible solution at current capacity, followed by the concrete options (already computed by the solver) explained in human terms. Each option must state what is given up and what becomes possible as a result.

**Hard constraint in the prompt:** The AI must not express a preference between the options. It presents consequences only. Framing like "you might want to consider…" or "the best option is…" is prohibited in the prompt contract.

**Model:** Claude Haiku.

**What breaks if removed:** The InfeasibilityResult options are still shown as raw numbers. The user can read the shortfall and the options. Narration is a quality-of-explanation enhancement.

---

## Key tradeoffs

1. **Equal weights as default — confirmed.** If the user provides no priority statement, the solver runs on equal weights. This is the correct behavior — no AI call is made, no assumption is introduced. The output is correct but generic.

2. **Confirmation before solver runs.** The extracted weight vector and its interpretation are shown to the user before the solver executes. This prevents the solver from optimizing for a misread priority and producing a plan the user doesn't recognize.

3. **Infeasibility is never resolved silently.** When no feasible allocation exists, the system surfaces the options and stops. It does not pick the "best" option on the user's behalf. This is a product guardrail, not just a prompt instruction — the UI does not allow advancing to the plan view until the user selects an option explicitly.

4. **`minimum_amount` per goal.** Users can optionally specify a floor for a goal (e.g., "at least ₹30L for the home even if I can't reach 80L"). The solver uses this as a hard constraint — the goal must receive at least this allocation or it is marked infeasible for that objective mode. This prevents the solver from "solving" the problem by allocating ₹500/month to a home purchase goal.

---

## What this module does not do

- It does not recommend which goals to keep or drop — that is the user's decision.
- It does not change the stated goal amounts or timelines without explicit user action.
- It does not run continuously on every session — it runs when the user explicitly invokes the multi-goal view, or when Phase 2 detects that stated goals collectively exceed capacity.
- It does not replace the single-goal feasibility engine for users with only one goal or sufficient capacity for all goals.
