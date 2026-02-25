---
layout: page
title: "Week 10 — Building the Agent Pipeline"
description: "Connecting multiple agents, defining workflows, and implementing the full capstone pipeline in your chosen domain."
permalink: /level-2/week-10/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 10'
---

## From Single Agent to Pipeline

In Week 9, you deployed a single agent with a knowledge base and action groups. That agent could reason, use tools, and retrieve documents. It was impressive as a demo.

It is not a production system.

Production agent systems rarely consist of a single agent doing everything. They are **pipelines** — sequences of specialized agents, each responsible for one stage of a workflow, passing structured data to the next. This week, you build the full pipeline for your capstone project.

---

## Why Pipelines, Not Monoliths?

You could, in theory, write one massive system prompt that handles your entire workflow. Here is why you should not:

| Monolith Agent | Pipeline of Agents |
|---------------|-------------------|
| One prompt handles everything | Each agent has a focused, testable prompt |
| Failure at any point kills the whole request | Failures are isolated to one stage |
| Hard to insert human review | Natural breakpoints between stages |
| One model size fits all | Use cheap/fast models for simple steps, capable models for complex ones |
| Debugging means reading one giant log | Each stage has its own input, output, and trace |
| Changes to one step risk breaking others | Stages are independently deployable |

The pipeline architecture maps directly to the prompt chaining pattern from Week 8 — but now you are implementing it with real AWS infrastructure.

---

## Pipeline Architecture

Every capstone domain follows the same fundamental architecture:

```
[Input] → [Stage 1] → [Stage 2] → [Stage 3] → ... → [Output]
              ↓            ↓            ↓
          [Validate]   [Validate]   [Validate]
              ↓            ↓            ↓
          [Log/Store]  [Log/Store]  [Log/Store]
```

Between any two stages, you can insert:
- **Validation** — check that the previous stage's output is well-formed and reasonable
- **Human review** — route to a human for approval before proceeding
- **Branching** — route to different next stages based on the output
- **Error handling** — retry, fall back, or escalate if something goes wrong

### Domain-Specific Pipelines

Here is how the pipeline maps to each capstone domain:

#### Document Processing

```
PDF Upload
  → Stage 1: Parse & Extract Text (Lambda + Textract)
  → Stage 2: Summarize Key Findings (Claude agent)
  → Stage 3: Classify & Tag (Claude Haiku agent)
  → Stage 4: Human Review (approval gate)
  → Stage 5: Generate Final Report (Claude agent)
  → Output: Structured summary with metadata
```

#### Legal Review

```
Contract Upload
  → Stage 1: Parse Document (Lambda + Textract)
  → Stage 2: Clause Extraction (Claude agent + knowledge base)
  → Stage 3: Risk Assessment per Clause (Claude agent + CoT)
  → Stage 4: Cross-Reference Precedent (knowledge base search)
  → Stage 5: Human Review — Legal Team (approval gate)
  → Stage 6: Generate Risk Report (Claude agent)
  → Output: Annotated contract with risk ratings
```

#### File Operations

```
File Drop to S3 Bucket
  → Stage 1: Detect & Classify File Type (Claude Haiku agent)
  → Stage 2: Route Based on Classification (Step Functions choice)
  → Stage 3a: Transform — Invoice Processing (Claude agent)
  → Stage 3b: Transform — Report Processing (Claude agent)
  → Stage 3c: Transform — Correspondence Filing (Claude Haiku agent)
  → Stage 4: Validate Output (Lambda)
  → Stage 5: Deliver to Destination (Lambda)
  → Output: Processed files in correct locations
```

#### Customer Support

```
Incoming Ticket
  → Stage 1: Triage & Categorize (Claude Haiku agent)
  → Stage 2: Retrieve Relevant Context (knowledge base)
  → Stage 3: Draft Response (Claude agent)
  → Stage 4: Tone & Policy Check (guardrails + Claude Haiku)
  → Stage 5: Human Approval (approval gate for high-priority)
  → Stage 6: Send Response (Lambda + API)
  → Output: Verified customer response
```

---

## Connecting Agents

Each stage in your pipeline is a separate agent invocation. The critical engineering challenge is **passing data between stages** reliably.

### Data Contract Between Stages

Every stage must have a clearly defined input and output schema. Document these explicitly:

