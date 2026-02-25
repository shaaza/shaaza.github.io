---
layout: page
title: "Week 16: Pre-training on Custom Data"
description: "Pre-train a small language model on your own data. Data pipelines, distributed training basics, loss curves, and hyperparameter tuning."
permalink: /level-3/week-16/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 16'
---

## From Toy Models to Real Training

In Week 15, you built nanoGPT and trained it on Shakespeare. The model was small, the dataset was small, and everything fit on one GPU. This week you confront the reality of training at scale.

**Pre-training** is the process of training a language model on a large corpus of text before specializing it for any particular task. It is the most expensive, most consequential, and most poorly understood phase of building a language model. Decisions made during pre-training — data composition, learning rate schedule, model size, training duration — determine the ceiling of everything that follows.

You will not train a frontier model this week. But you will build a complete pre-training pipeline on custom data, encounter the same engineering problems that arise at scale, and develop the intuitions needed to reason about training dynamics.

---

## Core Concepts

### Why Pre-train?

Pre-training works because of a remarkable empirical observation: **a model trained to predict the next token on diverse text learns general-purpose representations that transfer to almost any downstream task**.

This is not obvious. Predicting the next word in a Wikipedia article seems unrelated to answering medical questions or writing code. But the next-token prediction objective forces the model to build internal representations of grammar, facts, reasoning patterns, and world knowledge — all as a side effect of getting better at prediction.

### The Pre-training Pipeline

A production pre-training pipeline has five stages:

```
Raw Data → Cleaning → Tokenization → Data Loading → Training Loop
```

Each stage has engineering challenges that do not appear in toy experiments.

---

## Stage 1: Data Collection and Cleaning

### Choosing Your Corpus

For this week, you will pre-train on a domain-specific corpus. Choose one:

- **Code**: a collection of repositories in a single language (Python, Rust, etc.)
- **Domain text**: medical papers, legal documents, technical manuals
- **Web text**: a subset of Common Crawl or similar web scrape

The choice matters. A model pre-trained on code will be better at code. A model pre-trained on medical text will know more medicine. There is no free lunch.

### Data Cleaning Pipeline

Raw text is messy. A cleaning pipeline typically includes:

```python
import re
from pathlib import Path

def clean_text(text):
    """Basic text cleaning pipeline."""
    # Remove null bytes and control characters
    text = re.sub(r'[\x00-\x08\x0b\x0c\x0e-\x1f\x7f]', '', text)

    # Normalize whitespace
    text = re.sub(r'[ \t]+', ' ', text)           # collapse horizontal whitespace
    text = re.sub(r'\n{3,}', '\n\n', text)         # collapse excessive newlines

    # Remove very short lines (likely noise)
    lines = text.split('\n')
    lines = [l for l in lines if len(l.strip()) > 0 or l == '']

    return '\n'.join(lines)

def process_file(path):
    """Process a single file with quality filters."""
    text = path.read_text(encoding='utf-8', errors='ignore')
    text = clean_text(text)

    # Quality filters
    if len(text) < 100:       # too short
        return None
    if len(text) > 1_000_000:  # too long — likely a data dump, not real text
        return None

    # Language detection (optional)
    # Deduplication (by hash, by n-gram overlap, etc.)

    return text
```

### Deduplication

Duplicate data is surprisingly common and surprisingly harmful. Models trained on duplicated data:
- Memorize specific passages instead of learning general patterns
- Generate verbatim copies of training data (privacy and legal concerns)
- Have inflated evaluation metrics that do not reflect real capability

```python
import hashlib
from collections import defaultdict

def deduplicate_exact(documents):
    """Remove exact duplicates using content hashing."""
    seen = set()
    unique = []
    for doc in documents:
        h = hashlib.sha256(doc.encode()).hexdigest()
        if h not in seen:
            seen.add(h)
            unique.append(doc)
    print(f"Removed {len(documents) - len(unique)} exact duplicates")
    return unique

def deduplicate_minhash(documents, threshold=0.8):
    """Remove near-duplicates using MinHash.
    For production, use the datasketch library."""
    from datasketch import MinHash, MinHashLSH

    lsh = MinHashLSH(threshold=threshold, num_perm=128)
    unique = []

    for i, doc in enumerate(documents):
        m = MinHash(num_perm=128)
        for word in doc.split():
            m.update(word.encode('utf-8'))

        if not lsh.query(m):
            lsh.insert(str(i), m)
            unique.append(doc)

    print(f"Removed {len(documents) - len(unique)} near-duplicates")
    return unique
```

