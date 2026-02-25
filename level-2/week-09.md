---
layout: page
title: "Week 9 — AWS Bedrock: Agents & Knowledge Bases"
description: "Setting up agents on AWS Bedrock, configuring knowledge bases, understanding guardrails and action groups. Your first cloud-deployed agent."
permalink: /level-2/week-09/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-2/">Level 2</a> / Week 9'
---

## From Local to Cloud

In Weeks 7 and 8, you engineered prompts and tested them locally. That is where the craft lives. But craft alone does not ship products. This week, you take your prompt engineering skills and deploy them on **AWS Bedrock** — Amazon's managed service for building generative AI applications.

Why Bedrock? Because it provides the infrastructure that turns your prompts into a production system: managed model hosting, retrieval-augmented generation, content guardrails, and tool execution — all without managing GPU clusters or inference servers yourself.

---

## AWS Bedrock Overview

Bedrock is not a single service. It is a collection of capabilities that work together:

| Component | What It Does | Why You Need It |
|-----------|-------------|-----------------|
| **Foundation Models** | Hosted LLMs (Claude, Titan, Llama, etc.) | The reasoning engine behind your agent |
| **Agents** | Autonomous reasoning + tool-use orchestration | Lets the model plan, decide, and act |
| **Knowledge Bases** | RAG with your own documents | Grounds agent responses in your data |
| **Guardrails** | Content filtering + behavioral constraints | Prevents harmful or off-topic outputs |
| **Action Groups** | Tool definitions that agents can invoke | Extends agent capabilities to external systems |

You will use all five in your capstone project. This week focuses on understanding each component and deploying your first working agent.

---

## Setting Up Your First Agent

An agent in Bedrock combines a foundation model with instructions, knowledge, and tools. Here is what you need:

### 1. Choose a Foundation Model

Bedrock supports multiple model families. For this course, you will use **Anthropic Claude** models. Key considerations:

- **Claude 3.5 Sonnet** — strong reasoning, good cost-performance balance, recommended for most capstone work.
- **Claude 3 Haiku** — fastest and cheapest, suitable for simple classification or routing steps in your pipeline.

Match the model to the task. Not every step in your pipeline needs the most capable model.

### 2. Write Agent Instructions

Agent instructions in Bedrock are the equivalent of the system prompts you wrote in Week 7, but they operate within Bedrock's orchestration framework:

```
You are a document processing agent for Acme Corp.

Your responsibilities:
1. Accept uploaded documents
2. Classify them using the classify_document action
3. Extract key information using the extract_info action
4. If the document is a contract, flag it for legal review
5. Store results in the knowledge base

Always explain your reasoning before taking an action.
Never process documents marked as confidential without human approval.
```

### 3. Define Action Groups

Action groups are the tools your agent can use. Each action group maps to a Lambda function that executes the actual operation:

```
Action Group: document_operations
  - classify_document(document_id) → Returns category and confidence
  - extract_info(document_id, fields[]) → Returns extracted field values
  - flag_for_review(document_id, reason) → Sends to human review queue
  - store_results(document_id, results) → Persists to database
```

The agent decides *when* to call each action based on its instructions and the current context. Your job is to define the actions precisely enough that the agent uses them correctly.

### 4. Connect a Knowledge Base

A knowledge base gives your agent access to your organization's documents through retrieval-augmented generation (RAG):

**How it works:**

1. You upload documents (PDFs, text files, web pages) to an S3 bucket.
2. Bedrock chunks the documents, generates embeddings, and stores them in a vector database.
3. When the agent needs information, it searches the knowledge base, retrieves relevant chunks, and incorporates them into its reasoning.

**Setting up a knowledge base:**

1. Create an S3 bucket with your source documents.
2. Choose an embedding model (Amazon Titan Embeddings is a solid default).
3. Configure a vector store (OpenSearch Serverless is the simplest option for getting started).
4. Define chunking strategy — fixed-size chunks (300–500 tokens) work for most document types. Overlap chunks by 10–20% to avoid splitting important context.
5. Sync the knowledge base to index your documents.

**When to use RAG vs. putting information in the prompt:**

