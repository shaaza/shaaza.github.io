---
layout: page
title: "Week 7 — Prompt Engineering Fundamentals"
description: "Move from implicit prompting to explicit craft. System prompts, few-shot examples, chain-of-thought, and structured outputs."
permalink: /level-2/week-07/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 7'
---

## The Shift

In Level 1, you wrote prompts without thinking much about it. You told the agent what you wanted, adjusted when it got things wrong, and moved on. That was fine when you were the only user — you could compensate for imprecise instructions in real time.

That era is over.

In Level 2, your prompts are **code**. They are specifications that must produce correct behavior without your supervision. They will be executed thousands of times against inputs you have never seen. They must be precise, testable, and versioned — just like any other piece of production software.

This week, you learn the foundations of that craft.

---

## System Prompts

A system prompt is the first message in a conversation. It sets the agent's identity, constraints, and behavioral expectations before the user ever speaks.

A bad system prompt:

```
You are a helpful assistant.
```

A good system prompt:

```
You are a document classification agent for Acme Corp's legal department.

Your task: Given a document, classify it into exactly one of these categories:
- CONTRACT
- INVOICE
- CORRESPONDENCE
- REGULATORY
- OTHER

Rules:
1. Always respond with a valid JSON object: {"category": "...", "confidence": 0.0-1.0, "reasoning": "..."}
2. If you cannot determine the category with confidence > 0.7, use "OTHER"
3. Never invent categories not in the list above
4. Your reasoning must reference specific features of the document
```

The difference: the good system prompt specifies the **task**, the **output format**, the **constraints**, and the **fallback behavior**. It leaves nothing to interpretation.

### Principles for Effective System Prompts

1. **State the role explicitly.** "You are a document classification agent" not "You are helpful."
2. **Define the output format.** If you want JSON, show the exact schema. If you want markdown, show the structure.
3. **Enumerate constraints.** What must the agent *never* do? What are the boundaries?
4. **Specify fallback behavior.** What should happen when the agent is uncertain?
5. **Keep it as short as possible while being complete.** Every unnecessary word is a potential source of confusion.

---

## Few-Shot Examples

Few-shot prompting is providing the model with example input-output pairs before the real input. It is the single most reliable way to control output format and behavioral patterns.

### Why Few-Shot Works

Language models are pattern-completion machines. When you show them three examples of input → output, they will match the pattern on the fourth input far more reliably than if you describe the desired behavior in prose.

### How to Write Good Few-Shot Examples

```
Classify the following support ticket.

Example 1:
Input: "My order #4521 hasn't arrived and it's been 2 weeks"
Output: {"category": "shipping", "priority": "high", "requires_human": false}

Example 2:
Input: "How do I change my password?"
Output: {"category": "account", "priority": "low", "requires_human": false}

Example 3:
Input: "I want to sue your company for damages"
Output: {"category": "legal", "priority": "critical", "requires_human": true}

Now classify this ticket:
Input: "{{user_input}}"
Output:
```

### Few-Shot Best Practices

- **Use 3–5 examples.** Fewer may not establish the pattern. More adds cost without proportional benefit.
- **Cover edge cases.** Include at least one example that demonstrates boundary behavior (like the legal threat above).
- **Make examples realistic.** Synthetic-looking examples produce synthetic-looking outputs.
- **Keep format identical across examples.** Any inconsistency in your examples will produce inconsistency in outputs.
- **Order matters.** Place the most representative example first and the most unusual last.

---

## Chain-of-Thought Prompting

Chain-of-thought (CoT) prompting asks the model to show its reasoning before giving a final answer. This consistently improves accuracy on tasks that require multi-step reasoning.

### Basic Chain-of-Thought

```
Analyze this contract clause for risk. Think step by step before giving your assessment.

Clause: "Vendor shall not be liable for any indirect, incidental, or consequential damages."

Step-by-step analysis:
```

The model will walk through its reasoning — identifying the clause type, noting what it excludes, flagging potential issues — before producing a final assessment. This intermediate reasoning dramatically reduces errors compared to asking for the answer directly.

### Structured Chain-of-Thought

For production use, you want the reasoning to follow a predictable structure:

```
Analyze the following clause for risk.

For each clause, follow this exact reasoning process:
1. IDENTIFY: What type of clause is this? (liability, indemnity, termination, IP, confidentiality, other)
2. PARTIES: Who does this clause benefit and who does it constrain?
3. RISKS: What specific risks does this create for our client?
4. PRECEDENT: Is this standard language or unusual?
5. VERDICT: Rate risk as LOW / MEDIUM / HIGH / CRITICAL with one-sentence justification.

Clause: "{{clause_text}}"
```

This approach gives you inspectable reasoning — you can audit not just the answer but the path the model took to reach it.

---

## Structured Outputs

In production systems, free-text responses are a liability. You need outputs you can parse, validate, and route programmatically. Structured outputs solve this.

### JSON Outputs

The most common structured output format. Specify the exact schema you expect:

```
Extract key information from this invoice. Respond with ONLY a JSON object matching this schema:

{
  "vendor_name": "string",
  "invoice_number": "string",
  "date": "YYYY-MM-DD",
  "line_items": [
    {
      "description": "string",
      "quantity": number,
      "unit_price": number,
      "total": number
    }
  ],
  "subtotal": number,
  "tax": number,
  "total": number
}

Do not include any text before or after the JSON object.
```

### Validation

Never trust model output blindly. Always validate structured outputs:

1. **Parse the output** — Does it parse as valid JSON?
2. **Schema validation** — Does it match the expected schema? Are all required fields present?
3. **Type checking** — Are numbers actually numbers? Are dates valid dates?
4. **Range checking** — Do the values make sense? (A negative price is probably wrong.)
5. **Consistency checking** — Do line item totals sum to the subtotal?

If validation fails, you have options: retry the prompt, flag for human review, or return an error. Which strategy you choose depends on the cost of failure in your domain.

---

## Practice: Real Tasks

This week is hands-on. You will practice prompt engineering on real tasks drawn from the capstone domains:

### Exercise 1: Document Classification
Write a system prompt that classifies documents into categories. Test it against at least 20 documents. Measure accuracy.

### Exercise 2: Information Extraction
Write a prompt that extracts structured data from unstructured text (invoices, contracts, or support tickets). Validate the output programmatically.

### Exercise 3: Risk Assessment
Write a chain-of-thought prompt that assesses risk in a given domain. Evaluate whether the reasoning is consistent and the conclusions are justified.

### Exercise 4: Prompt Comparison
Take one of the tasks above. Write three different prompt strategies (zero-shot, few-shot, CoT). Compare outputs on the same inputs. Document which approach works best and why.

---

## Key Takeaways

1. **Prompts are code.** Version them. Test them. Review them.
2. **System prompts set constraints.** Be explicit about role, format, boundaries, and fallback behavior.
3. **Few-shot examples are behavioral specifications.** Three to five well-chosen examples beat paragraphs of instruction.
4. **Chain-of-thought improves reasoning.** Show the model how to think, not just what to output.
5. **Structured outputs enable automation.** Parse, validate, and route — never trust free text in production.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/">← Level 2 Overview</a> · Next: <a href="/level-2/week-08/">Week 8: Advanced Prompt Engineering →</a>
</p>