### Data Mixing

If you have multiple data sources, the **mixing ratio** significantly affects model behavior:

```python
data_mix = {
    'code':    0.30,   # 30% source code
    'docs':    0.25,   # 25% documentation
    'web':     0.25,   # 25% web text
    'books':   0.15,   # 15% books
    'qa':      0.05,   # 5% question-answer pairs
}
```

There is no formula for the optimal mix. It depends on what you want the model to be good at. The Llama papers, the Chinchilla paper, and the GPT-4 technical report all discuss data mixing strategies — but the exact recipes remain proprietary.

---

## Stage 2: Tokenization

### Training a Tokenizer

For domain-specific pre-training, you may want a custom tokenizer instead of a general-purpose one:

```python
from tokenizers import Tokenizer, models, trainers, pre_tokenizers

def train_bpe_tokenizer(texts, vocab_size=8192, save_path="tokenizer.json"):
    """Train a Byte-Pair Encoding tokenizer on custom data."""
    tokenizer = Tokenizer(models.BPE())
    tokenizer.pre_tokenizer = pre_tokenizers.ByteLevel(add_prefix_space=False)

    trainer = trainers.BpeTrainer(
        vocab_size=vocab_size,
        special_tokens=["<|endoftext|>", "<|padding|>"],
        min_frequency=2,
    )

    tokenizer.train_from_iterator(texts, trainer=trainer)
    tokenizer.save(save_path)
    print(f"Trained tokenizer with vocab size {tokenizer.get_vocab_size()}")
    return tokenizer
```

### Tokenization Efficiency

The tokenizer's vocabulary directly affects training efficiency. A domain-specific tokenizer will produce fewer tokens for in-domain text:

```python
# Compare tokenization efficiency
general_tokenizer = ...  # GPT-2's tokenizer
custom_tokenizer = ...   # trained on your domain

sample_text = "def forward(self, x): return self.linear(x)"
general_tokens = general_tokenizer.encode(sample_text)
custom_tokens = custom_tokenizer.encode(sample_text)

print(f"General tokenizer: {len(general_tokens.ids)} tokens")
print(f"Custom tokenizer:  {len(custom_tokens.ids)} tokens")
# Custom tokenizer typically produces fewer tokens for in-domain text
```

Fewer tokens means:
- Longer effective context window
- Faster training (fewer tokens to process)
- Better per-token predictions (each token carries more information)

---

## Stage 3: Efficient Data Loading

### Memory-Mapped Dataset

For datasets that do not fit in memory, use memory-mapped files:

```python
import numpy as np
import torch
from torch.utils.data import Dataset, DataLoader

class MemmapDataset(Dataset):
    """Memory-mapped dataset for large-scale pre-training."""

    def __init__(self, data_path, block_size):
        self.data = np.memmap(data_path, dtype=np.uint16, mode='r')
        self.block_size = block_size

    def __len__(self):
        return len(self.data) - self.block_size - 1

    def __getitem__(self, idx):
        x = torch.from_numpy(self.data[idx:idx+self.block_size].astype(np.int64))
        y = torch.from_numpy(self.data[idx+1:idx+self.block_size+1].astype(np.int64))
        return x, y

# Prepare: tokenize all data and save as a flat binary file
def prepare_data(texts, tokenizer, output_path):
    """Tokenize all texts and write to a binary file."""
    all_tokens = []
    for text in texts:
        tokens = tokenizer.encode(text).ids
        all_tokens.extend(tokens)
        all_tokens.append(tokenizer.token_to_id("<|endoftext|>"))

    arr = np.array(all_tokens, dtype=np.uint16)
    arr.tofile(output_path)
    print(f"Wrote {len(arr):,} tokens to {output_path}")
    return arr
```

### DataLoader with Prefetching

```python
train_dataset = MemmapDataset('train.bin', block_size=1024)
train_loader = DataLoader(
    train_dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,        # parallel data loading
    pin_memory=True,      # faster GPU transfer
    persistent_workers=True,
    prefetch_factor=2,    # prefetch 2 batches per worker
)
```

---

## Stage 4: The Training Loop

### Learning Rate Schedule

Modern pre-training uses a learning rate schedule with **warmup** and **cosine decay**:

