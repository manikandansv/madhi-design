# LLD — Phase 1: Intake + Milestone Setting

This document is the design authority for code. It covers exact TypeScript types, function signatures, API contracts, form state machine, prompt structure, and edge cases. Do not begin implementation without confirming this document.

---

## 0. Architecture Model — DDD + event-driven workflow

Phase 1 is intentionally structured as a command-driven domain workflow. The UI sends commands, the application/domain layer handles orchestration, and the result is emitted as events that update the plan state. This keeps the business model clean without prematurely overcommitting to an actor runtime.

### Command/event model

```scala
sealed trait IntakeCommand
case class SubmitIntake(profile: UserProfile, goals: List[String]) extends IntakeCommand
case class ParseGoal(rawText: String) extends IntakeCommand
case class GenerateMilestones(profile: UserProfile, goals: List[StructuredGoal]) extends IntakeCommand
case class ApproveMilestone(milestoneId: String, userId: String) extends IntakeCommand
case class SkipMilestone(milestoneId: String, reason: String, userId: String) extends IntakeCommand

sealed trait IntakeEvent
case class IntakeReceived(planId: String) extends IntakeEvent
case class GoalsParsed(goals: List[StructuredGoal]) extends IntakeEvent
case class MilestonesGenerated(milestones: List[PendingMilestone]) extends IntakeEvent
case class MilestoneApproved(id: String, approvedAt: Instant) extends IntakeEvent
case class MilestoneSkipped(id: String, skippedAt: Instant) extends IntakeEvent
case class Phase1PlanPersisted(planId: String) extends IntakeEvent
```

### Workflow responsibilities

- `IntakeWorkflow`: validates and coordinates the overall session flow
- `GoalParserService`: parses each raw goal to a structured goal object
- `MilestoneRuleEngine`: runs deterministic rule checks
- `MilestoneSuggestionService`: executes AI-generated non-obvious milestone suggestions
- `ApprovalWorkflow`: records approve / skip transitions and enforces the gate
- `PlanPersistenceService`: writes the final `Phase1Plan` after approval completion

### Messaging principles

- Commands are imperative: they ask the domain to do something.
- Events are descriptive: they reflect what has already happened.
- No business rule sits in a controller or a React component.
- AI provider failures are treated as recoverable workflow failures with timeout and retry handling in the infrastructure layer.

This is the preferred model for the early implementation. Akka can be introduced later if the orchestration, timeout handling, or supervision requirements become sufficiently complex to justify an actor runtime.

---

## 1. Shared Types

TypeScript types for the client only. Lives at `client/src/types/`. The Java server has equivalent records in `com/madhi/model/` — client and server are independent projects with no shared type library.

### 1.1 UserProfile

