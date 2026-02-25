---
layout: page
title: "Week 12 — Human-in-the-Loop & Production"
description: "Verification patterns, production readiness, monitoring, alerting, graceful degradation. What happens when the agent is wrong? Final demo."
permalink: /level-2/week-12/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 12'
---

## The Hardest Question

Your pipeline is built. It is deployed. It is instrumented. It processes inputs, chains through stages, handles errors, and produces outputs.

Now answer the hardest question in production AI: **What happens when the agent is wrong?**

Not when it crashes — you handle that with retries and circuit breakers. Not when it refuses — you handle that with fallbacks. When it confidently produces output that *looks correct but is not*. When the legal review agent misses a critical clause. When the document summarizer omits a key finding. When the customer support agent sends a response with incorrect information.

This is the problem that separates demos from production systems. This week, you solve it.

---

## Human-in-the-Loop Verification

Human-in-the-loop (HITL) is not about distrust. It is about appropriate trust. You do not trust a junior employee to sign contracts without review. You do not trust automated tests without code review. And you do not trust an AI agent to make consequential decisions without human verification — at least not until you have enough data to calibrate your confidence.

### Verification Patterns

#### Pattern 1: Approve Before Action

The agent prepares an action but does not execute it. A human reviews and approves (or rejects) before the action takes effect.

```
Agent: "I've drafted a response to the customer's complaint about
        order #4521. Here is the draft: [response text]"

Human reviews:
  ✓ Approve → Response is sent
  ✗ Reject → Agent receives feedback, drafts a revision
  ✎ Edit → Human modifies the draft, then sends
```

**When to use:** Before any irreversible action — sending an email, filing a document, making a payment, publishing content.

#### Pattern 2: Confidence-Based Escalation

The agent processes the request and self-assesses its confidence. Low-confidence results are escalated to a human; high-confidence results proceed automatically.

```
Agent output: {
  "result": "CONTRACT",
  "confidence": 0.62,
  "reasoning": "Document contains contractual language but also
                resembles a letter of intent."
}

if confidence < 0.8:
  → Route to human review queue
else:
  → Proceed to next pipeline stage
```

**When to use:** High-volume pipelines where reviewing every output is impractical. The threshold should be calibrated empirically — start conservative (0.9) and lower it as you build trust in the agent's self-assessment.

**Warning:** Model confidence scores are not perfectly calibrated. A model that says "0.95 confidence" might be wrong 20% of the time. Measure actual accuracy at each confidence level and adjust thresholds based on real data, not the model's self-report.

#### Pattern 3: Sampling-Based Audit

A random sample of agent outputs is reviewed by humans, even when the agent is confident. This catches systematic errors that confidence thresholds miss.

```
Processing 1000 documents per day
  → Agent processes all 1000
  → 5% random sample (50 documents) routed to human review
  → Reviewer checks accuracy, flags errors
  → Error patterns feed back into prompt improvements
```

**When to use:** At scale, when you need ongoing quality assurance without reviewing every output. Essential during the first weeks after deployment when you are still learning the agent's failure modes.

#### Pattern 4: Dual-Agent Verification

Two agents process the same input independently. If they disagree, the case is escalated to a human.

```
Agent A (Sonnet): classifies document as "CONTRACT"
Agent B (Haiku):  classifies document as "LETTER OF INTENT"

Disagree → Route to human review
Agree    → Proceed with high confidence
```

**When to use:** High-stakes decisions where the cost of error is severe. More expensive (two model calls per input) but significantly more reliable for critical applications.

---

## Feedback Loops

Human reviews are not just safety nets — they are **training data for prompt improvement**. Every human correction is a signal about where your prompts fall short.

### Building the Feedback Loop

1. **Capture every human correction.** When a reviewer overrides the agent, log the original input, agent output, and human correction.

2. **Categorize errors.** Build a taxonomy:
   - **Format errors** — output did not match expected schema
   - **Classification errors** — wrong category or label
   - **Extraction errors** — missed information or hallucinated information
   - **Reasoning errors** — correct conclusion from wrong reasoning (or vice versa)
   - **Policy errors** — violated a business rule

3. **Identify patterns.** Do errors cluster around specific input types? Specific stages? Specific times of day (indicating API degradation)?

4. **Update prompts.** Use error patterns to add few-shot examples, clarify instructions, or add explicit constraints. Then re-run your evaluation suite to verify the fix does not cause regressions.

5. **Measure improvement.** Track error rates over time. If your feedback loop is working, error rates should decrease week over week.

---

## Production Readiness Checklist

Before declaring your pipeline production-ready, verify each item:

### Reliability

- [ ] Every stage has retry logic with exponential backoff
- [ ] Timeout is configured for every Lambda function and Step Functions state
- [ ] Circuit breakers prevent cascading failures
- [ ] Dead letter queues capture failed invocations for later analysis
- [ ] Graceful degradation is implemented for each failure mode
- [ ] The pipeline recovers automatically from transient failures

### Correctness

- [ ] Evaluation suite covers at least 50 test cases
- [ ] Accuracy metrics meet acceptance thresholds for your domain
- [ ] Edge cases are tested: empty input, huge input, malformed input, adversarial input
- [ ] Human review gates are in place for high-stakes decisions
- [ ] Output validation runs at every stage transition

