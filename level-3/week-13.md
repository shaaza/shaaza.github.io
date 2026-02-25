---
layout: page
title: "Week 13: micrograd — Backpropagation from Scratch"
description: "Build a tiny autograd engine from scratch. Understand computational graphs, automatic differentiation, and gradients at the deepest level."
permalink: /level-3/week-13/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 13'
---

## Why Start Here

Every modern neural network — from the smallest classifier to the largest language model — learns by the same fundamental mechanism: **backpropagation**. Frameworks like PyTorch and TensorFlow hide this mechanism behind convenient APIs, but if you cannot explain what happens when you call `loss.backward()`, you are operating on faith.

This week, you build [micrograd](https://github.com/karpathy/micrograd), a tiny autograd engine that implements backpropagation over a dynamically built computational graph. It supports scalar-valued operations only — no tensors, no GPU, no optimization tricks. Just the raw algorithm, exposed and understandable.

By the end of this week, you will have implemented the same core mechanism that powers every deep learning framework in existence.

---

## Core Concepts

### What Is a Computational Graph?

Every mathematical expression can be represented as a **directed acyclic graph (DAG)** where:

- **Leaf nodes** are inputs (data, parameters)
- **Internal nodes** are operations (+, *, tanh, etc.)
- **Edges** carry values forward and gradients backward

Consider the expression `L = (a * b + c).tanh()`:

```
a ──┐
    ├── * ──┐
b ──┘       ├── + ── tanh ── L
c ──────────┘
```

The **forward pass** computes `L` from the inputs. The **backward pass** computes `dL/da`, `dL/db`, and `dL/dc` — the gradients that tell us how to adjust each input to reduce the loss.

### The Chain Rule

Backpropagation is just the chain rule applied recursively through the computational graph:

$$\frac{dL}{da} = \frac{dL}{d\text{tanh}} \cdot \frac{d\text{tanh}}{d\text{sum}} \cdot \frac{d\text{sum}}{d\text{prod}} \cdot \frac{d\text{prod}}{da}$$

Each node only needs to know its **local derivative** — how its output changes with respect to its inputs. The global gradient is assembled by multiplying local derivatives along the path from output to input.

### Automatic Differentiation

**Automatic differentiation** (autodiff) is not symbolic differentiation (manipulating algebraic expressions) and it is not numerical differentiation (finite differences with epsilon perturbations). It is the exact computation of derivatives by accumulating local gradients through the graph.

There are two modes:
- **Forward mode**: propagate derivatives from inputs to outputs (efficient when few inputs)
- **Reverse mode**: propagate derivatives from outputs to inputs (efficient when few outputs)

Neural network training has one scalar output (the loss) and millions of parameters, so **reverse mode** — backpropagation — is the right choice.

---

## Building micrograd Step by Step

### Step 1: The Value Class

The foundation of micrograd is a `Value` object that wraps a scalar and tracks the computational graph:

```python
class Value:
    def __init__(self, data, _children=(), _op=''):
        self.data = data
        self.grad = 0.0
        self._backward = lambda: None
        self._prev = set(_children)
        self._op = _op

    def __repr__(self):
        return f"Value(data={self.data}, grad={self.grad})"
```

Every `Value` stores:
- `data` — the actual scalar value
- `grad` — the derivative of the loss with respect to this value (initialized to 0)
- `_backward` — a function that computes the local gradient contribution
- `_prev` — the set of `Value` objects that produced this one
- `_op` — a label for the operation (for debugging and visualization)

### Step 2: Arithmetic Operations

Each operation creates a new `Value` and defines its `_backward` function:

```python
def __add__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data + other.data, (self, other), '+')

    def _backward():
        self.grad += out.grad    # d(a+b)/da = 1, so grad accumulates out.grad * 1
        other.grad += out.grad   # d(a+b)/db = 1

    out._backward = _backward
    return out

def __mul__(self, other):
    other = other if isinstance(other, Value) else Value(other)
    out = Value(self.data * other.data, (self, other), '*')

    def _backward():
        self.grad += other.data * out.grad   # d(a*b)/da = b
        other.grad += self.data * out.grad   # d(a*b)/db = a

    out._backward = _backward
    return out
```

Notice the `+=` in gradient accumulation. A value might be used in multiple operations, and its total gradient is the **sum** of contributions from all paths. This is the multivariate chain rule.

### Step 3: More Operations

You need power, negation, subtraction, division, and activation functions:

```python
def __pow__(self, other):
    assert isinstance(other, (int, float)), "only supporting int/float powers"
    out = Value(self.data ** other, (self,), f'**{other}')

    def _backward():
        self.grad += (other * self.data ** (other - 1)) * out.grad

    out._backward = _backward
    return out

def tanh(self):
    x = self.data
    t = (math.exp(2*x) - 1) / (math.exp(2*x) + 1)
    out = Value(t, (self,), 'tanh')

    def _backward():
        self.grad += (1 - t**2) * out.grad  # d(tanh)/dx = 1 - tanh^2(x)

    out._backward = _backward
    return out

def relu(self):
    out = Value(0 if self.data < 0 else self.data, (self,), 'ReLU')

    def _backward():
        self.grad += (out.data > 0) * out.grad

    out._backward = _backward
    return out

def __neg__(self):
    return self * -1

def __sub__(self, other):
    return self + (-other)

def __truediv__(self, other):
    return self * other**-1
```

### Step 4: Topological Sort and Backpropagation

To run backpropagation, you need to process nodes in reverse topological order — ensuring every node's gradient is fully accumulated before propagating further backward:

```python
def backward(self):
    topo = []
    visited = set()

    def build_topo(v):
        if v not in visited:
            visited.add(v)
            for child in v._prev:
                build_topo(child)
            topo.append(v)

    build_topo(self)

    self.grad = 1.0  # dL/dL = 1
    for v in reversed(topo):
        v._backward()
```

The `self.grad = 1.0` line is the seed: the derivative of the loss with respect to itself is 1.

### Step 5: Neural Network Layers

With the autograd engine complete, you can build neural network abstractions on top:

```python
import random

class Neuron:
    def __init__(self, nin):
        self.w = [Value(random.uniform(-1, 1)) for _ in range(nin)]
        self.b = Value(0)

    def __call__(self, x):
        act = sum((wi * xi for wi, xi in zip(self.w, x)), self.b)
        return act.tanh()

    def parameters(self):
        return self.w + [self.b]

class Layer:
    def __init__(self, nin, nout):
        self.neurons = [Neuron(nin) for _ in range(nout)]

    def __call__(self, x):
        out = [n(x) for n in self.neurons]
        return out[0] if len(out) == 1 else out

    def parameters(self):
        return [p for n in self.neurons for p in n.parameters()]

class MLP:
    def __init__(self, nin, nouts):
        sz = [nin] + nouts
        self.layers = [Layer(sz[i], sz[i+1]) for i in range(len(nouts))]

    def __call__(self, x):
        for layer in self.layers:
            x = layer(x)
        return x

    def parameters(self):
        return [p for layer in self.layers for p in layer.parameters()]
```

### Step 6: Training Loop

Now you can train a small network:

```python
# Create a simple dataset
xs = [
    [2.0, 3.0, -1.0],
    [3.0, -1.0, 0.5],
    [0.5, 1.0, 1.0],
    [1.0, 1.0, -1.0],
]
ys = [1.0, -1.0, -1.0, 1.0]  # desired targets

# Build a network: 3 inputs -> 4 hidden -> 4 hidden -> 1 output
model = MLP(3, [4, 4, 1])

# Training loop
for k in range(100):
    # Forward pass
    ypred = [model(x) for x in xs]
    loss = sum((yout - ygt)**2 for ygt, yout in zip(ys, ypred))

    # Zero gradients
    for p in model.parameters():
        p.grad = 0.0

    # Backward pass
    loss.backward()

    # Update parameters (gradient descent)
    learning_rate = 0.05
    for p in model.parameters():
        p.data += -learning_rate * p.grad

    if k % 10 == 0:
        print(f"step {k}, loss = {loss.data:.4f}")
```

---

## Exercises

### Exercise 1: Verify Gradients Numerically

For any expression, verify that micrograd's gradients match numerical derivatives:

```python
def numerical_grad(f, x, h=1e-6):
    return (f(x + h).data - f(x - h).data) / (2 * h)

# Test with a simple expression
a = Value(2.0)
b = Value(-3.0)
c = Value(10.0)
d = a * b + c
d.backward()

# Compare a.grad with numerical derivative
print(f"Analytical: {a.grad}")
print(f"Numerical:  {numerical_grad(lambda a: a * Value(-3.0) + Value(10.0), Value(2.0))}")
```

### Exercise 2: Add More Operations

Extend micrograd with:
- `exp()` — the exponential function (local derivative: `exp(x)`)
- `log()` — the natural logarithm (local derivative: `1/x`)
- `sigmoid()` — implement as `1 / (1 + exp(-x))`

Test each by comparing analytical gradients with numerical gradients.

### Exercise 3: Binary Cross-Entropy Loss

Implement binary cross-entropy loss using your `log` and `sigmoid`:

```python
def binary_cross_entropy(y_pred, y_true):
    """
    y_pred: Value (raw logit)
    y_true: 0 or 1
    """
    p = y_pred.sigmoid()
    return -(y_true * p.log() + (1 - y_true) * (Value(1) - p).log())
```

Train the MLP using this loss instead of MSE. Compare convergence behavior.

### Exercise 4: Visualize the Graph

Use [graphviz](https://graphviz.readthedocs.io/) to draw the computational graph:

```python
from graphviz import Digraph

def trace(root):
    nodes, edges = set(), set()
    def build(v):
        if v not in nodes:
            nodes.add(v)
            for child in v._prev:
                edges.add((child, v))
                build(child)
    build(root)
    return nodes, edges

def draw_dot(root):
    dot = Digraph(format='svg', graph_attr={'rankdir': 'LR'})
    nodes, edges = trace(root)
    for n in nodes:
        uid = str(id(n))
        dot.node(uid, label=f"{{ data {n.data:.4f} | grad {n.grad:.4f} }}", shape='record')
        if n._op:
            dot.node(uid + n._op, label=n._op)
            dot.edge(uid + n._op, uid)
    for n1, n2 in edges:
        dot.edge(str(id(n1)), str(id(n2)) + n2._op)
    return dot
```

Visualize the forward and backward passes for a small expression.

### Exercise 5: Compare with PyTorch

Rewrite the same training loop using PyTorch tensors. Verify that:
- The forward pass produces the same values
- The gradients match micrograd's gradients exactly
- The training converges similarly

This exercise demonstrates that PyTorch's autograd is doing *exactly* the same thing as micrograd, just with tensors instead of scalars.

---

## Key Takeaways

- **Backpropagation is the chain rule applied recursively** through a computational graph. There is nothing more to it.
- **Gradients accumulate** when a value participates in multiple operations (`+=`, not `=`).
- **Topological sort** ensures gradients are fully computed before propagating further backward.
- **Every deep learning framework** implements this same algorithm — micrograd just makes it visible.
- **Zeroing gradients** between training steps is essential. Forgetting this is one of the most common bugs in deep learning code.

---

## Resources

- [Andrej Karpathy: The spelled-out intro to neural networks and backpropagation](https://www.youtube.com/watch?v=VMj-3S1tku0) — the lecture this week is based on
- [micrograd source code](https://github.com/karpathy/micrograd) — the complete implementation in ~100 lines
- [3Blue1Brown: What is backpropagation really doing?](https://www.youtube.com/watch?v=Ilg3gGewQ5U) — visual intuition for gradients
- [Calculus on Computational Graphs: Backpropagation](https://colah.github.io/posts/2015-08-Backprop/) — Chris Olah's excellent illustrated explanation

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/">← Level 3 Overview</a> · Next: <a href="/level-3/week-14/">Week 14: makemore — Character-Level Language Models →</a>
</p>