```typescript
// types/UserProfile.ts

export type EducationLevel =
  | "high_school"
  | "diploma"
  | "undergraduate"
  | "postgraduate"
  | "doctorate"
  | "professional_degree";

export type FieldOfStudy =
  | "engineering"
  | "medicine"
  | "commerce"
  | "law"
  | "arts"
  | "other";

export type EmploymentType =
  | "salaried_private"
  | "salaried_govt"
  | "self_employed"
  | "business_owner"
  | "professional"
  | "not_employed";

export type Seniority = "entry" | "mid" | "senior" | "leadership";

export type InvestmentExperience = "beginner" | "intermediate" | "experienced";

export type MaritalStatus = "single" | "married" | "widowed" | "separated";

export type LivingSituation =
  | "renting"
  | "own_home_no_emi"
  | "own_home_with_emi"
  | "family_home";

export type CommitmentCategory =
  | "financial"
  | "personal"
  | "educational"
  | "other";

export type IncomeStability = "very_stable" | "moderate" | "variable";

export interface PresentCommitment {
  id: string;                             // client-generated uuid
  category: CommitmentCategory;
  label: string;
  monthly_amount: number;
  ends_in_years: string | null;           // raw: "3", "10", "ongoing", null
}

export interface FutureCommitment {
  id: string;                             // client-generated uuid
  category: CommitmentCategory;
  label: string;
  description: string | null;
  expected_in_years: string | null;       // raw: "3", "5-6", "someday", null
  estimated_monthly_impact: number | null;
}

export interface UserProfile {
  // Personal
  age: number;
  location: string;                       // context only; never used to infer commitments

  // Education & employment
  highest_education: EducationLevel;
  field_of_study: FieldOfStudy;
  employment_type: EmploymentType;
  years_of_experience: number;
  seniority: Seniority;
  employer_benefits: {
    pf_enrolled: boolean;
    gratuity_eligible: boolean;
    pension_scheme: boolean;
    health_insurance_employer: boolean;
  };
  investment_experience: InvestmentExperience;

  // Income stability — set only for self_employed | business_owner | professional
  income_stability: IncomeStability | null;

  // Family
  marital_status: MaritalStatus;
  spouse: {
    employed: boolean;
    monthly_income: number | null;
  } | null;
  children: Array<{ age: number }>;
  parents_dependent: boolean;
  parent_ages: number[];
  living_situation: LivingSituation;

  // Commitments
  present_commitments_nil: boolean;
  present_commitments: PresentCommitment[];

  future_commitments_nil: boolean;
  future_commitments: FutureCommitment[];

  // Money
  monthly_gross_income: number;
  monthly_take_home: number;
  monthly_living_expenses: number;
  monthly_investable_capacity: number;    // derived: take_home − living_expenses − sum(present monthly)

  // Current assets — all optional
  current_assets: {
    epf_balance: number | null;
    ppf_balance: number | null;
    nps_balance: number | null;
    mutual_funds_value: number | null;
    fixed_deposits: number | null;
    direct_equities_value: number | null;
    gold_silver_value: number | null;
    real_estate_investment: number | null;
  };

  // Insurance
  current_insurance: {
    term_cover_amount: number | null;
    health_cover_amount: number | null;
    has_life_insurance: boolean;
  };

  // Optional — for future-phase math; null if not provided
  epf_contributions: {
    monthly_employee_contribution: number;
    monthly_employer_contribution: number;
  } | null;
  ppf_annual_contribution: number | null;
  home_loan_details: {
    annual_interest_rate: number;
    outstanding_principal: number;
    remaining_tenure_months: number;
  } | null;
  tax_details: {
    regime: "old" | "new";
    other_80c_investments: number | null;
    hra_applicable: boolean;
  } | null;
}
```

---

### 1.2 Goals

```typescript
// types/Goal.ts

export type GoalType =
  | "home_purchase"
  | "retirement"
  | "child_education"
  | "child_marriage"
  | "travel"
  | "emergency_fund"
  | "vehicle"
  | "own_wedding"
  | "parent_care_fund"
  | "business_startup"
  | "other";

export type ParseConfidence = "high" | "medium" | "low";

export interface StructuredGoal {
  id: string;                             // uuid generated server-side
  type: GoalType;
  target_amount: number | null;           // null if not stated by user
  timeframe_years: number | null;         // null if vague ("someday", "when I can")
  description: string;                    // AI's one-line parse summary
  source_text: string;                    // the raw text unchanged
  parse_confidence: ParseConfidence;
  clarification_needed: string | null;    // present when confidence = "low"; the specific question
}
```

---

### 1.3 Milestones

```typescript
// types/Milestone.ts

export type MilestoneSource = "deterministic" | "ai_suggested";
export type MilestoneStatus = "pending" | "approved" | "skipped";

export interface PendingMilestone {
  id: string;                             // uuid
  source: MilestoneSource;
  title: string;
  rationale: string;                      // specific to this person; matches their experience level
  status: "pending";
}

export interface ConfirmedMilestone extends Omit<PendingMilestone, "status"> {
  status: "approved";
  approved_at: string;                    // ISO8601
}

export interface SkippedMilestone extends Omit<PendingMilestone, "status"> {
  status: "skipped";
  skipped_at: string;                     // ISO8601
}
```

---

### 1.4 Plan

```typescript
// types/Plan.ts

import { UserProfile } from "./UserProfile";
import { StructuredGoal } from "./Goal";
import { ConfirmedMilestone, SkippedMilestone } from "./Milestone";

export interface Phase1Plan {
  profile: UserProfile;
  goals: StructuredGoal[];
  milestones: ConfirmedMilestone[];
  skipped_milestones: SkippedMilestone[];
  created_at: string;                     // ISO8601
  version: "1";
}
```

---

## 2. Form State Machine

### 2.1 Steps

```typescript
// types/FormState.ts

export type FormStep = 1 | 2 | 3 | 4 | 5 | 6 | 7;

export interface FormState {
  currentStep: FormStep;
  profile: Partial<UserProfile>;
  rawGoals: string[];                     // one entry per goal as the user typed it
  validationErrors: Partial<Record<FormStep, string[]>>;
}
```

### 2.2 Step Transition Rules

