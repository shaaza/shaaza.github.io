---
layout: page
title: "Week 11 — Deployment & Observability"
description: "Lambda, Step Functions, API Gateway. Logging, tracing, and cost tracking. Error handling patterns for production agent systems."
permalink: /level-2/week-11/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 11'
---

## The Last Mile

You have a working agent pipeline. It processes inputs, chains through multiple stages, calls tools, retrieves from knowledge bases, and produces correct outputs — at least on your test cases.

Now you deploy it for real.

Deployment is where most agent projects fail. Not because the prompts are wrong or the models are bad, but because the engineering around them is missing. No one thought about what happens when the Lambda times out. No one instrumented the pipeline to show which stage is slow. No one noticed the cost was $0.50 per request until the monthly bill arrived.

This week, you solve all of that.

---

## Deployment Architecture

Your capstone pipeline runs on three AWS services that work together:

### AWS Lambda

Lambda runs your code without managing servers. Each stage of your pipeline is a Lambda function that:

1. Receives input (from Step Functions, API Gateway, or another Lambda)
2. Calls the Bedrock agent or model
3. Validates the output
4. Returns structured results

**Key Lambda considerations for agent pipelines:**

| Setting | Recommendation | Why |
|---------|---------------|-----|
| **Timeout** | 5–10 minutes per stage | LLM calls are slow. Default 3 seconds will fail. |
| **Memory** | 512 MB–1 GB | More memory = more CPU = faster JSON parsing and validation |
| **Concurrency** | Set reserved concurrency | Prevent runaway invocations from burning your Bedrock quota |
| **Retry policy** | 0 automatic retries | You handle retries in your code with backoff, not Lambda's default retry |
| **Dead letter queue** | Configure SQS DLQ | Failed invocations must go somewhere you can inspect them |

### AWS Step Functions

Step Functions orchestrate your pipeline — they define the order of stages, handle branching, manage retries, and coordinate human review gates.

A Step Functions state machine for a document processing pipeline:

```json
{
  "StartAt": "ParseDocument",
  "States": {
    "ParseDocument": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:parse-document",
      "Next": "Summarize",
      "Retry": [{"ErrorEquals": ["States.ALL"], "MaxAttempts": 2, "BackoffRate": 2}],
      "Catch": [{"ErrorEquals": ["States.ALL"], "Next": "HandleError"}]
    },
    "Summarize": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:summarize",
      "Next": "ClassifyRisk"
    },
    "ClassifyRisk": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:classify-risk",
      "Next": "CheckConfidence"
    },
    "CheckConfidence": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.confidence",
          "NumericLessThan": 0.8,
          "Next": "HumanReview"
        }
      ],
      "Default": "GenerateReport"
    },
    "HumanReview": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sqs:sendMessage.waitForTaskToken",
      "Next": "GenerateReport"
    },
    "GenerateReport": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:generate-report",
      "End": true
    },
    "HandleError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:handle-error",
      "End": true
    }
  }
}
```

**Why Step Functions, not just chaining Lambdas directly?**

- **Visual debugging** — the Step Functions console shows exactly which state your execution is in
- **Built-in retry and error handling** — configure retries, timeouts, and catch blocks declaratively
- **Human review gates** — the `waitForTaskToken` pattern pauses execution until a human approves
- **State persistence** — the execution state survives Lambda cold starts and failures
- **Audit trail** — every state transition is logged automatically

### API Gateway

API Gateway exposes your pipeline as an HTTP endpoint that external systems can call:

```
POST /api/process-document
Content-Type: application/json

{
  "document_url": "s3://my-bucket/contracts/contract-2024-001.pdf",
  "processing_type": "legal_review",
  "callback_url": "https://my-app.com/webhooks/processing-complete"
}
```

**Configuration considerations:**

- **Throttling** — set rate limits to prevent abuse and control costs
- **Authentication** — use API keys, IAM, or Cognito depending on your audience
- **Request validation** — validate the request body against a schema before invoking the pipeline
- **Asynchronous processing** — for long pipelines, return a 202 (Accepted) with a tracking ID, then notify via callback when complete

---

## Observability

A pipeline you cannot observe is a pipeline you cannot fix. Observability has three pillars: logging, tracing, and metrics.

### Logging

Every stage of your pipeline must log:

```json
{
  "timestamp": "2025-01-15T14:23:01.456Z",
  "execution_id": "exec-abc123",
  "stage": "classify_risk",
  "event": "agent_invocation",
  "input_tokens": 1523,
  "output_tokens": 287,
  "model_id": "anthropic.claude-3-5-sonnet",
  "latency_ms": 3421,
  "cost_usd": 0.0089,
  "output_valid": true,
  "confidence": 0.92
}
```

**Logging rules:**

1. **Use structured logging (JSON).** Not printf-style strings. Structured logs are searchable and aggregatable.
2. **Include the execution ID in every log entry.** This lets you trace a single request across all stages.
3. **Log input and output sizes, not content.** For cost analysis. Log full content only in debug mode — it is expensive to store and may contain sensitive data.
4. **Log at stage boundaries.** Entry and exit of each stage, with timing.
5. **Log all errors with full context.** The error message alone is useless. Include the input that caused it, the stage that failed, and the state of the pipeline at failure time.