### Observability

- [ ] Structured logging is in every Lambda function
- [ ] Execution ID is present in every log entry
- [ ] X-Ray tracing is enabled end-to-end
- [ ] CloudWatch dashboard shows: throughput, latency, error rate, cost
- [ ] Alerts fire for: error rate > threshold, latency > threshold, cost > budget

### Security

- [ ] API Gateway authentication is configured
- [ ] Lambda functions have least-privilege IAM roles
- [ ] Sensitive data (PII, credentials) is not logged
- [ ] Guardrails are configured and tested
- [ ] S3 buckets with documents are encrypted and access-controlled

### Cost

- [ ] Per-invocation cost is measured and documented
- [ ] Cost-per-unit (per document, per ticket, etc.) is calculated
- [ ] Budget alerts are set in AWS Billing
- [ ] Model selection is optimized (Haiku for simple stages, Sonnet for complex)
- [ ] Token usage is minimized at each stage

### Operations

- [ ] Deployment is automated (CloudFormation, CDK, or SAM)
- [ ] Rollback procedure is documented and tested
- [ ] On-call rotation is established (even if it is just you)
- [ ] Runbook exists for common failure scenarios
- [ ] Human review queue has an SLA (e.g., reviewed within 2 hours)

---

## Monitoring and Alerting

### What to Monitor

| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| Error rate | > 5% over 15 minutes | Page on-call, investigate immediately |
| P95 latency | > 30 seconds per stage | Investigate slow stages, check model availability |
| Human review queue depth | > 50 items | Add reviewers or adjust confidence threshold |
| Daily cost | > 120% of budget | Review token usage, check for runaway retries |
| Guardrail activation rate | > 10% | Review inputs — are users testing boundaries? |
| Evaluation accuracy | Drops below 90% | Investigate prompt drift, model changes |

### Alert Fatigue

More alerts is not better. Every alert should require action. If an alert fires and the response is "ignore it," the alert should not exist. Start with the five most important metrics and add more only when you have evidence they are needed.

---

## Graceful Degradation in Practice

When parts of your system fail, the user experience should degrade gracefully, not catastrophically:

| Failure | Hard Failure | Graceful Degradation |
|---------|-------------|---------------------|
| Knowledge base is down | "Error: Service unavailable" | Agent responds with base knowledge + warning: "Note: I could not access the knowledge base. This response is based on general knowledge only." |
| Classification agent times out | Pipeline halts, document is lost | Document is queued for retry + human notification: "Document #123 could not be auto-classified. Routed to manual review." |
| Report generation fails | No output delivered | Deliver raw extracted data: "Automated report generation failed. Here are the raw findings for manual compilation." |
| Guardrails block valid input | User sees an error message | Rephrase the request and retry. If still blocked, route to human: "This request requires manual processing due to content policy." |

The principle: **always deliver something useful.** A partial result is infinitely more valuable than an error message.

---

## The Final Demo

Your Level 2 capstone is a **live demonstration** of your production agent pipeline. You will show:

1. **End-to-end processing.** Feed a real input into the pipeline and trace it through every stage to the final output.

2. **Human review in action.** Demonstrate the approval gate — show what the reviewer sees, how they approve or reject, and how the pipeline responds.

3. **Error handling.** Deliberately inject a failure (malformed input, model timeout) and show the pipeline recovering gracefully.

4. **Observability.** Walk through your dashboard. Show tracing for a single request. Show the cost breakdown.

5. **Evaluation results.** Present your accuracy metrics, your evaluation suite, and how they have improved over the project.

6. **Lessons learned.** What surprised you? What was harder than expected? What would you do differently?

### Demo Evaluation Criteria

| Criterion | Weight | What We Look For |
|-----------|--------|-----------------|
| Pipeline functionality | 25% | Does it work end-to-end with real inputs? |
| Error handling | 20% | Does it fail gracefully under stress? |
| Observability | 15% | Can you trace a request and explain what happened? |
| Human-in-the-loop | 15% | Are review gates well-designed and functional? |
| Prompt quality | 15% | Are prompts precise, tested, and versioned? |
| Cost awareness | 10% | Have you optimized model selection and token usage? |

---

## What Comes Next

Level 2 taught you to build agent systems for others — to engineer prompts, deploy pipelines, observe operations, and handle the full lifecycle of a production AI system.

Level 3 takes the next step: **building systems that can build systems.** You will work with agents that write code, review code, test code, and deploy code — the full software engineering lifecycle, automated.

The principles you learned here — precise specifications, human verification, observability, graceful degradation — are not just relevant to Level 3. They are *foundational*. Every agent system you build for the rest of your career will use these patterns.

---

## Key Takeaways

1. **The hardest problem is silent failure.** Crashes are easy to detect. Confident wrong answers are not. Design for both.
2. **Human-in-the-loop is trust calibration.** Start with high human oversight, reduce it as you build evidence of reliability.
3. **Feedback loops improve the system.** Every human correction is a prompt improvement waiting to happen.
4. **Production readiness is a checklist, not a feeling.** Reliability, correctness, observability, security, cost, operations — verify each one.
5. **Always deliver something useful.** Graceful degradation beats hard failure every time.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/week-11/">← Week 11: Deployment & Observability</a> · <a href="/">Back to Schedule</a>
</p>
