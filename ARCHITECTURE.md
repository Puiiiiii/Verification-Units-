# Architecture: Tester Log Analyzer

## Status and scope

This document describes a proposed architecture for the project, based on the goals in the README. The repository currently contains documentation only; the components and flows below are design guidance, not claims about existing software.

The analyzer is intended to help engineers investigate manufacturing test failures by turning tester logs into traceable, reviewable classifications. It is decision support: the system should preserve the evidence behind a result and make uncertainty visible rather than silently presenting a guess as fact.

### Goals

- Accept tester logs and extract consistent, useful test evidence.
- Classify failures as **True Reject**, **Tester Issue**, **Firmware Issue**, **Limitation**, or **Unknown Failure**.
- Apply deterministic, reviewable rules first; use AI and historical cases as optional supporting evidence.
- Preserve enough provenance to reproduce and audit an analysis.

### Not yet specified

The project has not selected a programming language, deployment model, database, log formats, user interface, or AI provider. These are implementation decisions to make when representative logs and operating constraints are known. Begin with a small local workflow and add services or infrastructure only when a concrete need requires them.

## System context

```text
 Tester / firmware logs
          |
          v
  Input adapter -----> Original input + source metadata
          |
          v
  Parser and normalizer -----> Parse diagnostics
          |
          v
  Structured analysis case
          |
          +------> Versioned deterministic rules ----+
          |                                          |
          +------> Historical case lookup (optional) +--> Decision and evidence
          |                                          |          |
          +------> AI assistance (optional) --------+          v
                                                       Report / export
```

Parsing, rule evaluation, historical lookup, and AI assistance should be separable. The first useful version can omit historical lookup, AI, and a web interface while keeping their future integration points clear.

## Components and responsibilities

### Input adapters

Accept supported files or streams and attach available source metadata, such as product or unit identifier, test station, fixture, lot, test time, and firmware version. An adapter should validate size and encoding, identify the input format, and report unsupported or incomplete input explicitly. Keep the original input available according to the project's data-retention policy so findings can be traced back to source.

### Parser and normalizer

Convert each supported format into a common representation of test runs, measurements, status values, error codes, and timestamps. Format-specific parsing belongs behind an adapter/parser boundary so a new tester format does not require changes to the classifier.

Parsing should be deterministic and report malformed or unrecognized sections as diagnostics. Do not silently drop relevant lines or coerce invalid values. A case may be partially analyzable, but its parse completeness must be visible to later stages and to the report.

### Analysis case and domain model

Use a technology-independent model shared by parsers, rules, and report generation. A minimal case should include:

- Case identifier and input/source metadata, with sensitive identifiers minimized or redacted where appropriate.
- Parsed test events and measurements, including units when known.
- Parse status and diagnostics.
- Analysis timestamp and versions of the parser and rules used.
- Candidate findings, supporting and contradicting evidence, and final outcome.

Keep raw input separate from normalized data. The normalized representation should retain references to source locations (for example, line numbers or event identifiers) so a reviewer can verify evidence.

### Rule engine

Evaluate versioned deterministic rules against normalized cases. A rule should state its matching conditions, classification or finding, evidence references, and rule identifier/version. Rules should be testable independently and produce the same result for the same case and rule version.

Rules may produce multiple candidate findings or no finding. They should not force a confident classification when evidence is incomplete or conflicting. Preserve both supporting and contradictory observations where practical.

### Historical case lookup (optional)

Compare a case with prior analyzed cases using suitable attributes such as product family, station, fixture, firmware, error signature, and time range. Return references and similarity context rather than treating a prior outcome as proof. Historical results need provenance and a way to distinguish confirmed outcomes from unreviewed or provisional labels.

### AI assistance (optional)

AI may summarize evidence, suggest hypotheses, or identify potentially related historical cases. It is advisory and must not replace deterministic parsing, alter source evidence, or silently override a rule outcome. Store the model/provider and prompt or configuration version where permitted, and clearly label generated content. If the AI service is unavailable, the deterministic result should still be reportable.

### Decision and report

Aggregate evidence into a result containing:

- One canonical classification, or `Unknown Failure` when available evidence does not justify another label.
- A concise rationale with references to source evidence and the rules/findings involved.
- Confidence or evidence sufficiency, with a documented meaning; do not imply statistical calibration unless it has been validated.
- Parse warnings, conflicts, and optional AI/historical contributions.
- Analysis, parser, and ruleset versions.