| Use the Prompt | Use a Knowledge Base |
|---------------|---------------------|
| Small, static information (< 1000 tokens) | Large document collections |
| Instructions and constraints | Reference material the agent searches |
| Few-shot examples | Data that changes over time |
| Information needed on every request | Information needed only sometimes |

---

## Guardrails

Guardrails are Bedrock's mechanism for constraining what your agent can say and do. They operate as filters on both input and output:

### Content Filters

Block or flag content across categories:

- **Hate speech** — discriminatory or derogatory content
- **Violence** — graphic or threatening content
- **Sexual content** — explicit material
- **Insults** — demeaning language

Each category has configurable sensitivity levels (LOW, MEDIUM, HIGH). Set them based on your domain. A customer support agent needs strict filters. A legal review agent analyzing contracts that mention violence may need more nuanced settings.

### Denied Topics

Define specific topics your agent must never discuss:

```
Denied topics:
- Competitor products or pricing
- Internal salary information
- Legal advice (agent should flag for human lawyer)
- Medical diagnoses
```

### Word Filters

Block specific words or phrases from appearing in agent responses. Useful for preventing disclosure of internal terminology, code names, or sensitive identifiers.

### Sensitive Information Filters

Automatically detect and redact PII (personally identifiable information):

- Names, addresses, phone numbers
- Social Security numbers, credit card numbers
- Email addresses, IP addresses

Configure whether to block, mask, or flag sensitive information based on your compliance requirements.

---

## Your First Agent Deployment

This week's hands-on work: deploy a working agent on Bedrock that demonstrates all four components.

### Step-by-Step

1. **Create the agent** in the Bedrock console. Select Claude as the foundation model. Write agent instructions based on your capstone domain.

2. **Define an action group** with at least two actions. Implement the backing Lambda functions — they can return mock data for now. The goal is to verify the agent calls the right actions at the right time.

3. **Set up a knowledge base** with 10–20 sample documents relevant to your capstone domain. Test that the agent retrieves relevant information when asked.

4. **Configure guardrails** appropriate to your domain. Test that they block inappropriate content while allowing legitimate use cases.

5. **Test the agent** with at least 10 different inputs. For each input, verify:
   - The agent follows its instructions
   - It calls the correct actions with correct parameters
   - It retrieves relevant knowledge base content
   - Guardrails activate appropriately
   - The response is well-formed and useful

### What to Log

From the very first deployment, log everything:

- Agent invocation timestamps
- Input text and token count
- Which actions were called and with what parameters
- Knowledge base queries and retrieved chunks
- Guardrail activations (what was filtered and why)
- Output text and token count
- Total latency (end-to-end and per-step)
- Cost per invocation

This logging discipline pays massive dividends in Weeks 11 and 12 when you build observability and debug production issues.

---

## Common Pitfalls

**"My agent never uses the knowledge base."** The agent instructions must explicitly tell it when to search. Add: "Before answering any factual question about [domain], always search the knowledge base first."

**"My agent calls the wrong action."** Action descriptions are ambiguous. Rewrite them to be more specific. Add negative examples: "Do NOT use classify_document for documents that have already been classified."

**"The knowledge base returns irrelevant results."** Your chunking strategy may be wrong. If chunks are too small, they lack context. If too large, they dilute relevance. Experiment with chunk size and overlap.

**"Guardrails block legitimate requests."** Sensitivity levels are too high. Lower them incrementally, testing after each change. Log every guardrail activation so you can distinguish false positives from true positives.

---

## Key Takeaways

1. **Bedrock provides the production infrastructure.** Models, agents, knowledge bases, guardrails, and action groups — each solves a specific deployment problem.
2. **Agents combine reasoning with action.** The model decides what to do; action groups execute it.
3. **Knowledge bases ground agents in your data.** RAG prevents hallucination on domain-specific questions.
4. **Guardrails are safety nets, not substitutes for good prompts.** Design your prompts to be correct first, then add guardrails as defense in depth.
5. **Log from day one.** You cannot improve what you cannot measure.

---

<p style="font-size: 0.9rem;">
  <a href="/level-2/week-08/">← Week 8: Advanced Prompt Engineering</a> · Next: <a href="/level-2/week-10/">Week 10: Building the Agent Pipeline →</a>
</p>