```python
import math

def get_lr(step, warmup_steps, max_steps, max_lr, min_lr):
    """Cosine learning rate schedule with linear warmup."""
    # Linear warmup
    if step < warmup_steps:
        return max_lr * (step / warmup_steps)

    # Cosine decay
    if step > max_steps:
        return min_lr

    decay_ratio = (step - warmup_steps) / (max_steps - warmup_steps)
    coeff = 0.5 * (1.0 + math.cos(math.pi * decay_ratio))
    return min_lr + coeff * (max_lr - min_lr)

# Typical values
warmup_steps = 2000
max_steps = 100_000
max_lr = 6e-4
min_lr = 6e-5
```

The warmup phase prevents early training instability. The cosine decay gradually reduces the learning rate, allowing the model to settle into a good minimum.

### Gradient Accumulation

If your batch size is too large for GPU memory, accumulate gradients over multiple mini-batches:

```python
gradient_accumulation_steps = 8  # effective batch = 8 * micro_batch_size

optimizer.zero_grad()
for micro_step in range(gradient_accumulation_steps):
    x, y = next(train_iter)
    x, y = x.to(device), y.to(device)

    logits, loss = model(x, y)
    loss = loss / gradient_accumulation_steps  # scale loss
    loss.backward()

optimizer.step()
```

### Gradient Clipping

Large gradients can destabilize training. Clip the global gradient norm:

```python
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
```

### The Complete Training Script

```python
def train(model, train_loader, config):
    optimizer = torch.optim.AdamW(
        model.parameters(),
        lr=config.max_lr,
        betas=(0.9, 0.95),
        weight_decay=0.1,
    )

    model.train()
    train_iter = iter(train_loader)
    losses = []

    for step in range(config.max_steps):
        # Update learning rate
        lr = get_lr(step, config.warmup_steps, config.max_steps,
                    config.max_lr, config.min_lr)
        for param_group in optimizer.param_groups:
            param_group['lr'] = lr

        # Gradient accumulation
        optimizer.zero_grad()
        total_loss = 0.0

        for micro_step in range(config.grad_accum_steps):
            try:
                x, y = next(train_iter)
            except StopIteration:
                train_iter = iter(train_loader)
                x, y = next(train_iter)

            x, y = x.to(config.device), y.to(config.device)
            logits, loss = model(x, y)
            loss = loss / config.grad_accum_steps
            loss.backward()
            total_loss += loss.item()

        # Gradient clipping
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

        optimizer.step()
        losses.append(total_loss)

        # Logging
        if step % 100 == 0:
            print(f"step {step:6d} | loss {total_loss:.4f} | lr {lr:.2e}")

        # Checkpointing
        if step % 5000 == 0 and step > 0:
            torch.save({
                'model': model.state_dict(),
                'optimizer': optimizer.state_dict(),
                'step': step,
                'loss': total_loss,
            }, f'checkpoint_{step}.pt')

    return losses
```

---

## Stage 5: Monitoring and Debugging

### Loss Curves

The training loss curve tells you almost everything about your training run:

```python
import matplotlib.pyplot as plt

def plot_loss_curve(losses, window=100):
    """Plot raw and smoothed training loss."""
    plt.figure(figsize=(12, 6))
    plt.plot(losses, alpha=0.3, label='Raw loss')

    # Smoothed loss (moving average)
    smoothed = [
        sum(losses[max(0,i-window):i+1]) / len(losses[max(0,i-window):i+1])
        for i in range(len(losses))
    ]
    plt.plot(smoothed, label=f'Smoothed (window={window})')

    plt.xlabel('Step')
    plt.ylabel('Loss')
    plt.title('Pre-training Loss Curve')
    plt.legend()
    plt.grid(True, alpha=0.3)
    plt.show()
```

What to look for:
- **Smooth decrease**: healthy training
- **Sudden spikes**: learning rate too high, bad data batch, or numerical instability
- **Plateau**: model capacity exhausted, learning rate too low, or data exhausted
- **Train-val gap widening**: overfitting — need more data or more regularization

### Validation and Perplexity

```python
@torch.no_grad()
def evaluate(model, val_loader, device, max_batches=50):
    model.eval()
    total_loss = 0.0
    total_tokens = 0

    for i, (x, y) in enumerate(val_loader):
        if i >= max_batches:
            break
        x, y = x.to(device), y.to(device)
        logits, loss = model(x, y)
        total_loss += loss.item() * y.numel()
        total_tokens += y.numel()

    avg_loss = total_loss / total_tokens
    perplexity = math.exp(avg_loss)
    model.train()
    return avg_loss, perplexity
```

**Perplexity** is `exp(loss)`. Intuitively, it is the effective number of equally likely next tokens the model is considering. A perplexity of 10 means the model is as uncertain as if it were choosing uniformly among 10 options at each step.

---