Step forward is blocked unless all required fields for the current step are valid.

| Step | Required to advance |
|------|---------------------|
| 1 | `age`, `employment_type`, `years_of_experience`, `seniority`, `investment_experience`, `highest_education`, `field_of_study` |
| 2 | `marital_status`, `living_situation`, `parents_dependent`. If `parents_dependent: true` → `parent_ages` must be non-empty. If `marital_status = "married"` → `spouse` must be set. |
| 3 | `present_commitments_nil: true` OR `present_commitments.length >= 1` with every entry having `category`, `label`, and `monthly_amount > 0` |
| 4 | `future_commitments_nil: true` OR `future_commitments.length >= 1` with every entry having `category` and `label` |
| 5 | `monthly_take_home > 0`, `monthly_living_expenses >= 0`. `monthly_investable_capacity` recomputed on every change. |
| 6 | No required fields — fully optional. |
| 7 | `rawGoals.length >= 1`. Each entry must be non-empty string. |

### 2.3 Derived Field — investable_capacity

Recomputed client-side on every change to take_home, living_expenses, or any present_commitment amount:

```typescript
function deriveInvestableCapacity(profile: Partial<UserProfile>): number {
  const takeHome = profile.monthly_take_home ?? 0;
  const expenses = profile.monthly_living_expenses ?? 0;
  const commitments = (profile.present_commitments ?? []).reduce(
    (sum, c) => sum + c.monthly_amount,
    0
  );
  return takeHome - expenses - commitments;
}
```

Shown inline as the user edits Step 5. Can go negative — a negative value triggers the "capacity gap" rule in MilestoneRuleEngine but does not block form submission.

---

## 3. API Contracts

### 3.1 POST /parse-goal

**Request:**
```typescript
interface ParseGoalRequest {
  raw_text: string;
  profile: UserProfile;
}
```

**Response — success:**
```typescript
interface ParseGoalResponse {
  goal: StructuredGoal;
}
```

**Response — error:**
```typescript
interface ApiError {
  error: string;
  code: "VALIDATION_ERROR" | "AI_PARSE_FAILED" | "PROVIDER_ERROR";
}
```

**Behavior:**
- Called once per `rawGoals` entry after Step 7 is submitted.
- Calls are sequential (each parse result feeds back to the UI before the next is called) so the user can resolve clarifications inline.
- If `parse_confidence = "low"` → the response still returns 200 with the goal object. `clarification_needed` is set to the specific question. Client renders the clarification prompt before advancing.
- HTTP 500 with `PROVIDER_ERROR` if the AI call fails; client shows a retry option.

---

### 3.2 POST /generate-milestones

**Request:**
```typescript
interface GenerateMilestonesRequest {
  profile: UserProfile;
  goals: StructuredGoal[];
}
```

**Response — success:**
```typescript
interface GenerateMilestonesResponse {
  milestones: PendingMilestone[];   // deterministic first, then ai_suggested (max 3); ordered by source
}
```

**Behavior:**
- Server runs `MilestoneRuleEngine` first, producing deterministic milestones.
- Deterministic output is passed to the AI prompt so suggestions don't repeat already-flagged items.
- Response combines both: deterministic milestones appear first, AI-suggested milestones follow.
- HTTP 500 with `PROVIDER_ERROR` if the AI call fails; deterministic milestones are still returned.

---

## 4. MilestoneRuleEngine — Server-Side (Java)

Runs inside `POST /generate-milestones` before the AI call. Pure deterministic logic — no I/O, no AI, testable in isolation.

