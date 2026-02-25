---
layout: page
title: "Level 2 — Building Agent Systems"
description: "Move from using agents to building agent systems for others. Prompt engineering becomes a first-class skill. Ship things real people use."
permalink: /level-2/
breadcrumb: '<a href="/">Schedule</a> / Level 2'
---

## Philosophy

Level 1 taught you to *use* AI agents as a developer tool. You gave them instructions, reviewed their output, and shipped code faster than you could alone.

Level 2 inverts the relationship. **You are no longer the user of agents — you are the builder of agent systems that other people use.**

This is a fundamentally different challenge. When you use an agent, you can compensate for its failures in real time. When you build an agent system for others, you must anticipate failures, constrain behavior, verify outputs, and handle every edge case before your users ever see the system. The gap between "it works when I try it" and "it works reliably for strangers" is where most agent projects die.

Two skills become first-class in Level 2:

1. **Prompt engineering** — the craft of writing instructions that reliably produce correct behavior from language models. Not vague suggestions. Precise, tested, versioned prompts that perform consistently across inputs.

2. **Human-in-the-loop design** — knowing when and how to insert human verification into automated workflows. Not every decision should be automated. Learning where to draw that line is what separates production systems from demos.

---

## Capstone: AWS Bedrock Agent Pipeline

By the end of Level 2, you will build and deploy a **production agentic workflow on AWS Bedrock**. This is not a toy. It is a system that accepts real inputs, processes them through multiple agent stages, incorporates human review at critical decision points, and produces verified outputs.

You will choose one of the following domains for your capstone (or propose your own with instructor approval):

| Domain | Input | Agent Pipeline | Output |
|--------|-------|---------------|--------|
| **Document Processing** | PDF upload | Summarize → Review → Format | Structured summary with key findings |
| **Legal Review** | Contract document | Clause extraction → Risk flagging → Sign-off | Annotated contract with risk assessment |
| **File Operations** | File drop to watched folder | Classify → Route → Transform → Deliver | Processed files in correct destinations |
| **Customer Support** | Incoming ticket | Triage → Draft response → Approval → Send | Verified customer response |

Every domain shares the same architectural pattern: **ingest → process → verify → output**. The specific content differs, but the engineering challenges are identical: reliable parsing, consistent agent behavior, graceful error handling, human review gates, and observable operations.

---

## Key Concepts

Level 2 introduces a set of concepts that will define the rest of your career working with AI systems:

### Prompt Engineering
- System prompts and their role in constraining behavior
- Few-shot examples as behavioral specifications
- Chain-of-thought prompting for complex reasoning
- Structured output formats (JSON, XML, markdown)
- Prompt versioning, testing, and evaluation

### Human-in-the-Loop
- Verification patterns: when to require human approval
- Confidence thresholds: letting the agent decide when to escalate
- Feedback loops: using human corrections to improve prompts
- Audit trails: recording every decision for review

### AWS Bedrock
- **Agents** — autonomous reasoning entities that can use tools
- **Knowledge Bases** — retrieval-augmented generation (RAG) with your own data
- **Guardrails** — content filters and behavioral constraints
- **Action Groups** — defining what tools an agent can use and when

### Deployment
- **AWS Lambda** — serverless compute for agent logic
- **Step Functions** — orchestrating multi-step agent workflows
- **API Gateway** — exposing your pipeline as an API

### Observability
- **Logging** — recording every agent invocation and response
- **Tracing** — following a request through every stage of the pipeline
- **Cost tracking** — understanding and controlling LLM spend

### Error Handling
- What happens when the model hallucinates?
- What happens when an upstream service is down?
- What happens when the input is malformed?
- What happens when the agent refuses a valid request?

---

## The Six Weeks

Each week builds on the last. Weeks 7–8 focus on the craft of prompt engineering. Weeks 9–10 focus on building the system. Weeks 11–12 focus on making it production-ready.

| Week | Title | What You Build |
|------|-------|---------------|
| [Week 7](/level-2/week-07/) | Prompt Engineering Fundamentals | System prompts, few-shot examples, chain-of-thought, structured outputs |
| [Week 8](/level-2/week-08/) | Advanced Prompt Engineering | Tool use, function calling, multi-turn conversations, evaluation frameworks |
| [Week 9](/level-2/week-09/) | AWS Bedrock — Agents & Knowledge Bases | First agent deployment with knowledge base and guardrails |
| [Week 10](/level-2/week-10/) | Building the Agent Pipeline | Full multi-agent capstone pipeline in your chosen domain |
| [Week 11](/level-2/week-11/) | Deployment & Observability | Lambda, Step Functions, API Gateway, logging, tracing, cost tracking |
| [Week 12](/level-2/week-12/) | Human-in-the-Loop & Production | Verification patterns, monitoring, alerting, graceful degradation, final demo |

---

## What You Will Be Able to Do

By the end of Level 2, you will be able to:

- Write precise, tested prompts that reliably produce correct agent behavior
- Design multi-agent workflows with clear handoff points
- Build and deploy agent pipelines on AWS Bedrock
- Implement human-in-the-loop verification at critical decision points
- Set up observability for cost, performance, and correctness
- Handle the full spectrum of agent failures gracefully
- Ship a production-ready agent system that real people can use

---

<p style="font-size: 0.9rem;">
  <a href="/">← Back to Schedule</a> · Next: <a href="/level-2/week-07/">Week 7: Prompt Engineering Fundamentals →</a>
</p>
