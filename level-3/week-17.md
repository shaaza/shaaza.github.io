---
layout: page
title: "Week 17: Fine-tuning & Evaluation"
description: "Fine-tune a pre-trained model for a specific task. LoRA, RLHF concepts, evaluation metrics and frameworks. When to fine-tune vs. prompt engineer."
permalink: /level-3/week-17/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 17'
---

## Why Fine-tune?

A pre-trained language model knows a lot about language. But it does not know about *your* task. Pre-training teaches the model to predict the next token on general text. Fine-tuning teaches it to behave the way you want.

There are two fundamental approaches to adapting a pre-trained model:

1. **Prompt engineering** — craft the input so the model produces the desired output, without changing any weights
2. **Fine-tuning** — update the model's weights on task-specific data

This week you will learn when each approach is appropriate, implement parameter-efficient fine-tuning with LoRA, understand the concepts behind RLHF, and build a rigorous evaluation framework.

---

## When to Fine-tune vs. Prompt Engineer

This is the most important practical decision in applied ML. Getting it wrong wastes weeks of work.

### Prompt Engineering Is Sufficient When:

- The task can be described clearly in a few sentences
- The model already has the knowledge needed (it is in the pre-training data)
- You need flexibility to change the task frequently
- You have few or no labeled examples
- Latency and cost of the base model are acceptable

### Fine-tuning Is Necessary When:

- The model needs domain-specific knowledge not in its pre-training data
- You need consistent output format or style that prompts cannot reliably enforce
- You want to reduce model size (fine-tune a smaller model to match a larger one)
- You need to reduce inference cost (specialized smaller model vs. general large model)
- You have a large labeled dataset and a well-defined task

### A Decision Framework

```
Do you have > 1000 labeled examples?
├── No → Try prompt engineering first
│        ├── Works well enough? → Stop. You are done.
│        └── Not good enough? → Try few-shot prompting with 5-20 examples
│             ├── Works? → Stop.
│             └── Still not good enough? → Fine-tune with what you have
└── Yes → Does prompt engineering already work well?
          ├── Yes → Prompt engineering is cheaper. Use it.
          └── No → Fine-tune.
```

The key insight: **always try the cheapest approach first**. Fine-tuning costs compute, requires infrastructure, and introduces a model management burden. It should be a deliberate choice, not a default.

---

## Full Fine-tuning

The simplest approach: update all parameters on your task-specific data.

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "gpt2"  # or any pre-trained model
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name)

# Prepare task-specific data
def format_example(instruction, response):
    return f"### Instruction:\n{instruction}\n\n### Response:\n{response}<|endoftext|>"

train_texts = [
    format_example("Summarize this article...", "The article discusses..."),
    format_example("Translate to French...", "Voici la traduction..."),
    # ... thousands more
]

# Tokenize
encodings = tokenizer(train_texts, truncation=True, padding=True,
                       max_length=512, return_tensors='pt')

# Fine-tune with standard training loop
optimizer = torch.optim.AdamW(model.parameters(), lr=2e-5, weight_decay=0.01)

model.train()
for epoch in range(3):
    for i in range(0, len(encodings['input_ids']), batch_size):
        batch = {k: v[i:i+batch_size].to(device) for k, v in encodings.items()}
        outputs = model(**batch, labels=batch['input_ids'])
        loss = outputs.loss

        optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), 1.0)
        optimizer.step()
```

### Problems with Full Fine-tuning

| Problem | Description |
|---------|-------------|
| **Catastrophic forgetting** | The model loses pre-trained knowledge as task-specific gradients overwrite it |
| **Memory cost** | You need to store optimizer states for every parameter (2x model size for Adam) |
| **Storage cost** | Each fine-tuned model is a full copy of the original |
| **Overfitting** | With small datasets, the model memorizes instead of generalizing |

---

## Parameter-Efficient Fine-tuning: LoRA

**Low-Rank Adaptation (LoRA)** solves these problems by freezing the pre-trained weights and injecting small trainable rank-decomposition matrices.

### The LoRA Idea

For a pre-trained weight matrix `W` of shape `(d, k)`, instead of learning a full update `ΔW` (also `d × k` parameters), LoRA decomposes the update into two small matrices:

```
ΔW = B × A
```

Where `B` has shape `(d, r)` and `A` has shape `(r, k)`, with rank `r` much smaller than `d` or `k`.

```
Original:  h = Wx
LoRA:      h = Wx + BAx
           h = (W + BA)x
