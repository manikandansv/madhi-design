# HLD — Phase 3 Addition: Preference Elicitation for Allocation

## What this document covers

Design of the preference elicitation capability that extends Phase 3's portfolio allocation guidance (section 3.4 of the product spec). The base allocation engine (age-based asset-class buckets, contribution-limit logic, AI narration of the why) is a Phase 3 prerequisite and is not re-specified here. This document covers the scored risk-tolerance questionnaire and the AI jobs that extend it.

Phase 1 scope is unchanged. This capability does not begin implementation until Phase 3.

---

## Problem statement

The base allocation engine uses age, timeline, and goal type to assign an asset-class mix. This is correct for the median case. It breaks down when two people with identical demographics have genuinely different risk tolerance — one who would stay calm through a 30% drawdown and one who would sell at the first sign of loss. Treating them the same produces an allocation that fits neither well.

The standard practice for this in regulated financial advice (SEBI-registered investment advisors are required to conduct this) is a scored risk-tolerance questionnaire. The questionnaire produces a risk profile that adjusts the base allocation formula. This is not a new idea and not an AI problem — it is standard financial planning practice implemented as a deterministic scoring mechanism.

---

## System boundary

```
UserProfile (from Phase 1 intake)
          │
          ▼
┌────────────────────────────────────────────┐
│  RiskQuestionnaire         (deterministic) │
│  Fixed scored questionnaire                │
│  → RiskScore (numeric) + RiskBand          │
└──────────────┬─────────────────────────────┘
               │ RiskScore + responses
       ┌───────┴────────────────────────┐
       ▼                                ▼
┌────────────────────┐  ┌──────────────────────────────┐
│  FollowUpElicitor  │  │  ContradictionDetector  (AI) │
│  (AI, optional)    │  │  Score vs. stated goals      │
│  Reuses Phase 1    │  │  Surfaces conflict to user   │
│  clarification loop│  │  Never overrides score       │
└────────┬───────────┘  └──────────────┬───────────────┘
         └──────────────────┬──────────┘
                            │ Confirmed RiskProfile
                            ▼
          ┌────────────────────────────────┐
          │  AllocationEngine  (deterministic)
          │  Base formula + RiskProfile    │
          │  → adjusted allocation mix     │
          └──────────────────┬────────────┘
                             │
                             ▼
          ┌────────────────────────────────┐
          │  RiskProfileNarrator  (AI)     │
          │  Why this allocation follows   │
          │  from this specific profile    │
          └────────────────────────────────┘
```

---

## Module 1 — RiskQuestionnaire

**Responsibility:** Administer a fixed, scored questionnaire that produces a numeric risk score and a risk band. This is fully deterministic — no AI. The questionnaire stands alone and is sufficient to produce a valid risk profile.

**Design:** The questionnaire covers five dimensions, consistent with what SEBI guidelines prescribe for risk profiling:

| Dimension | Sample question |
|-----------|----------------|
| Investment horizon | "When do you expect to need a significant portion of this money?" |
| Loss tolerance | "If your portfolio dropped 20% in a year, what would you do?" |
| Income stability | "How stable is your primary income over the next 3–5 years?" |
| Financial dependents | "How many people depend on your income?" |
| Prior behavior in downturns | "Have you ever sold investments during a market drop?" |

Each question has 3–5 answer options, each mapped to a point value. Total score → risk band:

| Score range | Risk band | Allocation direction |
|-------------|-----------|---------------------|
| 0–25 | Conservative | Higher debt weighting; lower equity exposure |
| 26–50 | Moderately conservative | Balanced; debt slightly overweighted |
| 51–70 | Moderate | Standard age-based formula applies |
| 71–85 | Moderately aggressive | Equity overweighted vs. age formula |
| 86–100 | Aggressive | Maximum equity exposure; debt only for liquidity |

**Output:**
```
RiskScore {
  total: number                     // 0–100
  band: "conservative" | "moderately_conservative" | "moderate" |
        "moderately_aggressive" | "aggressive"
  dimension_scores: Map<string, number>   // for explainability and contradiction detection
  responses: Map<question_id, answer_id>  // verbatim for audit and AI context
}
```

**What breaks if this module is removed:** The base age-based allocation formula is used, as it was before Phase 3. Correct but undifferentiated for risk tolerance. No capability is lost that didn't exist before Phase 3.

---

## Module 2 — FollowUpElicitor (optional)

**Responsibility:** For cases where the fixed questionnaire produces a result that doesn't seem to fully capture the person's situation — typically detectable from low within-dimension consistency or from a score very close to a band boundary — ask a single open-ended follow-up question and incorporate the answer.

**This is an optional enhancement, not a structural requirement.** The questionnaire result is valid without this step. The follow-up is surfaced only when it is likely to be materially useful.

**Implementation:** Reuse the goal-clarification-loop pattern from Phase 1. This is a 2-turn loop: the AI reviews the questionnaire responses, identifies the dimension with lowest consistency or the boundary ambiguity, asks one targeted question. The person's answer is appended to the RiskScore context and used by ContradictionDetector and RiskProfileNarrator. No new mechanism is built for this.

