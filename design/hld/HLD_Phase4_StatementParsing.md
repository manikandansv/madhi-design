# HLD — Phase 4: Statement Parsing

## What this document covers

Design of the statement parsing capability (section 3.5 of the product spec). Covers the four sequential modules: input validation, PII masking, PDF parsing agent, and output validation. CSV parsing is deterministic and not detailed here beyond noting that it shares the input validation and PII masking modules — it does not use the parsing agent.

Phase 1–3 scope is unchanged. This capability does not begin implementation until Phase 4.

---

## System boundary

```
User uploads PDF or CSV
          │
          ▼
┌─────────────────────────────────────────┐
│  InputValidator          (deterministic) │
│  Size · format · readability            │
│  → pass or reject-with-explanation      │
└────────────────┬────────────────────────┘
                 │ (only if valid)
                 ▼
┌─────────────────────────────────────────┐
│  PIIMasker               (deterministic) │
│  Account numbers · names · IFSC codes   │
│  → masked document, same structure      │
└────────────────┬────────────────────────┘
                 │ masked content stored + job enqueued
                 ▼
┌─────────────────────────────────────────┐
│  StatementParsingAgent   (agentic AI)   │
│  Identify format → extract rows →       │
│  categorize transactions                │
└────────────────┬────────────────────────┘
                 │ extracted transactions
                 ▼
┌─────────────────────────────────────────┐
│  OutputValidator         (deterministic) │
│  Totals reconciliation against          │
│  statement summary figures              │
│  → accepted or flagged as anomalous     │
└─────────────────────────────────────────┘
```

---

## Module 1 — InputValidator

**Responsibility:** Validate every uploaded file before any storage or processing occurs. This module runs synchronously in the HTTP request handler — it does not enqueue a job. Rejected files return immediately with a plain explanation.

**This is deterministic engineering discipline, not an AI capability.** No model is involved.

**Validation rules:**

| Check | Limit / condition | Rejection message |
|-------|------------------|-------------------|
| File size | Configurable max (default 10 MB) | "File exceeds the maximum allowed size of 10 MB." |
| File format | PDF or CSV only | "Only PDF and CSV files are accepted." |
| PDF readability | File must be parseable (not encrypted, not corrupt) | "This PDF could not be read — it may be password-protected or damaged." |
| CSV structure | Must have a header row; at least one data row | "This CSV file appears to be empty or has no header row." |

**Failure behavior:** Every failure returns a 400 with a specific, human-readable reason. No generic "upload failed" messages. No silent failures. No job is enqueued for a rejected file.

**What breaks if this module is removed:** Malformed or oversized files reach the parsing agent, causing unpredictable failures deep in the pipeline that produce confusing error messages or silent bad output.

---

## Module 2 — PIIMasker

**Responsibility:** Mask sensitive identifiers in the validated document before any content is stored or sent to the AI. This module runs synchronously, immediately after InputValidator passes.

**This is deterministic string replacement, not an AI capability.** The masking rules are fixed and do not depend on model judgment.

**What gets masked:**

| Identifier | Pattern | Replacement |
|-----------|---------|-------------|
| Account numbers (bank) | 9–18 digit sequences in account number context | `ACCT-XXXX` + last 4 digits |
| Card numbers | 16-digit sequences, with or without spaces | `CARD-XXXX-XXXX-XXXX-` + last 4 |
| IFSC codes | 11-character alphanumeric in standard IFSC format | `IFSC-XXXXX` |
| Customer name | Text matching the account holder name field | `[ACCOUNT HOLDER]` |
| Address fields | Lines following address label patterns | `[ADDRESS REDACTED]` |

**Scope:** Account numbers, names, and addresses are masked. Transaction descriptions (merchant names, UPI reference IDs, narration text) are **not** masked — this is the data the parsing agent needs. The distinction is: identifying information about the account holder vs. information about individual transactions.

**Storage constraint:** The masked version is stored. The original unmasked file is not persisted server-side after masking completes. If the original must be retained for user-initiated re-download, it is stored in an encrypted store with a separate access control from the parsing pipeline.