```java
// com/madhi/milestones/MilestoneRuleEngine.java

package com.madhi.milestones;

import com.madhi.model.PendingMilestone;
import com.madhi.model.UserProfile;
import com.madhi.model.StructuredGoal;
import org.springframework.stereotype.Component;

import java.text.NumberFormat;
import java.util.*;
import java.util.stream.Collectors;

@Component
public class MilestoneRuleEngine {

    public List<PendingMilestone> run(UserProfile profile, List<StructuredGoal> goals) {
        List<PendingMilestone> results = new ArrayList<>();
        Set<String> goalTypes = goals.stream()
            .map(StructuredGoal::getType)
            .collect(Collectors.toSet());

        // Rule 1: Emergency fund
        if (!goalTypes.contains("emergency_fund")) {
            double buffer = profile.getMonthlyLivingExpenses() * 6;
            results.add(milestone(
                "Build an emergency fund",
                "A 6-month buffer (\u20b9" + formatINR(buffer) + ") covers income disruption without touching investments. No goal for this was found."
            ));
        }

        // Rule 2: Term insurance — dependents present, no cover
        boolean hasDependents = !profile.getChildren().isEmpty()
            || profile.isParentsDependent()
            || profile.getSpouse() != null;
        if (hasDependents && profile.getCurrentInsurance().getTermCoverAmount() == null) {
            results.add(milestone(
                "Secure term life cover",
                "You have dependents and no term policy. If income stops, there is no safety net for them."
            ));
        }

        // Rule 3: Health coverage
        if (profile.getCurrentInsurance().getHealthCoverAmount() == null) {
            results.add(milestone(
                "Get health insurance",
                "No health cover found. A single hospitalisation can erase months of savings."
            ));
        }

        // Rule 4: No retirement goal
        if (!goalTypes.contains("retirement")) {
            results.add(milestone(
                "Set a retirement target",
                "No retirement goal was entered. Starting the estimate early gives compounding time to work."
            ));
        }

        // Rule 5: Children present, no education goal
        if (!profile.getChildren().isEmpty() && !goalTypes.contains("child_education")) {
            String who = profile.getChildren().size() > 1 ? "children" : "a child";
            results.add(milestone(
                "Start a child education fund",
                "You have " + who + " but no education fund goal. Education costs compound faster than inflation."
            ));
        }

        // Rule 6: Investable capacity gap
        if (profile.getMonthlyInvestableCapacity() < 0) {
            double gap = Math.abs(profile.getMonthlyInvestableCapacity());
            results.add(milestone(
                "Commitments exceed take-home income",
                "Monthly commitments exceed take-home by \u20b9" + formatINR(gap) + ". No savings are possible until this is resolved."
            ));
        }

        return results;
    }

    private PendingMilestone milestone(String title, String rationale) {
        return PendingMilestone.builder()
            .id(UUID.randomUUID().toString())
            .source("deterministic")
            .title(title)
            .rationale(rationale)
            .status("pending")
            .build();
    }

    private String formatINR(double amount) {
        return NumberFormat.getNumberInstance(new Locale("en", "IN"))
            .format((long) amount);
    }
}
```

---

## 5. AI Wrapper — Server Side (Java)

### 5.1 Provider Interface

```java
// com/madhi/ai/AIProvider.java

package com.madhi.ai;

import java.util.List;
import java.util.Map;

public interface AIProvider {
    Object complete(String systemPrompt, String userMessage, Map<String, Object> schema)
        throws AIProviderException;

    AgentResult runAgent(String systemPrompt, List<AgentTool> tools, String initialMessage)
        throws AIProviderException;
}
```

```java
// com/madhi/ai/AIProviderException.java

package com.madhi.ai;

public class AIProviderException extends Exception {
    public AIProviderException(String message) { super(message); }
    public AIProviderException(String message, Throwable cause) { super(message, cause); }
}
```

### 5.2 ClaudeProvider

```java
// com/madhi/ai/provider/ClaudeProvider.java

package com.madhi.ai.provider;

import com.anthropic.client.AnthropicClient;
import com.anthropic.client.okhttp.AnthropicOkHttpClient;
import com.anthropic.models.*;
import com.madhi.ai.AIProvider;
import com.madhi.ai.AIProviderException;
import com.madhi.ai.AgentResult;
import com.madhi.ai.AgentTool;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.Map;

@Component
@ConditionalOnProperty(name = "ai.provider", havingValue = "claude", matchIfMissing = true)
public class ClaudeProvider implements AIProvider {

    private final AnthropicClient client;
    private final String haiku  = "claude-haiku-4-5-20251001";
    private final String sonnet = "claude-sonnet-4-6";

    public ClaudeProvider(@Value("${claude.api-key}") String apiKey) {
        this.client = AnthropicOkHttpClient.builder().apiKey(apiKey).build();
    }

    @Override
    public Object complete(String systemPrompt, String userMessage, Map<String, Object> schema)
            throws AIProviderException {
        try {
            var response = client.messages().create(MessageCreateParams.builder()
                .model(sonnet)
                .maxTokens(1024)
                .system(systemPrompt)
                .addUserMessage(userMessage)
                .addTool(Tool.builder()
                    .name("structured_output")
                    .description("Return structured JSON matching the provided schema")
                    .inputSchema(ToolInputSchema.builder().putAllAdditionalProperties(schema).build())
                    .build())
                .toolChoice(ToolChoiceToolChoiceAny.builder().build())
                .build());

            return response.content().stream()
                .filter(b -> b instanceof ContentBlockToolUse)
                .map(b -> ((ContentBlockToolUse) b).input())
                .findFirst()
                .orElseThrow(() -> new AIProviderException("Model did not return a tool_use block"));

        } catch (AIProviderException e) {
            throw e;
        } catch (Exception e) {
            throw new AIProviderException("Claude API call failed", e);
        }
    }

    @Override
    public AgentResult runAgent(String systemPrompt, List<AgentTool> tools, String initialMessage)
            throws AIProviderException {
        // Implemented in Phase 4 — stub for now
        throw new UnsupportedOperationException("AgentRunner not active until Phase 4");
    }
}
```