```
Stage 2: Clause Extraction
  Input: {
    "document_id": "string",
    "raw_text": "string",
    "document_type": "CONTRACT | AMENDMENT | ADDENDUM"
  }
  Output: {
    "document_id": "string",
    "clauses": [
      {
        "id": "string",
        "text": "string",
        "type": "LIABILITY | INDEMNITY | TERMINATION | IP | CONFIDENTIALITY | OTHER",
        "section_number": "string",
        "page_number": integer
      }
    ],
    "extraction_confidence": float,
    "processing_time_ms": integer
  }
```

This is the contract. Stage 2's output schema **must** match Stage 3's input schema. If they drift, the pipeline breaks.

### Handling Mismatches

What happens when Stage 2 produces output that Stage 3 cannot process?

1. **Schema validation** — before passing data to the next stage, validate it against the expected schema. Catch malformed data immediately.
2. **Retry with guidance** — if the output is close but not quite right (e.g., missing a required field), retry the prompt with explicit correction instructions.
3. **Default values** — for non-critical fields, define sensible defaults rather than failing.
4. **Escalation** — if validation fails after retries, route to human review with the raw output attached.

---

## Implementing Your Pipeline

### Step 1: Define Your Stages

Write down every stage of your pipeline. For each stage, specify:

- **Purpose** — one sentence describing what this stage does
- **Input schema** — exact JSON structure expected
- **Output schema** — exact JSON structure produced
- **Model** — which foundation model to use (Sonnet for complex reasoning, Haiku for simple tasks)
- **Tools** — which action groups this stage needs
- **Knowledge base** — whether this stage needs RAG access
- **Failure mode** — what happens if this stage fails

### Step 2: Build Each Stage Independently

Implement and test each stage in isolation:

1. Write the agent instructions (system prompt) for the stage.
2. Create test inputs that cover normal cases and edge cases.
3. Run the stage against all test inputs.
4. Validate outputs against the expected schema.
5. Measure accuracy, latency, and cost.

Do not connect stages until each one works reliably on its own.

### Step 3: Connect the Stages

Now wire the stages together:

1. Stage 1 output feeds into Stage 2 input.
2. Add validation between each pair of stages.
3. Add logging at every transition point.
4. Run end-to-end tests with real inputs.
5. Verify the final output is correct and complete.

### Step 4: Add Error Handling

For each stage transition, implement:

- **Retry logic** — how many times to retry a failed stage, with what backoff
- **Timeout handling** — maximum time allowed per stage before escalation
- **Partial failure** — if processing a batch, handle individual item failures without killing the entire pipeline
- **Circuit breakers** — if a stage fails repeatedly, stop sending requests and alert

### Step 5: Add Human Review Gates

Identify the points in your pipeline where human verification adds the most value:

- After high-stakes decisions (legal risk ratings, financial categorizations)
- When agent confidence is below a threshold
- When the input is unusual or outside the training distribution
- Before any irreversible action (sending an email, filing a document, making a payment)

Implement review gates as asynchronous holds — the pipeline pauses, notifies a human, and resumes when approved.

---

## Testing the Full Pipeline

### End-to-End Test Cases

Build at least 10 end-to-end test cases for your pipeline. Each should include:

- A realistic input document or request
- Expected output at every stage (not just the final output)
- Expected human review triggers (if any)
- Expected error handling behavior for known edge cases

### Stress Testing

- **Volume** — run 50+ inputs through the pipeline concurrently. Do stages bottleneck?
- **Malformed input** — what happens when the input is garbage? Does the pipeline fail gracefully?
- **Large input** — what happens when the document is 100 pages? Does the pipeline handle token limits?
- **Ambiguous input** — what happens when the classification is genuinely uncertain? Does the pipeline escalate correctly?

---

## Key Takeaways

1. **Pipelines beat monoliths.** Decompose your workflow into focused, independently testable stages.
2. **Data contracts are law.** Define input and output schemas for every stage. Validate at every transition.
3. **Match models to tasks.** Use capable models for complex reasoning and cheap models for simple routing.
4. **Build bottom-up.** Get each stage working independently before connecting them.
5. **Design for failure.** Every stage will fail eventually. Retries, timeouts, circuit breakers, and human escalation are not optional — they are core architecture.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/week-09/">← Week 9: AWS Bedrock — Agents & Knowledge Bases</a> · Next: <a href="/level-2/week-11/">Week 11: Deployment & Observability →</a>
</p>