**What breaks if this module is removed:** Account numbers, customer names, and IFSC codes reach the language model in plaintext. This violates the basic trust contract with the user and creates unnecessary PII exposure to a third-party API.

---

## Module 3 — StatementParsingAgent

**Responsibility:** Extract structured transaction records from the masked document. This is an agentic AI call — the next step depends on what the previous step found.

**This is the only module in this pipeline that involves AI.**

**Agent steps:**

```
Step 1: Format identification
  → Examine the first page/rows of the masked document
  → Identify the bank/institution and statement format (e.g., HDFC savings CSV, ICICI PDF)
  → Select the appropriate extraction strategy

Step 2: Row extraction
  → Extract raw rows from the identified format
  → Handle multi-line transaction descriptions
  → Normalize date formats, amount signs (debit vs. credit), and currency symbols

Step 3: Categorization
  → For each extracted transaction, assign a category (merchant lookup first, AI only for what the lookup can't resolve)
  → Emit extracted + categorized transactions

Step 4: Anomaly pre-flagging
  → Before returning, note any rows that could not be parsed (merged cells, unexpected format breaks, missing amounts)
```

**Model:** Claude Sonnet. Multi-step reasoning over variable document formats.

**Failure mode:** If the agent cannot extract a usable set of transactions (e.g., the format is unrecognized), it returns a structured failure with a description of what it found and why extraction failed — not a generic error. The client shows this explanation to the user.

**What breaks if this module is removed:** PDF parsing is not possible — the product falls back to CSV-only. This is the designed fallback; CSV parsing (Module 1 + Module 2 + Module 4 only) is a valid, fully functional subset.

---

## Module 4 — OutputValidator

**Responsibility:** Validate extracted transaction totals against the statement's own summary figures before accepting the output. This module runs after the parsing agent returns.

**This is deterministic arithmetic, not an AI capability.** It is the final step of the agentic pipeline described in Module 3 — not a separate feature.

**Validation checks:**

| Check | How |
|-------|-----|
| Total debits | Sum of extracted debit amounts vs. statement's "Total Debits" figure |
| Total credits | Sum of extracted credit amounts vs. statement's "Total Credits" figure |
| Opening + net = closing | Opening balance + (credits − debits) = closing balance |

**Tolerance:** ±₹1 to account for rounding in statement formatting. Differences larger than ₹1 trigger the anomaly path.

**On mismatch:** The extraction is flagged as anomalous, not silently accepted. The client shows: "Extracted totals don't match the statement summary — some transactions may be missing. You can review and correct them." The user can accept, correct, or re-upload.

**On match:** Extraction is accepted. Transactions are written to MongoDB and emitted to the `transactions.parsed` Kafka topic.

**What breaks if this module is removed:** Partial extractions (where the agent missed some rows) are silently accepted. The user's transaction history is incomplete without any indication, and downstream variance and alert calculations run on wrong numbers.

---

## Key tradeoffs

1. **PII masking at ingestion, not at query time.** Masking happens once, at the moment of upload, before the file is stored. The alternative — masking only when sending to the AI — risks storing unmasked PII server-side. Masking at ingestion means the stored copy is always clean.

2. **Reject-and-explain, never silent failure.** Every failure in InputValidator and OutputValidator returns a specific, human-readable message. This is a product constraint, not just engineering hygiene — users uploading financial documents need to know exactly why a file was rejected, not that "something went wrong."

3. **CSV and PDF share the same first two modules.** InputValidator and PIIMasker are format-agnostic. CSV gets Modules 1 + 2 + 4 (deterministic extraction replaces Module 3). PDF gets all four. This keeps the two paths consistent for validation and PII handling without duplicating logic.

4. **One format first.** The parsing agent is initially built for one PDF format (e.g., HDFC savings account). Adding new formats extends the agent's format identification step and extraction strategy — no new pipeline modules.

---

## What this module does not do

- It does not store the original unmasked file in the parsing pipeline's accessible storage.
- It does not send account numbers, customer names, or IFSC codes to any external API, including the AI provider.
- It does not silently accept a partial extraction — mismatch always surfaces to the user.
- It does not parse investment account statements (mutual fund CAS, demat statements) in this phase — bank and credit card statements only.