### 5.3 PromptBuilder

```java
// com/madhi/ai/PromptBuilder.java

package com.madhi.ai;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.core.io.Resource;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import java.io.IOException;
import java.nio.charset.StandardCharsets;

@Component
public class PromptBuilder {

    @Value("classpath:ai/context/indian-finance.md")
    private Resource contextResource;

    private String financeContext;

    @PostConstruct
    public void loadContext() throws IOException {
        this.financeContext = contextResource.getContentAsString(StandardCharsets.UTF_8);
    }

    public String buildGoalParsePrompt(String experienceLevel) {
        return """
            You are a financial planning assistant for Indian personal finance.

            %s

            Your task: extract a structured goal from the user's free-text input.

            The user's investment experience level is: %s.
            - beginner: they are new to investing; use plain language in descriptions
            - intermediate: they understand MF/FD basics; standard terms are fine
            - experienced: they know asset classes, compounding; you can be precise

            Rules:
            1. Extract only what is stated. Do not infer unstated amounts or timelines.
            2. Set parse_confidence to "low" if type, amount, OR timeframe cannot be determined.
            3. When confidence is "low", set clarification_needed to a single, specific question.
            4. Set clarification_needed to null when confidence is "high" or "medium".
            5. target_amount must be in INR (number only, no commas or symbols).
            6. timeframe_years must be a number (integer or decimal). Use null if vague.
            """.formatted(financeContext, experienceLevel).strip();
    }

    public String buildMilestoneSuggestPrompt(String experienceLevel) {
        return """
            You are a financial planning assistant for Indian personal finance.

            %s

            Your task: review this person's financial profile, their parsed goals, and the
            milestones already flagged by deterministic rules. Identify up to 3 additional
            milestones specific to their situation that a fixed rule would not have caught.

            The user's investment experience level is: %s.
            - beginner: plain language; avoid jargon; explain why this matters
            - intermediate: standard terms are fine; brief rationale
            - experienced: precise; reference specific products/structures if relevant

            Rules:
            1. Do not repeat or restate any milestone already in the deterministic list (passed in the request).
            2. Each suggestion must be specific — reference their actual profile data.
            3. Rank by relevance. Hard cap: 3 milestones. Return fewer if warranted.
            4. Rationale must answer: why does THIS matter for THIS person specifically.
            """.formatted(financeContext, experienceLevel).strip();
    }
}
```

### 5.4 OutputParser

