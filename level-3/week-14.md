---
layout: page
title: "Week 14: makemore — Character-Level Language Models"
description: "Build character-level language models from scratch. Understand tokenization, bigrams, MLPs, and how language models predict the next token."
permalink: /level-3/week-14/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 14'
---

## From Gradients to Language

Last week you built an autograd engine that can train a neural network on any differentiable loss function. This week you point that machinery at the problem that defines modern AI: **predicting the next token**.

[makemore](https://github.com/karpathy/makemore) is a character-level language model. Given a dataset of names, it learns the statistical patterns of character sequences and generates new, plausible names. The problem is simple enough to understand completely, but the techniques generalize directly to GPT-scale language models.

You will implement three increasingly powerful approaches:
1. **Bigram model** — count-based statistics, no neural network
2. **MLP model** — a neural network that looks at a fixed context window
3. **Sampling strategies** — controlling generation quality and diversity

---

## Core Concepts

### Tokenization

Before a model can process text, it must convert characters (or words, or subwords) into numbers. This is **tokenization**.

For makemore, the tokenization is simple: each character maps to an integer.

```python
chars = sorted(list(set(''.join(words))))
stoi = {s: i + 1 for i, s in enumerate(chars)}
stoi['.'] = 0  # special start/end token
itos = {i: s for s, i in stoi.items()}

# Example
print(stoi)  # {'.' : 0, 'a': 1, 'b': 2, ..., 'z': 27}
print(itos)  # {0: '.', 1: 'a', 2: 'b', ..., 27: 'z'}
```

The `.` token marks the beginning and end of a name. This is the character-level equivalent of the `<BOS>` and `<EOS>` tokens used in large language models.

Real-world models use more sophisticated tokenization (BPE, SentencePiece, tiktoken), but the principle is identical: convert raw text into a sequence of integers that the model can process.

### The Language Modeling Task

A language model learns the probability distribution:

**P(next character | previous characters)**

Given the name "emma":
- P(e | .) — how likely is 'e' as the first character?
- P(m | .e) — how likely is 'm' after 'e' at the start?
- P(m | .em) — how likely is another 'm'?
- P(a | .emm) — how likely is 'a' after 'emm'?
- P(. | .emma) — how likely is the name to end here?

The model is trained to maximize these probabilities across all names in the dataset. During generation, it samples from the learned distribution one character at a time.

---

## Approach 1: Bigram Model

The simplest language model looks at only the **previous one character** to predict the next. This is a bigram model.

### Counting Bigrams

```python
import torch

# Load dataset
words = open('names.txt', 'r').read().splitlines()

# Count all bigram occurrences
N = torch.zeros((28, 28), dtype=torch.int32)

for w in words:
    chs = ['.'] + list(w) + ['.']
    for ch1, ch2 in zip(chs, chs[1:]):
        ix1 = stoi[ch1]
        ix2 = stoi[ch2]
        N[ix1, ix2] += 1
```

### Visualizing the Counts

```python
import matplotlib.pyplot as plt

plt.figure(figsize=(16, 16))
plt.imshow(N, cmap='Blues')
for i in range(28):
    for j in range(28):
        chstr = itos[i] + itos[j]
        plt.text(j, i, chstr, ha="center", va="bottom", color='gray')
        plt.text(j, i, N[i, j].item(), ha="center", va="top", color='gray')
plt.axis('off')
plt.title('Bigram counts')
plt.show()
```

This visualization immediately reveals patterns: names rarely start with 'x', 'q' is almost always followed by 'u', and certain character pairs like 'th' are far more common than others.

### Converting Counts to Probabilities

```python
# Add smoothing to avoid zero probabilities
P = (N + 1).float()
P = P / P.sum(dim=1, keepdim=True)  # normalize each row

# Now P[i, j] = P(character j | character i)
```

The `+1` is **Laplace smoothing** — it ensures no bigram has zero probability, which would cause the model to assign negative infinity log-likelihood to any name containing that bigram.

### Sampling from the Bigram Model

```python
g = torch.Generator().manual_seed(2147483647)

for i in range(10):
    out = []
    ix = 0  # start with '.'
    while True:
        p = P[ix]
        ix = torch.multinomial(p, num_samples=1, replacement=True, generator=g).item()
        if ix == 0:  # end token
            break
        out.append(itos[ix])
    print(''.join(out))
```

The bigram model generates names like "mor", "axi", "minaymorede" — recognizably name-like in their character patterns, but often implausible because the model has no memory beyond one character.

### Evaluating with Negative Log-Likelihood

```python
log_likelihood = 0.0
n = 0

for w in words:
    chs = ['.'] + list(w) + ['.']
    for ch1, ch2 in zip(chs, chs[1:]):
        ix1 = stoi[ch1]
        ix2 = stoi[ch2]
        prob = P[ix1, ix2]
        logprob = torch.log(prob)
        log_likelihood += logprob
        n += 1

nll = -log_likelihood / n
print(f"Average negative log-likelihood: {nll:.4f}")
```

The negative log-likelihood (NLL) is the standard metric for language models. Lower is better. A perfect model would have NLL = 0 (it predicts every next character with certainty). A uniform random model over 27 characters would have NLL = log(27) = 3.30.

---

## Approach 2: Neural Network (MLP)

The bigram model is limited to one character of context. A neural network can learn to use a **fixed window** of previous characters.

### Building the Dataset

```python
block_size = 3  # context length: how many characters we use to predict the next one

def build_dataset(words):
    X, Y = [], []
    for w in words:
        context = [0] * block_size  # start with '...'
        for ch in w + '.':
            ix = stoi[ch]
            X.append(context)
            Y.append(ix)
            context = context[1:] + [ix]  # slide the window
    X = torch.tensor(X)
    Y = torch.tensor(Y)
    return X, Y

# Split into train/dev/test
import random
random.seed(42)
random.shuffle(words)
n1 = int(0.8 * len(words))
n2 = int(0.9 * len(words))

Xtr, Ytr = build_dataset(words[:n1])    # 80% train
Xdev, Ydev = build_dataset(words[n1:n2]) # 10% dev
Xte, Yte = build_dataset(words[n2:])     # 10% test
```

### The Embedding Layer

Characters are integers, but neural networks work with continuous vectors. An **embedding layer** maps each integer to a learned vector:

```python
C = torch.randn((27, 10))  # 27 characters, each embedded in 10 dimensions

# Look up embeddings for a batch
emb = C[X]  # shape: (batch_size, block_size, 10)
```

This is equivalent to a one-hot encoding followed by a matrix multiply, but the lookup is faster and more memory-efficient.

### The MLP Architecture

```python
g = torch.Generator().manual_seed(2147483647)

n_embd = 10    # embedding dimensionality
n_hidden = 200  # hidden layer size

C = torch.randn((27, n_embd),            generator=g)
W1 = torch.randn((n_embd * block_size, n_hidden), generator=g) * 0.2
b1 = torch.randn(n_hidden,               generator=g) * 0.01
W2 = torch.randn((n_hidden, 27),         generator=g) * 0.01
b2 = torch.randn(27,                     generator=g) * 0

parameters = [C, W1, b1, W2, b2]
for p in parameters:
    p.requires_grad = True

print(f"Total parameters: {sum(p.nelement() for p in parameters)}")
```

### Training Loop

```python
for i in range(200000):
    # Mini-batch
    ix = torch.randint(0, Xtr.shape[0], (32,))

    # Forward pass
    emb = C[Xtr[ix]]                          # (32, 3, 10)
    h = torch.tanh(emb.view(-1, 30) @ W1 + b1)  # (32, 200)
    logits = h @ W2 + b2                       # (32, 27)
    loss = torch.nn.functional.cross_entropy(logits, Ytr[ix])

    # Backward pass
    for p in parameters:
        p.grad = None
    loss.backward()

    # Update
    lr = 0.1 if i < 100000 else 0.01
    for p in parameters:
        p.data += -lr * p.grad

    if i % 10000 == 0:
        print(f"step {i:7d}, loss = {loss.item():.4f}")
```

### Evaluating the MLP

```python
@torch.no_grad()
def eval_split(X, Y):
    emb = C[X]
    h = torch.tanh(emb.view(-1, 30) @ W1 + b1)
    logits = h @ W2 + b2
    loss = torch.nn.functional.cross_entropy(logits, Y)
    return loss.item()

print(f"Train loss: {eval_split(Xtr, Ytr):.4f}")
print(f"Dev loss:   {eval_split(Xdev, Ydev):.4f}")
```

The MLP should achieve a significantly lower NLL than the bigram model — typically around 2.1-2.2 versus the bigram's 2.4-2.5.

---

## Sampling Strategies

How you sample from the model's probability distribution dramatically affects generation quality.

### Temperature Scaling

```python
def sample(model_fn, temperature=1.0, max_len=20):
    context = [0] * block_size
    out = []
    for _ in range(max_len):
        logits = model_fn(context)
        # Temperature scaling: divide logits by temperature before softmax
        probs = torch.softmax(logits / temperature, dim=-1)
        ix = torch.multinomial(probs, num_samples=1).item()
        if ix == 0:
            break
        out.append(itos[ix])
        context = context[1:] + [ix]
    return ''.join(out)
```

| Temperature | Effect | Example Output |
|---|---|---|
| 0.5 | More conservative, picks high-probability characters | Common, safe names |
| 1.0 | Samples from the learned distribution as-is | Balanced variety |
| 1.5 | More random, explores low-probability characters | Unusual, creative names |

### Top-k Sampling

```python
def sample_topk(model_fn, k=5, max_len=20):
    context = [0] * block_size
    out = []
    for _ in range(max_len):
        logits = model_fn(context)
        # Zero out everything except top-k
        topk_vals, topk_idx = torch.topk(logits, k)
        probs = torch.zeros_like(logits)
        probs[topk_idx] = torch.softmax(topk_vals, dim=-1)
        ix = torch.multinomial(probs, num_samples=1).item()
        if ix == 0:
            break
        out.append(itos[ix])
        context = context[1:] + [ix]
    return ''.join(out)
```

Top-k restricts sampling to the `k` most likely next characters, preventing the model from ever picking very unlikely characters that would make the output incoherent.

---

## Exercises

### Exercise 1: Trigram Count Model

Build a trigram count model (using the previous two characters). Compare its NLL to the bigram model. How much does the extra character of context help?

### Exercise 2: Hyperparameter Search

Systematically vary the MLP hyperparameters and measure dev set loss:
- Embedding dimensionality: 2, 5, 10, 20
- Hidden layer size: 50, 100, 200, 500
- Context length: 2, 3, 5, 8
- Learning rate schedule

Plot the results. What matters most?

### Exercise 3: Embedding Visualization

After training, extract the 2D embeddings (set `n_embd = 2`) and plot them:

```python
plt.figure(figsize=(8, 8))
plt.scatter(C[:, 0].data, C[:, 1].data, s=200)
for i in range(27):
    plt.annotate(itos[i], (C[i, 0].item(), C[i, 1].item()), ha='center', va='center')
plt.grid(True)
plt.title('Character embeddings (2D)')
plt.show()
```

Do the embeddings capture meaningful structure? Are vowels clustered together? Consonants?

### Exercise 4: Batch Normalization

Add batch normalization between layers. Measure the effect on:
- Training speed (steps to reach a target loss)
- Final loss
- Gradient magnitudes (are they more uniform across layers?)

### Exercise 5: Deeper Networks

Stack 2-3 hidden layers instead of one. Does depth help for this problem? At what point do you encounter vanishing gradients?

---

## Key Takeaways

- **Language modeling is next-token prediction** — this is the fundamental task behind GPT, Claude, and all modern LLMs
- **Embeddings** convert discrete tokens into continuous vectors that neural networks can process
- **Cross-entropy loss** is the standard objective for classification (including next-token prediction)
- **Train/dev/test splits** are essential for detecting overfitting — never evaluate on training data
- **Sampling temperature** trades off between quality (low temperature) and diversity (high temperature)
- **Context window size** directly affects model quality — more context means better predictions, up to a point

---

## Resources

- [Andrej Karpathy: The spelled-out intro to language modeling](https://www.youtube.com/watch?v=PaCmpygFfXo) — building makemore part 1
- [Andrej Karpathy: Building makemore Part 2 — MLP](https://www.youtube.com/watch?v=TCH_1BHY58I) — the MLP approach
- [makemore source code](https://github.com/karpathy/makemore) — the complete implementation
- [Bengio et al., 2003: A Neural Probabilistic Language Model](https://www.jmlr.org/papers/volume3/bengio03a/bengio03a.pdf) — the paper that introduced neural language models

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/week-13/">← Week 13: micrograd</a> · Next: <a href="/level-3/week-15/">Week 15: nanoGPT — Building Transformers →</a>
</p>