```

### Implementing LoRA from Scratch

```python
import torch
import torch.nn as nn
import math

class LoRALayer(nn.Module):
    """LoRA adapter for a linear layer."""

    def __init__(self, original_layer, rank=8, alpha=16):
        super().__init__()
        self.original_layer = original_layer
        self.rank = rank
        self.alpha = alpha
        self.scaling = alpha / rank

        in_features = original_layer.in_features
        out_features = original_layer.out_features

        # LoRA matrices
        self.lora_A = nn.Parameter(torch.randn(in_features, rank) / math.sqrt(rank))
        self.lora_B = nn.Parameter(torch.zeros(rank, out_features))

        # Freeze the original layer
        for param in self.original_layer.parameters():
            param.requires_grad = False

    def forward(self, x):
        # Original forward pass (frozen)
        original_output = self.original_layer(x)
        # LoRA forward pass (trainable)
        lora_output = (x @ self.lora_A @ self.lora_B) * self.scaling
        return original_output + lora_output

def apply_lora(model, rank=8, alpha=16, target_modules=None):
    """Apply LoRA to specified modules in a model."""
    if target_modules is None:
        target_modules = ['q_proj', 'v_proj']  # common choice for transformers

    for name, module in model.named_modules():
        if any(target in name for target in target_modules):
            if isinstance(module, nn.Linear):
                parent_name = '.'.join(name.split('.')[:-1])
                child_name = name.split('.')[-1]
                parent = dict(model.named_modules())[parent_name]
                setattr(parent, child_name, LoRALayer(module, rank=rank, alpha=alpha))

    # Count trainable parameters
    trainable = sum(p.numel() for p in model.parameters() if p.requires_grad)
    total = sum(p.numel() for p in model.parameters())
    print(f"Trainable: {trainable:,} / {total:,} ({100*trainable/total:.2f}%)")
```

### Using LoRA with PEFT

In practice, use the Hugging Face PEFT library:

```python
from peft import LoraConfig, get_peft_model, TaskType

lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=8,                          # rank
    lora_alpha=16,                # scaling factor
    lora_dropout=0.1,             # dropout on LoRA layers
    target_modules=["q_proj", "v_proj"],  # which layers to adapt
)

model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# Output: trainable params: 294,912 || all params: 124,734,464 || trainable%: 0.24%
```

### LoRA Hyperparameters

| Parameter | What It Controls | Typical Values |
|-----------|-----------------|----------------|
| `r` (rank) | Capacity of the adaptation | 4, 8, 16, 32 |
| `alpha` | Scaling factor (learning rate multiplier) | Usually 2x the rank |
| `target_modules` | Which layers to adapt | q_proj, v_proj (minimum); all linear layers (maximum) |
| `dropout` | Regularization on LoRA layers | 0.05 - 0.1 |

Higher rank means more capacity but more parameters. Start with `r=8` and increase only if the model is underfitting.

---

## RLHF Concepts

**Reinforcement Learning from Human Feedback** is the technique that turns a pre-trained language model into an assistant that follows instructions and avoids harmful outputs. You will not implement full RLHF this week, but understanding the concepts is essential.

### The Three Stages of RLHF

**Stage 1: Supervised Fine-tuning (SFT)**
Train the model on high-quality instruction-response pairs:
```
Human: What is the capital of France?
Assistant: The capital of France is Paris.
```

**Stage 2: Reward Model Training**
Collect human preferences — for each prompt, show two model responses and ask which is better. Train a reward model to predict human preferences:

```python
# Simplified reward model
class RewardModel(nn.Module):
    def __init__(self, base_model):
        super().__init__()
        self.base_model = base_model
        self.reward_head = nn.Linear(base_model.config.hidden_size, 1)

    def forward(self, input_ids, attention_mask):
        outputs = self.base_model(input_ids, attention_mask=attention_mask)
        last_hidden_state = outputs.last_hidden_state[:, -1, :]
        reward = self.reward_head(last_hidden_state)
        return reward