```java
// com/madhi/ai/OutputParser.java

package com.madhi.ai;

import com.madhi.model.StructuredGoal;
import com.madhi.model.PendingMilestone;
import org.springframework.stereotype.Component;

import java.util.*;

@Component
public class OutputParser {

    private static final Set<String> VALID_GOAL_TYPES = Set.of(
        "home_purchase", "retirement", "child_education", "child_marriage",
        "travel", "emergency_fund", "vehicle", "own_wedding",
        "parent_care_fund", "business_startup", "other"
    );

    private static final Set<String> VALID_CONFIDENCE = Set.of("high", "medium", "low");

    @SuppressWarnings("unchecked")
    public StructuredGoal parseGoalOutput(Object raw, String sourceText) throws OutputParseException {
        if (!(raw instanceof Map)) throw new OutputParseException("Goal output is not an object");
        var r = (Map<String, Object>) raw;

        String type = (String) r.get("type");
        if (!VALID_GOAL_TYPES.contains(type))
            throw new OutputParseException("Invalid goal type: " + type);

        String confidence = (String) r.get("parse_confidence");
        if (!VALID_CONFIDENCE.contains(confidence))
            throw new OutputParseException("Invalid parse_confidence: " + confidence);

        return StructuredGoal.builder()
            .id(UUID.randomUUID().toString())
            .type(type)
            .targetAmount(r.get("target_amount") instanceof Number n ? n.doubleValue() : null)
            .timeframeYears(r.get("timeframe_years") instanceof Number n ? n.doubleValue() : null)
            .description(String.valueOf(r.getOrDefault("description", "")))
            .sourceText(sourceText)
            .parseConfidence(confidence)
            .clarificationNeeded(r.get("clarification_needed") instanceof String s ? s : null)
            .build();
    }

    @SuppressWarnings("unchecked")
    public List<PendingMilestone> parseMilestoneOutput(Object raw) throws OutputParseException {
        if (!(raw instanceof List)) throw new OutputParseException("Milestone output is not an array");
        var list = (List<Object>) raw;

        return list.stream().limit(3).map(item -> {
            if (!(item instanceof Map)) throw new RuntimeException("Milestone item is not an object");
            var m = (Map<String, Object>) item;

            String title = (String) m.get("title");
            String rationale = (String) m.get("rationale");
            if (title == null || title.isBlank()) throw new RuntimeException("Milestone missing title");
            if (rationale == null || rationale.isBlank()) throw new RuntimeException("Milestone missing rationale");

            return PendingMilestone.builder()
                .id(UUID.randomUUID().toString())
                .source("ai_suggested")
                .title(title)
                .rationale(rationale)
                .status("pending")
                .build();
        }).toList();
    }
}
```

---

## 6. Route Handlers (Spring Boot)

### 6.1 Request / Response Models

```java
// com/madhi/model/request/ParseGoalRequest.java

package com.madhi.model.request;

import com.fasterxml.jackson.annotation.JsonProperty;
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

public record ParseGoalRequest(
    @JsonProperty("raw_text") @NotBlank @Size(max = 500) String rawText,
    @Valid UserProfileDto profile
) {}
```

```java
// com/madhi/model/request/SuggestMilestonesRequest.java

public record SuggestMilestonesRequest(
    @Valid UserProfileDto profile,
    List<StructuredGoalDto> goals,
    @JsonProperty("deterministic_milestones") List<MilestoneTitleDto> deterministicMilestones
) {}
```

```java
// com/madhi/model/ApiError.java
public record ApiError(String error, String code) {}
```

### 6.2 POST /api/parse-goal

```java
// com/madhi/controller/GoalController.java

package com.madhi.controller;

import com.madhi.ai.AIProvider;
import com.madhi.ai.AIProviderException;
import com.madhi.ai.OutputParser;
import com.madhi.ai.OutputParseException;
import com.madhi.ai.PromptBuilder;
import com.madhi.model.ApiError;
import com.madhi.model.StructuredGoal;
import com.madhi.model.request.ParseGoalRequest;
import com.madhi.validation.GoalSchemas;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import jakarta.validation.Valid;

@RestController
@RequestMapping("/api")
public class GoalController {

    private final AIProvider aiProvider;
    private final PromptBuilder promptBuilder;
    private final OutputParser outputParser;

    public GoalController(AIProvider aiProvider, PromptBuilder promptBuilder, OutputParser outputParser) {
        this.aiProvider = aiProvider;
        this.promptBuilder = promptBuilder;
        this.outputParser = outputParser;
    }

    @PostMapping("/parse-goal")
    public ResponseEntity<?> parseGoal(@RequestBody @Valid ParseGoalRequest request) {
        String systemPrompt = promptBuilder.buildGoalParsePrompt(
            request.profile().investmentExperience()
        );

        String userMessage = """
            Profile context:
            - Age: %d
            - Employment: %s
            - Children: %d

            Goal to parse:
            "%s"
            """.formatted(
            request.profile().age(),
            request.profile().employmentType(),
            request.profile().children().size(),
            request.rawText()
        ).strip();

        Object rawOutput;
        try {
            rawOutput = aiProvider.complete(systemPrompt, userMessage, GoalSchemas.GOAL_OUTPUT_SCHEMA);
        } catch (AIProviderException e) {
            return ResponseEntity.internalServerError()
                .body(new ApiError("AI provider failed", "PROVIDER_ERROR"));
        }

        try {
            StructuredGoal goal = outputParser.parseGoalOutput(rawOutput, request.rawText());
            return ResponseEntity.ok(Map.of("goal", goal));
        } catch (OutputParseException e) {
            return ResponseEntity.internalServerError()
                .body(new ApiError("Could not parse AI response", "AI_PARSE_FAILED"));
        }
    }
}
```