### Tracing

Tracing follows a single request through every stage of the pipeline. Use **AWS X-Ray** to visualize the full journey:

```
Request received (API Gateway)          0 ms
  └── ParseDocument (Lambda)            0-850 ms
        └── Textract call              120-780 ms
  └── Summarize (Lambda)               850-4200 ms
        └── Bedrock invocation         920-4150 ms
  └── ClassifyRisk (Lambda)            4200-7600 ms
        └── Bedrock invocation         4280-7100 ms
        └── Knowledge base query       7100-7450 ms
  └── GenerateReport (Lambda)          7600-11200 ms
        └── Bedrock invocation         7680-11100 ms
Total: 11200 ms
```

With this trace, you can immediately see:

- The Summarize stage takes 3.35 seconds — is the prompt too long?
- Bedrock invocations dominate latency — can you use a faster model for some stages?
- The knowledge base query adds 350ms — is it returning relevant results?

### Cost Tracking

LLM calls cost money. At scale, small inefficiencies compound into real expenses.

**Track cost per invocation:**

```
Cost breakdown per pipeline execution:
  ParseDocument:   $0.00  (no LLM call)
  Summarize:       $0.012 (1523 input + 287 output tokens, Sonnet)
  ClassifyRisk:    $0.004 (890 input + 52 output tokens, Haiku)
  GenerateReport:  $0.018 (2100 input + 650 output tokens, Sonnet)
  Total:           $0.034 per document
```

**Cost optimization strategies:**

1. **Use Haiku for simple tasks.** Classification, routing, and format validation do not need Sonnet.
2. **Minimize input tokens.** Trim unnecessary context. Do not send the entire document when a summary suffices.
3. **Cache repeated queries.** If the same knowledge base query is made frequently, cache the results.
4. **Set cost alerts.** CloudWatch alarms that fire if daily spend exceeds a threshold.
5. **Monitor token usage trends.** Increasing token counts over time may indicate prompt drift or unnecessary verbosity.

---

## Error Handling Patterns

Agent pipelines fail in ways that traditional software does not. The model might hallucinate, refuse a valid request, or produce valid-looking output that is factually wrong. You need patterns for all of these.

### Retry with Escalation

```
Attempt 1: Call the agent
  → If success: proceed
  → If format error: retry with corrective prompt
Attempt 2: Retry with correction
  → If success: proceed
  → If format error again: retry with simpler prompt
Attempt 3: Retry simplified
  → If success: proceed
  → If failure: route to human review with all three attempts attached
```

### Fallback Models

If the primary model is unavailable or returning errors, fall back to an alternative:

```
Primary: Claude 3.5 Sonnet
Fallback 1: Claude 3 Haiku (reduced quality, but functional)
Fallback 2: Static rules engine (no LLM, deterministic but limited)
Fallback 3: Human queue (manual processing)
```

### Circuit Breaker

If a stage fails more than N times in M minutes, stop sending requests:

1. **Closed** (normal operation) — requests flow through
2. **Open** (too many failures) — requests are immediately rejected or rerouted
3. **Half-open** (testing recovery) — allow one request through to test if the service has recovered

This prevents cascading failures and protects your budget from runaway retry loops.

### Graceful Degradation

When part of the pipeline fails, the system should degrade gracefully rather than crash entirely:

- If the knowledge base is unreachable, the agent can still operate with its base knowledge (with a warning flag).
- If the classification stage fails, route all documents to human review instead of dropping them.
- If the report generation stage fails, deliver the raw extracted data rather than nothing.

---

## Practice: Deploying Your Pipeline

### Exercise 1: Step Functions State Machine
Define and deploy a Step Functions state machine for your capstone pipeline. Include at least one Choice state (for branching), one retry policy, and one error handler.

### Exercise 2: Observability Setup
Instrument every Lambda function with structured logging. Enable X-Ray tracing. Build a CloudWatch dashboard showing: execution count, success rate, average latency per stage, and daily cost.

### Exercise 3: Error Injection
Deliberately break each stage of your pipeline (malformed input, model timeout, invalid output). Verify that your error handling works correctly in every case.

### Exercise 4: Cost Analysis
Run 50 documents through your pipeline. Calculate the per-document cost. Identify the most expensive stage. Optimize it by using a cheaper model or reducing input tokens.

---

## Key Takeaways

1. **Lambda + Step Functions + API Gateway = your deployment stack.** Lambda runs the code, Step Functions orchestrates the flow, API Gateway exposes the API.
2. **Observability is not a feature — it is a requirement.** Structured logging, distributed tracing, and cost tracking from day one.
3. **Errors in agent pipelines are different.** Models hallucinate, refuse, and produce subtly wrong output. Plan for all three.
4. **Cost is a first-class concern.** Track it per invocation, optimize aggressively, and set alerts before you need them.
5. **Graceful degradation beats hard failure.** When something breaks, deliver reduced functionality instead of nothing.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/week-10/">← Week 10: Building the Agent Pipeline</a> · Next: <a href="/level-2/week-12/">Week 12: Human-in-the-Loop & Production →</a>
</p>