```

**Stage 3: RL Optimization (PPO)**
Use the reward model as a signal to optimize the language model policy via Proximal Policy Optimization:

```
For each prompt:
    1. Generate a response from the current policy
    2. Score the response with the reward model
    3. Update the policy to increase expected reward
    4. Add a KL penalty to prevent the policy from diverging
       too far from the SFT model
```

### Direct Preference Optimization (DPO)

DPO is a simpler alternative to PPO-based RLHF that skips the reward model entirely:

```python
def dpo_loss(model, ref_model, chosen_ids, rejected_ids, beta=0.1):
    """Direct Preference Optimization loss."""
    # Log probabilities under current model
    chosen_logprobs = get_log_probs(model, chosen_ids)
    rejected_logprobs = get_log_probs(model, rejected_ids)

    # Log probabilities under reference (frozen) model
    with torch.no_grad():
        ref_chosen_logprobs = get_log_probs(ref_model, chosen_ids)
        ref_rejected_logprobs = get_log_probs(ref_model, rejected_ids)

    # DPO objective
    chosen_rewards = beta * (chosen_logprobs - ref_chosen_logprobs)
    rejected_rewards = beta * (rejected_logprobs - ref_rejected_logprobs)
    loss = -torch.nn.functional.logsigmoid(chosen_rewards - rejected_rewards).mean()

    return loss
```

DPO is becoming the preferred alignment technique due to its simplicity and stability.

---

## Evaluation

Fine-tuning without rigorous evaluation is guesswork. You need to measure whether your fine-tuned model actually improves on the task.

### Automated Metrics

```python
from collections import Counter
import numpy as np

def compute_metrics(predictions, references):
    """Compute standard NLP metrics."""
    metrics = {}

    # Exact match
    metrics['exact_match'] = np.mean([
        p.strip() == r.strip()
        for p, r in zip(predictions, references)
    ])

    # BLEU (for generation tasks)
    from nltk.translate.bleu_score import sentence_bleu
    metrics['bleu'] = np.mean([
        sentence_bleu([r.split()], p.split())
        for p, r in zip(predictions, references)
    ])

    # ROUGE (for summarization)
    from rouge_score import rouge_scorer
    scorer = rouge_scorer.RougeScorer(['rouge1', 'rouge2', 'rougeL'])
    rouge_scores = [scorer.score(r, p) for p, r in zip(predictions, references)]
    metrics['rouge1'] = np.mean([s['rouge1'].fmeasure for s in rouge_scores])
    metrics['rougeL'] = np.mean([s['rougeL'].fmeasure for s in rouge_scores])

    return metrics
```

### LLM-as-Judge Evaluation

For open-ended generation, automated metrics correlate poorly with human judgment. Use a strong model to evaluate:

```python
def llm_judge(prompt, response_a, response_b, judge_model):
    """Use an LLM to compare two responses."""
    eval_prompt = f"""You are an expert evaluator. Given a prompt and two responses,
determine which response is better. Consider accuracy, helpfulness, and clarity.

Prompt: {prompt}

Response A: {response_a}

Response B: {response_b}

Which response is better? Respond with "A" or "B" and a brief explanation."""

    judgment = judge_model.generate(eval_prompt)
    return judgment
```

### Building an Evaluation Framework

```python
class EvalFramework:
    """Systematic evaluation of fine-tuned models."""

    def __init__(self, test_cases):
        self.test_cases = test_cases

    def evaluate(self, model, tokenizer):
        results = []
        for case in self.test_cases:
            # Generate response
            input_ids = tokenizer.encode(case['prompt'], return_tensors='pt')
            output_ids = model.generate(
                input_ids.to(model.device),
                max_new_tokens=256,
                temperature=0.0,  # deterministic for evaluation
                do_sample=False,
            )
            response = tokenizer.decode(output_ids[0], skip_special_tokens=True)

            results.append({
                'prompt': case['prompt'],
                'expected': case.get('expected'),
                'response': response,
                'category': case.get('category', 'general'),
            })

        return results

    def report(self, results):
        """Generate an evaluation report."""
        # Group by category
        categories = {}
        for r in results:
            cat = r['category']
            if cat not in categories:
                categories[cat] = []
            categories[cat].append(r)

        print("=" * 60)
        print("EVALUATION REPORT")
        print("=" * 60)
        for cat, cases in categories.items():
            if cases[0].get('expected'):
                exact_match = np.mean([
                    c['expected'].strip() in c['response']
                    for c in cases
                ])
                print(f"\n{cat}: {len(cases)} cases, exact match: {exact_match:.1%}")
            else:
                print(f"\n{cat}: {len(cases)} cases (manual review needed)")