### 6.3 POST /api/suggest-milestones

```java
// com/madhi/controller/MilestoneController.java

@RestController
@RequestMapping("/api")
public class MilestoneController {

    private final AIProvider aiProvider;
    private final PromptBuilder promptBuilder;
    private final OutputParser outputParser;

    public MilestoneController(AIProvider aiProvider, PromptBuilder promptBuilder, OutputParser outputParser) {
        this.aiProvider = aiProvider;
        this.promptBuilder = promptBuilder;
        this.outputParser = outputParser;
    }

    @PostMapping("/suggest-milestones")
    public ResponseEntity<?> suggestMilestones(@RequestBody @Valid SuggestMilestonesRequest request) {
        String systemPrompt = promptBuilder.buildMilestoneSuggestPrompt(
            request.profile().investmentExperience()
        );

        String alreadyFlagged = request.deterministicMilestones().stream()
            .map(m -> "- " + m.title())
            .collect(Collectors.joining("\n"));

        String userMessage = """
            Profile:
            %s

            Parsed goals:
            %s

            Milestones already flagged (do not repeat):
            %s
            """.formatted(
            toJson(request.profile()),
            request.goals().stream().map(g -> "- " + g.type() + ": " + g.description()).collect(Collectors.joining("\n")),
            alreadyFlagged.isBlank() ? "None" : alreadyFlagged
        ).strip();

        Object rawOutput;
        try {
            rawOutput = aiProvider.complete(systemPrompt, userMessage, GoalSchemas.MILESTONE_OUTPUT_SCHEMA);
        } catch (AIProviderException e) {
            return ResponseEntity.internalServerError()
                .body(new ApiError("AI provider failed", "PROVIDER_ERROR"));
        }

        try {
            var milestones = outputParser.parseMilestoneOutput(rawOutput);
            return ResponseEntity.ok(Map.of("milestones", milestones));
        } catch (OutputParseException e) {
            return ResponseEntity.internalServerError()
                .body(new ApiError("Could not parse AI response", "AI_PARSE_FAILED"));
        }
    }
}
```

---

## 7. AI Output Schemas (Java)

```java
// com/madhi/validation/GoalSchemas.java

package com.madhi.validation;

import java.util.List;
import java.util.Map;

public final class GoalSchemas {

    public static final Map<String, Object> GOAL_OUTPUT_SCHEMA = Map.of(
        "type", "object",
        "properties", Map.of(
            "type", Map.of("type", "string", "enum", List.of(
                "home_purchase", "retirement", "child_education", "child_marriage",
                "travel", "emergency_fund", "vehicle", "own_wedding",
                "parent_care_fund", "business_startup", "other"
            )),
            "target_amount",        Map.of("type", List.of("number", "null")),
            "timeframe_years",      Map.of("type", List.of("number", "null")),
            "description",          Map.of("type", "string"),
            "parse_confidence",     Map.of("type", "string", "enum", List.of("high", "medium", "low")),
            "clarification_needed", Map.of("type", List.of("string", "null"))
        ),
        "required", List.of("type", "description", "parse_confidence")
    );

    public static final Map<String, Object> MILESTONE_OUTPUT_SCHEMA = Map.of(
        "type", "array",
        "maxItems", 3,
        "items", Map.of(
            "type", "object",
            "properties", Map.of(
                "title",    Map.of("type", "string"),
                "rationale", Map.of("type", "string")
            ),
            "required", List.of("title", "rationale")
        )
    );

    private GoalSchemas() {}
}
```

---

## 8. Client API Layer

```typescript
// client/src/api/intake.ts

import { UserProfile } from "../types/UserProfile";
import { StructuredGoal } from "../types/Goal";
import { PendingMilestone } from "../types/Milestone";

const BASE = import.meta.env.VITE_API_BASE ?? "";

export async function parseGoal(
  rawText: string,
  profile: UserProfile
): Promise<{ goal: StructuredGoal } | { error: string; code: string }> {
  const res = await fetch(`${BASE}/api/parse-goal`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ raw_text: rawText, profile }),
  });
  return res.json();
}

export async function suggestMilestones(
  profile: UserProfile,
  goals: StructuredGoal[],
  deterministicMilestones: PendingMilestone[]
): Promise<{ milestones: PendingMilestone[] } | { error: string; code: string }> {
  const res = await fetch(`${BASE}/api/suggest-milestones`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      profile,
      goals,
      deterministic_milestones: deterministicMilestones,
    }),
  });
  return res.json();
}
```

