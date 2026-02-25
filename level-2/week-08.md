---
layout: page
title: "Week 8 — Advanced Prompt Engineering"
description: "Tool use, function calling, multi-turn conversations, prompt chaining, and evaluation frameworks. Testing prompts systematically."
permalink: /level-2/week-08/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 8'
---

## Beyond Single Prompts

Week 7 taught you to write a single prompt that produces reliable output. That is necessary but not sufficient. Real agent systems involve **sequences** of prompts, **tools** that extend what the model can do, **conversations** that span multiple turns, and **evaluation** that tells you when your prompts have regressed.

This week, you learn to engineer all of it.

---

## Tool Use and Function Calling

Language models can reason, but they cannot act. They cannot query a database, call an API, read a file, or send an email. **Tool use** bridges this gap by letting the model *request* that an external function be executed on its behalf.

### How Function Calling Works

1. You define a set of **tools** — functions the model is allowed to call, each with a name, description, and parameter schema.
2. The model receives the user's request along with the tool definitions.
3. Instead of (or in addition to) generating text, the model generates a **tool call** — a structured request specifying which function to invoke and with what arguments.
4. Your code executes the function and returns the result to the model.
5. The model incorporates the result into its response.

### Defining Tools

A tool definition is a contract. It tells the model what the tool does, what parameters it accepts, and what it returns:

```json
{
  "name": "search_knowledge_base",
  "description": "Search the company knowledge base for relevant documents. Use this when the user asks about company policies, procedures, or historical decisions.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search query. Use specific keywords, not full sentences."
      },
      "max_results": {
        "type": "integer",
        "description": "Maximum number of results to return (1-10)",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

### Tool Use Best Practices

- **Write descriptions as instructions.** The description tells the model *when* to use the tool. "Search the knowledge base for relevant documents" is better than "Knowledge base search."
- **Constrain parameters tightly.** Use enums, ranges, and required fields to prevent invalid calls.
- **Include negative guidance.** "Do NOT use this tool for general knowledge questions — only for company-specific information."
- **Test tool selection.** Given an ambiguous request, does the model choose the right tool? Or does it hallucinate an answer instead of searching?

---

## Multi-Turn Conversations

Single-turn interactions (one input, one output) are the simplest case. Many real systems require **multi-turn conversations** where context accumulates across exchanges.

### Context Window Management

Every message in a conversation consumes tokens. Long conversations eventually exceed the model's context window. You must manage this proactively:

1. **Summarize history.** Periodically compress earlier messages into a summary, preserving key facts while reducing token count.
2. **Sliding window.** Keep only the most recent N messages, plus the system prompt and any pinned context.
3. **Selective retrieval.** Instead of keeping all history, retrieve relevant past messages on demand using embeddings or keyword matching.

### Conversation State

In multi-turn systems, you need to track state that persists across turns:

```
Turn 1: User uploads a document
  → Agent extracts metadata, stores it
Turn 2: User asks "What are the key risks?"
  → Agent needs the document from Turn 1
Turn 3: User says "Flag the third one as critical"
  → Agent needs the risk list from Turn 2
```

Each turn depends on context from previous turns. Design your system to maintain a structured state object alongside the conversation, rather than relying entirely on the model's ability to recall earlier messages.

### Maintaining Persona Consistency

Over long conversations, models can drift from their system prompt. Techniques to prevent this:

- **Re-inject key instructions** at intervals (every N turns, include a reminder of the core constraints).
- **Use structured state** rather than relying on the model to remember facts from 20 turns ago.
- **Test for drift** — run evaluation conversations of 30+ turns and check if later responses still respect the system prompt.

---

## Prompt Chaining

Prompt chaining breaks a complex task into a **sequence of simpler prompts**, where each prompt's output becomes the next prompt's input. This is the architectural pattern at the heart of your capstone pipeline.

### Why Chain Instead of Using One Big Prompt?

| Single Prompt | Chained Prompts |
|--------------|-----------------|
| One model call handles everything | Each step is a focused, testable unit |
| Errors compound invisibly | Errors are caught at each stage |
| Hard to debug | Intermediate outputs are inspectable |
| Difficult to insert human review | Natural breakpoints for verification |
| One-size-fits-all instructions | Each step has specialized instructions |

### A Chaining Example: Document Processing

```
Step 1: Extract
  Input: Raw PDF text
  Prompt: "Extract all factual claims from this document. Return as a JSON array."
  Output: ["claim1", "claim2", ...]

Step 2: Classify
  Input: Array of claims from Step 1
  Prompt: "Classify each claim as VERIFIABLE, OPINION, or AMBIGUOUS."
  Output: [{"claim": "...", "type": "VERIFIABLE"}, ...]

