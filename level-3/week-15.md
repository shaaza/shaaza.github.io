---
layout: page
title: "Week 15: nanoGPT — Building Transformers"
description: "Implement the transformer architecture from scratch. Attention mechanism, positional encoding, and a complete GPT model."
permalink: /level-3/week-15/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 15'
---

## The Architecture That Changed Everything

In Week 13, you built the gradient engine. In Week 14, you built a language model with an MLP. This week, you replace the MLP with the architecture that powers every frontier language model: **the transformer**.

The transformer, introduced in [Vaswani et al., 2017](https://arxiv.org/abs/1706.03762), solved a fundamental limitation of previous architectures: the inability to efficiently attend to all positions in a sequence simultaneously. RNNs process tokens sequentially. CNNs have limited receptive fields. The transformer's **self-attention** mechanism lets every token directly interact with every other token, with learned importance weights.

You will implement [nanoGPT](https://github.com/karpathy/nanoGPT), a small but complete GPT model. Every component — embedding, attention, feed-forward network, layer normalization, residual connections — will be written from scratch in PyTorch.

---

## Core Concepts

### Self-Attention: The Key Idea

The central question of self-attention is: **when processing a token, how much should it attend to each other token in the sequence?**

Consider the sentence "The cat sat on the mat because it was tired." When processing "it", the model needs to figure out that "it" refers to "the cat", not "the mat". Self-attention learns to make this connection by computing a relevance score between every pair of tokens.

### Queries, Keys, and Values

Self-attention uses three projections of each token's embedding:

- **Query (Q)**: "What am I looking for?"
- **Key (K)**: "What do I contain?"
- **Value (V)**: "What information do I provide?"

The attention score between token *i* and token *j* is the dot product of token *i*'s query with token *j*'s key. High dot product means high relevance.

```python
# Single head of self-attention
import torch
import torch.nn as nn
import torch.nn.functional as F

class Head(nn.Module):
    def __init__(self, n_embd, head_size, block_size, dropout=0.1):
        super().__init__()
        self.key   = nn.Linear(n_embd, head_size, bias=False)
        self.query = nn.Linear(n_embd, head_size, bias=False)
        self.value = nn.Linear(n_embd, head_size, bias=False)
        self.register_buffer('tril', torch.tril(torch.ones(block_size, block_size)))
        self.dropout = nn.Dropout(dropout)

    def forward(self, x):
        B, T, C = x.shape
        k = self.key(x)    # (B, T, head_size)
        q = self.query(x)  # (B, T, head_size)

        # Compute attention scores
        wei = q @ k.transpose(-2, -1) * C**-0.5  # (B, T, T)
        wei = wei.masked_fill(self.tril[:T, :T] == 0, float('-inf'))  # causal mask
        wei = F.softmax(wei, dim=-1)  # (B, T, T)
        wei = self.dropout(wei)

        # Weighted aggregation of values
        v = self.value(x)  # (B, T, head_size)
        out = wei @ v      # (B, T, head_size)
        return out
```

The key details:

1. **Scaled dot-product**: dividing by `sqrt(head_size)` prevents the dot products from growing too large, which would push softmax into regions with tiny gradients.

2. **Causal mask**: the `tril` mask ensures token *i* can only attend to tokens at positions <= *i*. This is what makes the model **autoregressive** — it cannot look into the future.

3. **Softmax normalization**: converts raw scores into a probability distribution. Each token's attention weights over all previous tokens sum to 1.

### Multi-Head Attention

A single attention head can only focus on one kind of relationship at a time. **Multi-head attention** runs multiple heads in parallel, each learning different attention patterns:

```python
class MultiHeadAttention(nn.Module):
    def __init__(self, n_embd, num_heads, head_size, block_size, dropout=0.1):
        super().__init__()
        self.heads = nn.ModuleList([
            Head(n_embd, head_size, block_size, dropout)
            for _ in range(num_heads)
        ])
        self.proj = nn.Linear(n_embd, n_embd)  # projection back to residual pathway
        self.dropout = nn.Dropout(dropout)

    def forward(self, x):
        out = torch.cat([h(x) for h in self.heads], dim=-1)
        out = self.dropout(self.proj(out))
        return out
```

In practice, if the model has embedding dimension 384 and 6 heads, each head operates on 384/6 = 64 dimensions. The heads are concatenated and projected back to the full embedding dimension.

### Positional Encoding

Self-attention is **permutation-invariant** — it treats the input as a set, not a sequence. To encode position information, we add **positional embeddings**:

```python
class GPT(nn.Module):
    def __init__(self, vocab_size, n_embd, block_size, n_head, n_layer, dropout=0.1):
        super().__init__()
        self.token_embedding_table = nn.Embedding(vocab_size, n_embd)
        self.position_embedding_table = nn.Embedding(block_size, n_embd)
        # ... (rest of the model)

    def forward(self, idx, targets=None):
        B, T = idx.shape
        tok_emb = self.token_embedding_table(idx)          # (B, T, n_embd)
        pos_emb = self.position_embedding_table(torch.arange(T, device=idx.device))  # (T, n_embd)
        x = tok_emb + pos_emb  # (B, T, n_embd) — position information added
        # ...
```

Each position (0, 1, 2, ..., block_size-1) gets a learned embedding vector. Adding this to the token embedding gives the model both "what token is this?" and "where is this token in the sequence?"

### Feed-Forward Network

After attention, each token passes through a **position-wise feed-forward network** — the same two-layer MLP applied independently to each position:

```python
class FeedForward(nn.Module):
    def __init__(self, n_embd, dropout=0.1):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(n_embd, 4 * n_embd),   # expand
            nn.GELU(),                         # non-linearity
            nn.Linear(4 * n_embd, n_embd),    # project back
            nn.Dropout(dropout),
        )

    def forward(self, x):
        return self.net(x)
```

The 4x expansion is a design choice from the original transformer paper. The feed-forward network gives each token a chance to "think" about the information gathered by attention.

### Layer Normalization and Residual Connections

Deep networks are hard to train. Two techniques make transformers trainable:

**Residual connections** add the input of each sub-layer to its output: `x = x + sublayer(x)`. This creates a "gradient highway" that allows gradients to flow directly from the loss to early layers.

**Layer normalization** normalizes activations across the feature dimension, stabilizing training:

```python
class Block(nn.Module):
    """Transformer block: communication (attention) followed by computation (FFN)"""

    def __init__(self, n_embd, n_head, block_size, dropout=0.1):
        super().__init__()
        head_size = n_embd // n_head
        self.sa = MultiHeadAttention(n_embd, n_head, head_size, block_size, dropout)
        self.ffwd = FeedForward(n_embd, dropout)
        self.ln1 = nn.LayerNorm(n_embd)
        self.ln2 = nn.LayerNorm(n_embd)

    def forward(self, x):
        x = x + self.sa(self.ln1(x))    # pre-norm + residual
        x = x + self.ffwd(self.ln2(x))  # pre-norm + residual
        return x
```

Note the **pre-norm** pattern: layer norm is applied *before* the sub-layer, not after. This is the convention used in GPT-2 and most modern transformers (the original paper used post-norm).

---

## The Complete nanoGPT Model

Putting it all together:

```python
class GPT(nn.Module):
    def __init__(self, vocab_size, n_embd, block_size, n_head, n_layer, dropout=0.1):
        super().__init__()
        self.block_size = block_size
        self.token_embedding_table = nn.Embedding(vocab_size, n_embd)
        self.position_embedding_table = nn.Embedding(block_size, n_embd)
        self.blocks = nn.Sequential(*[
            Block(n_embd, n_head, block_size, dropout)
            for _ in range(n_layer)
        ])
        self.ln_f = nn.LayerNorm(n_embd)  # final layer norm
        self.lm_head = nn.Linear(n_embd, vocab_size)

    def forward(self, idx, targets=None):
        B, T = idx.shape

        # Embeddings
        tok_emb = self.token_embedding_table(idx)
        pos_emb = self.position_embedding_table(torch.arange(T, device=idx.device))
        x = tok_emb + pos_emb

        # Transformer blocks
        x = self.blocks(x)
        x = self.ln_f(x)

        # Language model head
        logits = self.lm_head(x)  # (B, T, vocab_size)

        if targets is None:
            loss = None
        else:
            B, T, C = logits.shape
            logits = logits.view(B * T, C)
            targets = targets.view(B * T)
            loss = F.cross_entropy(logits, targets)

        return logits, loss

    @torch.no_grad()
    def generate(self, idx, max_new_tokens):
        for _ in range(max_new_tokens):
            # Crop context to block_size
            idx_cond = idx[:, -self.block_size:]
            logits, _ = self(idx_cond)
            logits = logits[:, -1, :]  # focus on last time step
            probs = F.softmax(logits, dim=-1)
            idx_next = torch.multinomial(probs, num_samples=1)
            idx = torch.cat((idx, idx_next), dim=1)
        return idx
```

### Model Configuration

For training on a single GPU with the Shakespeare dataset:

```python
# Hyperparameters
batch_size = 64
block_size = 256      # context length
n_embd = 384          # embedding dimension
n_head = 6            # number of attention heads
n_layer = 6           # number of transformer blocks
dropout = 0.2
learning_rate = 3e-4
max_iters = 5000
eval_interval = 500
eval_iters = 200
device = 'cuda' if torch.cuda.is_available() else 'cpu'
```

This configuration produces a model with roughly 10M parameters — tiny by modern standards, but large enough to generate coherent Shakespeare-like text.

---

## Training Loop

```python
# Data loading
def get_batch(split):
    data = train_data if split == 'train' else val_data
    ix = torch.randint(len(data) - block_size, (batch_size,))
    x = torch.stack([data[i:i+block_size] for i in ix])
    y = torch.stack([data[i+1:i+block_size+1] for i in ix])
    x, y = x.to(device), y.to(device)
    return x, y

@torch.no_grad()
def estimate_loss():
    out = {}
    model.eval()
    for split in ['train', 'val']:
        losses = torch.zeros(eval_iters)
        for k in range(eval_iters):
            X, Y = get_batch(split)
            logits, loss = model(X, Y)
            losses[k] = loss.item()
        out[split] = losses.mean()
    model.train()
    return out

# Initialize model and optimizer
model = GPT(vocab_size, n_embd, block_size, n_head, n_layer, dropout).to(device)
print(f"Parameters: {sum(p.numel() for p in model.parameters()) / 1e6:.2f}M")

optimizer = torch.optim.AdamW(model.parameters(), lr=learning_rate)

# Training
for iter in range(max_iters):
    if iter % eval_interval == 0:
        losses = estimate_loss()
        print(f"step {iter}: train loss {losses['train']:.4f}, val loss {losses['val']:.4f}")

    xb, yb = get_batch('train')
    logits, loss = model(xb, yb)
    optimizer.zero_grad(set_to_none=True)
    loss.backward()
    optimizer.step()

# Generate text
context = torch.zeros((1, 1), dtype=torch.long, device=device)
print(decode(model.generate(context, max_new_tokens=500)[0].tolist()))
```

---

## Understanding Attention Patterns

After training, you can visualize what the attention heads have learned:

```python
def visualize_attention(model, text, layer=0, head=0):
    """Visualize attention weights for a given input."""
    tokens = encode(text)
    x = torch.tensor([tokens], device=device)

    # Forward through embeddings
    tok_emb = model.token_embedding_table(x)
    pos_emb = model.position_embedding_table(torch.arange(len(tokens), device=device))
    x_emb = tok_emb + pos_emb

    # Forward through blocks, capturing attention weights
    # (requires modifying the Head class to store weights)
    # ...

    plt.figure(figsize=(10, 10))
    plt.imshow(attention_weights, cmap='viridis')
    plt.xlabel('Key position')
    plt.ylabel('Query position')
    plt.title(f'Attention weights — Layer {layer}, Head {head}')
    plt.colorbar()
    plt.show()
```

Common patterns you will observe:
- **Heads that attend to the previous token** — learning bigram-like patterns
- **Heads that attend to the first token** — anchoring on the start of the sequence
- **Heads with diagonal patterns** — tracking positional relationships
- **Heads that attend to specific token types** — learning syntactic roles

---

## Exercises

### Exercise 1: Ablation Study

Remove one component at a time and measure the impact on validation loss:
- Remove positional embeddings
- Remove residual connections
- Remove layer normalization
- Use a single attention head instead of multi-head
- Remove the feed-forward network

Which component matters most?

### Exercise 2: Scaling Laws

Train models of different sizes and plot loss vs. parameter count:

| Config | n_embd | n_head | n_layer | Params |
|--------|--------|--------|---------|--------|
| Tiny   | 64     | 2      | 2       | ~0.1M  |
| Small  | 128    | 4      | 4       | ~1M    |
| Medium | 384    | 6      | 6       | ~10M   |
| Large  | 768    | 12     | 12      | ~85M   |

Do you observe the power-law scaling relationship described in [Kaplan et al., 2020](https://arxiv.org/abs/2001.08361)?

### Exercise 3: Different Datasets

Train nanoGPT on different text corpora:
- Shakespeare (the default)
- Wikipedia articles
- Source code (Python, JavaScript)
- Song lyrics

How does the dataset affect the model's generation? How does the optimal tokenization change?

### Exercise 4: KV-Cache for Fast Inference

During generation, the model recomputes attention for all previous tokens at every step. Implement a **KV-cache** that stores computed key-value pairs and only computes attention for the new token:

```python
@torch.no_grad()
def generate_with_cache(self, idx, max_new_tokens):
    kv_cache = [None] * len(self.blocks)
    for _ in range(max_new_tokens):
        # Only process the last token, using cached K and V
        # ...
        pass
    return idx
```

Measure the speedup for long sequences.

### Exercise 5: Rotary Position Embeddings (RoPE)

Replace the learned positional embeddings with **Rotary Position Embeddings**:

```python
def apply_rotary_emb(q, k, freqs_cis):
    """Apply rotary embeddings to queries and keys."""
    q_complex = torch.view_as_complex(q.reshape(*q.shape[:-1], -1, 2))
    k_complex = torch.view_as_complex(k.reshape(*k.shape[:-1], -1, 2))
    q_rotated = torch.view_as_real(q_complex * freqs_cis).flatten(-2)
    k_rotated = torch.view_as_real(k_complex * freqs_cis).flatten(-2)
    return q_rotated, k_rotated
```

Compare the model's ability to generalize to longer sequences than it was trained on.

---

## Key Takeaways

- **Self-attention** computes a weighted sum of all positions, with learned weights that depend on content
- **Multi-head attention** lets the model attend to different types of information simultaneously
- **Positional encoding** is necessary because attention is permutation-invariant
- **Residual connections and layer normalization** are essential for training deep transformers
- **The transformer block** is just attention (communication) followed by feed-forward (computation), repeated
- **nanoGPT is a real GPT** — the same architecture as GPT-2/3, just smaller. The concepts scale directly.

---

## Resources

- [Andrej Karpathy: Let's build GPT from scratch](https://www.youtube.com/watch?v=kCc8FmEb1nY) — the lecture this week is based on
- [nanoGPT source code](https://github.com/karpathy/nanoGPT) — the complete implementation
- [Vaswani et al., 2017: Attention Is All You Need](https://arxiv.org/abs/1706.03762) — the original transformer paper
- [The Illustrated Transformer](https://jalammar.github.io/illustrated-transformer/) — Jay Alammar's visual walkthrough
- [The Annotated Transformer](https://nlp.seas.harvard.edu/annotated-transformer/) — Harvard NLP's line-by-line annotation

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/week-14/">← Week 14: makemore</a> · Next: <a href="/level-3/week-16/">Week 16: Pre-training on Custom Data →</a>
</p>