---

## 9. PlanStore — localStorage

```typescript
// client/src/store/PlanStore.ts

import { Phase1Plan } from "../types/Plan";

const STORAGE_KEY = "madhi_phase1_plan";

export function savePlan(plan: Phase1Plan): void {
  localStorage.setItem(STORAGE_KEY, JSON.stringify(plan));
}

export function loadPlan(): Phase1Plan | null {
  const raw = localStorage.getItem(STORAGE_KEY);
  if (!raw) return null;
  try {
    return JSON.parse(raw) as Phase1Plan;
  } catch {
    return null;
  }
}

export function clearPlan(): void {
  localStorage.removeItem(STORAGE_KEY);
}

export function exportPlan(plan: Phase1Plan): void {
  const blob = new Blob([JSON.stringify(plan, null, 2)], {
    type: "application/json",
  });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `madhi-plan-${new Date().toISOString().split("T")[0]}.json`;
  a.click();
  URL.revokeObjectURL(url);
}
```

---

## 10. Edge Cases

| Scenario | Behavior |
|----------|----------|
| `parse_confidence = "low"` | Client blocks advancement past the goal entry. Renders clarification_needed question. User answers; `/parse-goal` called again with amended text. |
| All milestones skipped | Allowed — Phase1Plan.milestones = []. Plan view shows zero confirmed milestones. No error. skipped_milestones preserved for Phase 6. |
| `monthly_investable_capacity < 0` | Rule engine flags it. Approval queue shows it prominently. User can still proceed — product does not block; it surfaces the problem. |
| `/parse-goal` returns PROVIDER_ERROR | Client shows inline error on the goal card with "Retry" button. Does not advance. Previous goals already parsed are preserved. |
| `/suggest-milestones` returns PROVIDER_ERROR | Client shows deterministic milestones only in ApprovalQueue. A banner notes: "Some AI suggestions couldn't be loaded — you can proceed with what's shown." Does not block. |
| `home_loan_details` not provided when `living_situation = "own_home_with_emi"` | Step 6 shows home loan section as optional. If skipped, Phase 2's prepayment tradeoff feature notes "home loan details needed for this calculation" when it runs. |
| `tax_details` null | No regression in Phase 1. Tax planning feature in Phase 3 will surface the gap with a prompt to fill it in. |
| Zero children, `child_education` goal entered | Allowed — user may be planning for a future child. GoalParser treats it as valid; Rule 5 does not re-flag since the goal type exists. |
| `rawGoals` length = 0 at Step 7 submit | Form validation catches this before API call. Step 7 cannot submit with no goals. |
| Goal text > 500 characters | Server returns VALIDATION_ERROR. Client shows "Please keep your goal description under 500 characters." |

---

## 11. Environment Variables

```properties
# server/src/main/resources/application.properties — committed, no secrets
server.port=8080
ai.provider=${AI_PROVIDER:claude}
claude.api-key=${CLAUDE_API_KEY}
claude.model=claude-sonnet-4-6
openai.api-key=${OPENAI_API_KEY}
```

```bash
# server/.env — never committed (loaded via env or IDE run config)
AI_PROVIDER=claude
CLAUDE_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...           # needed only if AI_PROVIDER=openai

# client/.env.local — never committed
VITE_API_BASE=http://localhost:8080
```

---

## 12. What Changes in LLD Phase 2

Nothing in this LLD changes in Phase 2. Phase 2 adds new routes (`/compute-feasibility`, `/suggest-alternatives`) and extends `Phase1Plan` with a `FeasibilityResult` — it does not touch the types, routes, or AI contracts defined here.

---

## Decisions locked in this LLD

| Decision | Rationale |
|----------|-----------|
| MilestoneRuleEngine runs client-side | Deterministic math; no server round-trip needed; output immediately available to pass to AI call |
| AI calls are sequential per goal, not batched | Allows inline clarification between goals; a batch call cannot surface `clarification_needed` per-goal without complication |
| ClaudeProvider uses tool_use for structured output | Forces schema adherence; avoids fragile JSON extraction from free-form text |
| OutputParser throws on schema violations | Malformed goal/milestone entering the plan is worse than a visible error |
| `ai.provider` Spring property selects implementation | Zero code change to swap providers; `@ConditionalOnProperty` on each impl isolates the switch |
| Skipped milestones are stored, not deleted | Required for Phase 6 adaptive milestones — knowing what was declined prevents re-surfacing |