Step 3: Verify
  Input: VERIFIABLE claims from Step 2
  Prompt: "For each claim, search the knowledge base and assess accuracy."
  Output: [{"claim": "...", "accurate": true, "source": "..."}, ...]

Step 4: Report
  Input: All classified and verified claims
  Prompt: "Generate a summary report with verified facts, flagged opinions, and confidence levels."
  Output: Formatted report
```

Each step is independently testable. If Step 2 misclassifies a claim, you can fix Step 2's prompt without touching the others. If you want human review of the verification results, you insert it between Steps 3 and 4.

### Chaining Best Practices

- **Each link does one thing.** If a step does two things, split it.
- **Validate between steps.** Parse and check the output of each step before passing it to the next.
- **Use consistent data formats.** JSON is a natural choice for inter-step communication.
- **Design for partial failure.** If Step 3 fails on one claim, the others should still process.
- **Log everything.** Capture the input, output, and latency of every step for debugging and optimization.

---

## Evaluation Frameworks

How do you know your prompts work? Not "they seem to work on the three examples I tried" — how do you *systematically measure* whether they produce correct outputs?

### Building an Evaluation Suite

An evaluation suite is a collection of **test cases**: known inputs paired with expected outputs or evaluation criteria.

```
Test case 1:
  Input: "My order #4521 hasn't arrived and it's been 2 weeks"
  Expected: {"category": "shipping", "priority": "high"}
  Criteria: category must be "shipping", priority must be "high" or "critical"

Test case 2:
  Input: "How do I change my password?"
  Expected: {"category": "account", "priority": "low"}
  Criteria: category must be "account", priority must be "low"
```

Run your prompt against every test case. Compute metrics:

- **Accuracy** — What percentage of outputs match expectations?
- **Consistency** — If you run the same input 10 times, how often do you get the same output?
- **Format compliance** — What percentage of outputs are valid JSON matching the expected schema?
- **Latency** — How long does each evaluation take?
- **Cost** — How many tokens does each evaluation consume?

### Regression Testing

Every time you change a prompt, re-run your evaluation suite. This is **prompt regression testing** — the same principle as software regression testing, applied to natural language instructions.

Workflow:

1. Write the initial prompt.
2. Build the evaluation suite (20+ test cases minimum).
3. Run the suite. Record baseline metrics.
4. Modify the prompt.
5. Re-run the suite. Compare against baseline.
6. If any metric regresses, investigate before deploying.

### Model-as-Judge Evaluation

For subjective outputs (summaries, explanations, recommendations), you can use a separate model call to evaluate quality:

```
You are an evaluation judge. Rate the following summary on these criteria:
1. Accuracy (1-5): Does it correctly represent the source material?
2. Completeness (1-5): Does it capture all key points?
3. Conciseness (1-5): Is it free of unnecessary information?
4. Clarity (1-5): Is it easy to understand?

Source document: {{source}}
Summary to evaluate: {{summary}}

Respond with JSON: {"accuracy": N, "completeness": N, "conciseness": N, "clarity": N, "justification": "..."}
```

This is not a replacement for human evaluation, but it scales to hundreds or thousands of test cases where human review is impractical.

---

## Practice: Building Your Evaluation Pipeline

### Exercise 1: Tool Definition
Define a set of three to five tools for your capstone domain. Write descriptions that cause the model to select the right tool for at least 10 different user requests.

### Exercise 2: Prompt Chain
Implement a three-step prompt chain for your capstone domain. Validate the output at each step. Measure end-to-end accuracy.

### Exercise 3: Evaluation Suite
Build a 20+ test case evaluation suite for one step of your prompt chain. Run it, compute metrics, then modify the prompt and measure regression.

### Exercise 4: Multi-Turn Stress Test
Create a 20-turn conversation scenario. Verify the agent maintains correct behavior, context, and persona throughout.

---

## Key Takeaways

1. **Tool use extends agent capabilities beyond text.** Define tools precisely and test tool selection rigorously.
2. **Multi-turn conversations require state management.** Do not rely on the model to remember — maintain structured state.
3. **Prompt chaining decomposes complexity.** Each link does one thing, is independently testable, and produces inspectable intermediate outputs.
4. **Evaluation is not optional.** Build test suites, measure metrics, run regression tests on every prompt change.
5. **Treat your prompts as a codebase.** They need version control, testing, code review, and continuous integration — just like any other software.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/week-07/">← Week 7: Prompt Engineering Fundamentals</a> · Next: <a href="/level-2/week-09/">Week 9: AWS Bedrock — Agents & Knowledge Bases →</a>
</p>