## Distributed Training Basics

For models or datasets that do not fit on a single GPU, you need distributed training.

### Data Parallelism with DDP

The simplest form: replicate the model on each GPU, split the data, and average gradients.

```python
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def setup_ddp(rank, world_size):
    dist.init_process_group("nccl", rank=rank, world_size=world_size)
    torch.cuda.set_device(rank)

def train_ddp(rank, world_size, config):
    setup_ddp(rank, world_size)

    model = GPT(config).to(rank)
    model = DDP(model, device_ids=[rank])

    # Each process gets a different slice of data
    sampler = torch.utils.data.distributed.DistributedSampler(
        train_dataset, num_replicas=world_size, rank=rank, shuffle=True
    )
    train_loader = DataLoader(train_dataset, batch_size=config.batch_size,
                              sampler=sampler, num_workers=4)

    # Training loop is the same — DDP handles gradient synchronization
    train(model, train_loader, config)

    dist.destroy_process_group()

# Launch with:
# torchrun --nproc_per_node=4 train.py
```

### Key Distributed Training Concepts

| Concept | What It Does | When You Need It |
|---------|-------------|-----------------|
| **Data Parallelism** | Same model on each GPU, different data | Model fits on one GPU, want faster training |
| **Model Parallelism** | Split model layers across GPUs | Model does not fit on one GPU |
| **Pipeline Parallelism** | Different layers on different GPUs, overlapping compute | Very deep models |
| **FSDP** | Shard model parameters, gradients, and optimizer states | Large models, memory-constrained |

For this week's exercises, data parallelism with DDP is sufficient.

---

## Exercises

### Exercise 1: Pre-train on a Custom Corpus

Choose a domain-specific corpus (code, medical, legal, fiction). Build the complete pipeline:
1. Clean and deduplicate the data
2. Train a BPE tokenizer
3. Tokenize and write to binary format
4. Pre-train a nanoGPT model (at least 10M parameters)
5. Plot the loss curve and compute perplexity

### Exercise 2: Data Ablation

Train three models on the same architecture but with different data:
- 10% of the corpus
- 50% of the corpus
- 100% of the corpus

Plot the loss curves together. How does data quantity affect final loss?

### Exercise 3: Hyperparameter Sensitivity

Vary one hyperparameter at a time and measure the effect on validation loss:
- Learning rate: 1e-4, 3e-4, 6e-4, 1e-3
- Batch size: 16, 32, 64, 128
- Warmup steps: 0, 500, 2000, 5000
- Weight decay: 0, 0.01, 0.1, 0.3

Which hyperparameter has the largest effect?

### Exercise 4: Chinchilla Scaling

The [Chinchilla paper](https://arxiv.org/abs/2203.15556) argues that many models are trained on too little data for their size. Compute the "Chinchilla-optimal" data-to-parameter ratio for your model and compare with how much data you actually have.

### Exercise 5: Resume from Checkpoint

Intentionally kill a training run at 50% completion. Resume from the last checkpoint and verify that:
- Training loss continues from where it left off (no regression)
- The optimizer state is correctly restored
- The learning rate schedule picks up at the right step

This is a critical engineering skill — real training runs crash, and you must be able to resume.

---

## Key Takeaways

- **Data quality matters more than quantity** — cleaning, deduplication, and mixing are not optional
- **The learning rate schedule** (warmup + cosine decay) is critical for stable training
- **Gradient accumulation** lets you train with large effective batch sizes on limited GPU memory
- **Loss curves are your primary diagnostic tool** — learn to read them
- **Checkpointing is not optional** — training runs will fail, and you must be able to resume
- **Distributed training** with DDP is conceptually simple: same code, gradients averaged across GPUs

---

## Resources

- [nanoGPT training script](https://github.com/karpathy/nanoGPT/blob/master/train.py) — a clean reference implementation
- [Chinchilla: Training Compute-Optimal Large Language Models](https://arxiv.org/abs/2203.15556) — the paper that changed how we think about scaling
- [Llama 2 Technical Report](https://arxiv.org/abs/2307.09288) — detailed description of a production pre-training pipeline
- [PyTorch DDP Tutorial](https://pytorch.org/tutorials/intermediate/ddp_tutorial.html) — official distributed training guide
- [The FineWeb dataset paper](https://arxiv.org/abs/2406.17557) — a detailed study of data quality for pre-training

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/week-15/">← Week 15: nanoGPT</a> · Next: <a href="/level-3/week-17/">Week 17: Fine-tuning & Evaluation →</a>
</p>