**Model:** Claude Haiku. Bounded extraction, same class as goal clarification.

**Trigger condition (deterministic, checked before AI call):** Run the follow-up only if:
- Any single-dimension score is more than 30 points away from the overall mean score, OR
- Total score falls within ±5 points of a band boundary

Otherwise skip. The AI is not in the path for the common case.

**What breaks if removed:** The questionnaire result stands as-is. No material loss for the majority of users whose scores are consistent and not boundary-adjacent.

---

## Module 3 — ContradictionDetector

**Responsibility:** Compare the questionnaire score against the person's stated goals (from Phase 1 intake) and any phrasing in goal descriptions that signals strong loss aversion. Surface contradictions to the user. Never silently override the score.

**Input:** `RiskScore` + `StructuredGoal[]` (including `source_text` — the original free-text)

**Detection cases:**

| Contradiction | Example |
|---------------|---------|
| High-risk score + loss-averse phrasing in goal | Score: aggressive. Goal text: "I really can't afford to lose this — it's my daughter's wedding fund." |
| Conservative score + aggressive timeline expectation | Score: conservative. Goal: "Double my money in 3 years." |
| Score inconsistency within questionnaire | Loss-tolerance answers: very high. Income stability answers: very low. (Handled by FollowUpElicitor, not this module.) |

**Output (if contradiction found):**
```
Contradiction {
  type: "score_vs_goal_phrasing" | "score_vs_timeline"
  score_band: RiskBand
  conflicting_goal_id: string
  conflicting_phrase: string     // verbatim excerpt from goal text
  surfaced_question: string      // what to show the user: "Your questionnaire suggests aggressive risk tolerance, but you wrote '[phrase]' for your [goal]. Which better reflects how you feel about this goal?"
}
```

**Resolution:** The user explicitly chooses. Options:
1. Keep the questionnaire score, override the phrasing concern
2. Accept a lower risk band for the specific goal (goal-level risk profile)
3. Retake the questionnaire

The AI does not pick a resolution. The UI does not proceed to allocation until the contradiction is resolved.

**Model:** Claude Haiku. Pattern matching against bounded input — goal text is already structured from Phase 1 parsing.

**What breaks if removed:** The questionnaire score is used as-is. Contradictions between score and goal phrasing are not surfaced. The allocation is still valid — it reflects the questionnaire result, which is what the person formally answered. No structural loss; edge cases where stated behavior contradicts formal answers are not caught.

---

## Module 4 — RiskProfileNarrator

**Responsibility:** Explain, in plain language matched to the person's investment experience level, why the resulting allocation mix follows from their specific risk profile — not just their age.

**Input:** `RiskScore` + `AllocationMix` (from allocation engine) + `UserProfile.investment_experience`

**Output:** A short narrative (3–5 sentences). Example: "Your questionnaire puts you in the moderate-aggressive band — you're comfortable with equity volatility and have a 15-year horizon before you need this money. That's why this allocation runs heavier on equity than a standard formula for your age would suggest. The debt allocation covers your 3-year home purchase goal, where you genuinely can't afford short-term drawdown."

**Model:** Claude Haiku. Short, structured narrative.

**What breaks if removed:** The allocation numbers are shown without explanation. For users with intermediate or experienced investment background, this is sufficient. For beginners, it's a meaningful quality gap but not a functional one.

---

## Key tradeoffs

1. **Questionnaire is the primary mechanism — confirmed.** The AI jobs (FollowUpElicitor, ContradictionDetector, RiskProfileNarrator) are enhancements, not dependencies. The questionnaire produces a complete, valid risk profile without any AI involvement. This is intentional — it mirrors regulated advisor practice and ensures the capability degrades gracefully.

2. **Goal-level vs. portfolio-level risk profile.** The default is a single risk profile applied to the overall portfolio allocation. ContradictionDetector may surface a case where a specific goal warrants a different treatment (e.g., the wedding fund is conservative while the retirement corpus is aggressive). The system supports goal-level risk profile overrides but does not require them — the UI surfaces this as an option when a contradiction is detected, not as a default workflow.

3. **Questionnaire retake.** The user can retake the questionnaire. Each retake replaces the previous score. No averaging — the most recent score is authoritative. This is consistent with SEBI guidance (periodic risk re-profiling).

4. **Score is transparent.** The numeric score, the band, and each dimension's contribution are shown to the user — not just the resulting band label. This is a guardrail against the system feeling like a black box that assigns a category. The person can see exactly what produced their result.

---

## What this module does not do

- It does not recommend specific funds or products — that remains category-level allocation only.
- It does not override the user's explicit decision when they resolve a contradiction.
- It does not run on every session — it runs once at Phase 3 entry and on explicit retake. The stored RiskProfile is used for subsequent sessions unless retaken.
- It does not infer risk tolerance from behavior (e.g., from statement data) — that would require Phase 4 data and introduces an entirely different class of inference risk. Questionnaire only.