Keep classification separate from workflow status (for example, awaiting human review or confirmed). A reviewer correction should be recorded as a new, attributed decision rather than erasing the original automated result.

## Classification semantics

The five labels are the project's canonical failure categories:

| Classification | Intended interpretation |
| --- | --- |
| True Reject | Evidence supports a genuine product/unit failure. |
| Tester Issue | Evidence points to the tester, station, fixture, or test setup. |
| Firmware Issue | Evidence points to firmware behavior or a firmware-related defect. |
| Limitation | The test or available method cannot reliably distinguish the condition, or a known test limitation applies. |
| Unknown Failure | Evidence is insufficient, ambiguous, or conflicting. |

These are initial descriptions, not a complete adjudication policy. Before implementing production rules, define boundary cases and examples with domain experts. In particular, do not classify a case as a true reject merely because a test failed; require evidence and retain an unknown outcome when the cause cannot be established.

## End-to-end flow

1. Receive an input and record its source metadata.
2. Validate and parse it; retain parse diagnostics and source references.
3. Normalize parsed events into the shared analysis-case model.
4. Evaluate the selected ruleset and, if enabled, retrieve historical context and request AI assistance.
5. Aggregate results into a traceable classification and rationale.
6. Present or export the result; allow an authorized reviewer to confirm or correct it.
7. Retain the case and review history according to the configured retention policy.

Each stage should return an explicit success, partial-success, or failure state. Errors should identify the stage and affected input without exposing credentials or unnecessary sensitive log content.

## Proposed module boundaries

The implementation language and packaging are undecided. A possible language-neutral organization is:

```text
src/
  domain/          # Analysis case, evidence, classifications, result types
  ingestion/       # Input adapters and validation
  parsing/         # Format-specific parsers and normalization
  rules/           # Rule definitions, evaluation, and rule versions
  history/         # Optional historical storage and retrieval
  ai/              # Optional AI integration and output labeling
  analysis/        # Pipeline orchestration and result aggregation
  reporting/       # Human-readable reports and exports
tests/
  fixtures/        # Sanitized representative logs
  parsing/
  rules/
  analysis/
```

This is a suggested boundary map, not a requirement to create every directory up front. Start with the smallest set needed for the first supported log format and keep interfaces explicit where components have different responsibilities.

## Data, privacy, and operations

- Treat manufacturing logs as potentially sensitive. Identify sensitive fields and apply least-privilege access, minimization, and redaction before external processing.
- Do not send raw logs to an AI service unless the organization has approved that data flow. Make AI use configurable and keep the non-AI path functional.
- Define retention, deletion, backup, and access policies before selecting persistent storage.
- Record enough metadata to reproduce decisions, but do not log secrets or duplicate full raw inputs into operational logs.
- If analysis is batch-based, make retries safe and prevent duplicate cases from being mistaken for new test runs.
- Expose parser failures, rule versions, and optional-service availability in diagnostics rather than masking them.

## Testing and quality strategy

- Use sanitized, representative log fixtures for each supported format, including malformed and incomplete inputs.
- Test parsing and normalization independently, including source-location and unit preservation.
- Give each rule positive, negative, and boundary examples; verify deterministic results for a fixed ruleset version.
- Test pipeline behavior when parsing is partial, evidence conflicts, historical lookup fails, or AI is disabled/unavailable.
- Review classification quality against domain-expert-labeled cases before using results operationally. Track false classifications and reviewer corrections.

## Suggested implementation sequence

1. Collect representative, sanitized logs and agree on the initial classification definitions.
2. Define the minimal analysis-case and evidence model.
3. Implement one input adapter/parser and a small set of deterministic rules with fixtures.
4. Produce a traceable local report and capture human review feedback.
5. Add historical storage/search when repeated-case analysis is needed.
6. Evaluate AI assistance separately, with privacy approval and an explicit advisory-only boundary.
7. Add a service, user interface, or deployment automation only when usage requirements are clear.

## Open decisions

- Which tester log formats and encoding/size limits must be supported first?
- Which fields are required for an analysis, and which may be absent?
- Who defines and approves classification rules and reviewer corrections?
- What data may be retained, for how long, and who may access it?
- Is the initial workflow local, batch-oriented, or service-based?
- Are external AI services permitted for any part of the logs?
- What evidence threshold or review policy is required before acting on a classification?