```

### Evaluation Anti-Patterns

| Anti-Pattern | Why It Is Wrong | What to Do Instead |
|---|---|---|
| Evaluating on training data | Measures memorization, not generalization | Use a held-out test set |
| Single metric | Hides failure modes | Use multiple complementary metrics |
| No baseline comparison | Cannot tell if fine-tuning helped | Always compare to the base model and to prompt engineering |
| Ignoring failure cases | Aggregate metrics hide dangerous behaviors | Manually review worst-performing examples |
| Training on the test set | Even accidentally, through iterative tuning | Lock the test set; use a separate dev set for tuning |

---

## Exercises

### Exercise 1: LoRA Fine-tuning

Fine-tune GPT-2 (or a similar small model) on a classification task using LoRA:
1. Choose a dataset (e.g., SST-2 sentiment, AG News topics)
2. Format examples as instruction-response pairs
3. Fine-tune with LoRA (rank 8)
4. Compare accuracy against: (a) zero-shot prompting, (b) few-shot prompting, (c) full fine-tuning

### Exercise 2: Rank Ablation

Fine-tune the same model with different LoRA ranks: 1, 2, 4, 8, 16, 32, 64. Plot:
- Task accuracy vs. rank
- Training time vs. rank
- Number of trainable parameters vs. rank

At what rank does performance plateau? How does this relate to the intrinsic dimensionality of the task?

### Exercise 3: Target Module Selection

Fine-tune with LoRA applied to different module combinations:
- Query and Value only (`q_proj`, `v_proj`)
- All attention weights (`q_proj`, `k_proj`, `v_proj`, `o_proj`)
- All linear layers (attention + MLP)

Which configuration gives the best accuracy-efficiency trade-off?

### Exercise 4: Build an Evaluation Suite

For a task of your choice, build a comprehensive evaluation suite with:
- At least 100 test cases across 3+ categories
- Automated metrics (exact match, BLEU, etc.)
- LLM-as-judge evaluation
- Manual spot-checking of 20 random samples

Document cases where automated metrics disagree with human judgment.

### Exercise 5: SFT vs. DPO

Using the TRL library, fine-tune a model using both SFT and DPO on the same preference dataset:

```python
from trl import SFTTrainer, DPOTrainer

# Compare the two approaches on the same evaluation suite
```

Which produces better outputs? On which types of prompts does each approach excel?

---

## Key Takeaways

- **Try prompt engineering first** — fine-tuning is expensive and creates maintenance burden
- **LoRA is the default** for parameter-efficient fine-tuning: 0.1-1% of parameters, comparable results to full fine-tuning
- **RLHF/DPO aligns models with human preferences** — SFT teaches format, alignment teaches behavior
- **Evaluation is the hardest part** — bad evaluation leads to confidently deploying bad models
- **Always compare against baselines** — base model, prompt engineering, and full fine-tuning
- **Catastrophic forgetting is real** — monitor the model's general capabilities, not just task performance

---

## Resources

- [LoRA: Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685) — the original LoRA paper
- [DPO: Direct Preference Optimization](https://arxiv.org/abs/2305.18290) — the paper that simplified alignment
- [Hugging Face PEFT documentation](https://huggingface.co/docs/peft) — the standard library for parameter-efficient fine-tuning
- [TRL documentation](https://huggingface.co/docs/trl) — Transformer Reinforcement Learning library
- [LMSYS Chatbot Arena](https://chat.lmsys.org/) — how the community evaluates language models via head-to-head comparison

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/week-16/">← Week 16: Pre-training</a> · Next: <a href="/level-3/week-18/">Week 18: Edge Deployment & Final Project →</a>
</p>
