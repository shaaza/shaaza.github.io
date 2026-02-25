---
layout: page
title: "Week 18: Edge Deployment & Final Project"
description: "Run models on constrained hardware. Quantization, ONNX export, latency optimization. Final capstone project spanning all levels."
permalink: /level-3/week-18/
breadcrumb: '<a href="/">Schedule</a> / <a href="/level-3/">Level 3</a> / Week 18'
---

## From Data Center to Device

You have built models (micrograd, makemore, nanoGPT), trained them on custom data, and fine-tuned them for specific tasks. But so far, everything has run on development machines with powerful GPUs and no latency constraints.

Real products often need models to run on **constrained hardware**: mobile phones, embedded devices, web browsers, or servers with strict latency budgets. A 7-billion parameter model that takes 5 seconds to generate a response is unusable for real-time applications, regardless of how good its outputs are.

This week you learn the techniques that bridge the gap between a trained model and a deployed product: **quantization**, **model export**, **latency optimization**, and **hardware-aware inference**. Then you bring everything in the course together in a final capstone project.

---

## Core Concepts

### Why Edge Deployment Is Hard

A typical language model's constraints:

| Resource | Training | Edge Inference |
|----------|----------|----------------|
| **GPU Memory** | 24-80 GB | 4-8 GB (or CPU only) |
| **Compute** | Hours to days acceptable | < 100ms per token required |
| **Power** | Unlimited | Battery-constrained |
| **Model Size** | Stored on disk, loaded once | Must fit in limited RAM |
| **Batch Size** | 32-128 | Usually 1 (single user) |

The fundamental challenge: models are trained with float32 precision (4 bytes per parameter), massive batch sizes, and unlimited time. Inference on the edge requires smaller, faster, leaner representations.

### The Optimization Stack

```
Trained Model (float32, full size)
    │
    ├── Quantization (reduce precision: float16, int8, int4)
    │
    ├── Pruning (remove unnecessary weights)
    │
    ├── Distillation (train a smaller model to mimic the larger one)
    │
    ├── Export (convert to optimized runtime: ONNX, TensorRT, Core ML)
    │
    └── Runtime optimization (operator fusion, memory planning, batching)
```

Each technique trades a small amount of quality for a large improvement in speed or size.

---

## Quantization

Quantization reduces the numerical precision of model weights and activations. It is the single most impactful optimization for edge deployment.

### Post-Training Quantization (PTQ)

Convert a trained float32 model to lower precision without retraining:

```python
import torch

def quantize_to_int8(tensor):
    """Symmetric int8 quantization."""
    scale = tensor.abs().max() / 127.0
    quantized = torch.round(tensor / scale).clamp(-128, 127).to(torch.int8)
    return quantized, scale

def dequantize(quantized, scale):
    """Convert back to float for computation."""
    return quantized.float() * scale

# Example: quantize a weight matrix
W = torch.randn(768, 768)  # original float32 weights
W_q, scale = quantize_to_int8(W)

print(f"Original size:   {W.nelement() * 4 / 1024:.1f} KB")
print(f"Quantized size:  {W_q.nelement() * 1 / 1024:.1f} KB")
print(f"Compression:     {W.nelement() * 4 / (W_q.nelement() * 1):.1f}x")
print(f"Max error:       {(W - dequantize(W_q, scale)).abs().max():.6f}")
```

### Quantization Levels

| Precision | Bits per Weight | Model Size (7B params) | Typical Quality Loss |
|-----------|----------------|------------------------|---------------------|
| float32 | 32 | 28 GB | Baseline |
| float16 | 16 | 14 GB | Negligible |
| int8 | 8 | 7 GB | Very small |
| int4 | 4 | 3.5 GB | Small but measurable |
| int2 | 2 | 1.75 GB | Significant |

### GPTQ: Quantization-Aware Weight Compression

GPTQ is a state-of-the-art post-training quantization method that minimizes the quantization error layer by layer:

```python
# Using the auto-gptq library
from auto_gptq import AutoGPTQForCausalLM, BaseQuantizeConfig

quantize_config = BaseQuantizeConfig(
    bits=4,               # quantize to 4-bit
    group_size=128,       # quantize in groups of 128 weights
    desc_act=True,        # use activation-aware ordering
)

model = AutoGPTQForCausalLM.from_pretrained(
    model_name,
    quantize_config=quantize_config,
)

# Calibrate on a small dataset (needed for GPTQ)
model.quantize(calibration_data)

# Save quantized model
model.save_quantized("model-4bit-gptq")
```

### Quantization-Aware Training (QAT)

If PTQ causes too much quality loss, you can simulate quantization during training:

```python
class QuantizedLinear(torch.nn.Module):
    """Linear layer with simulated quantization during training."""

    def __init__(self, in_features, out_features, bits=8):
        super().__init__()
        self.linear = torch.nn.Linear(in_features, out_features)
        self.bits = bits
        self.qmin = -(2 ** (bits - 1))
        self.qmax = 2 ** (bits - 1) - 1

    def quantize_simulate(self, x):
        """Simulate quantization: quantize then immediately dequantize.
        The rounding error is real, but gradients flow through via STE."""
        scale = x.abs().max() / self.qmax
        x_q = torch.round(x / scale).clamp(self.qmin, self.qmax)
        x_deq = x_q * scale
        # Straight-Through Estimator: use dequantized in forward,
        # but pass gradients to original x
        return x + (x_deq - x).detach()

    def forward(self, x):
        w_q = self.quantize_simulate(self.linear.weight)
        return torch.nn.functional.linear(x, w_q, self.linear.bias)
```

---

## Model Export

### ONNX Export

ONNX (Open Neural Network Exchange) is a vendor-neutral format that lets you run models across different hardware and runtimes:

```python
import torch
import onnx
import onnxruntime as ort

# Export PyTorch model to ONNX
dummy_input = torch.randint(0, vocab_size, (1, block_size))

torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    input_names=['input_ids'],
    output_names=['logits'],
    dynamic_axes={
        'input_ids': {0: 'batch_size', 1: 'sequence_length'},
        'logits': {0: 'batch_size', 1: 'sequence_length'},
    },
    opset_version=17,
)

# Verify the exported model
onnx_model = onnx.load("model.onnx")
onnx.checker.check_model(onnx_model)
print("ONNX model is valid")

# Run inference with ONNX Runtime
session = ort.InferenceSession("model.onnx")
input_ids = dummy_input.numpy()
outputs = session.run(None, {"input_ids": input_ids})
logits = outputs[0]

print(f"Output shape: {logits.shape}")
```

### ONNX Runtime Optimization

```python
# Optimize the ONNX model
import onnxruntime as ort

# Session options for optimization
sess_options = ort.SessionOptions()
sess_options.graph_optimization_level = ort.GraphOptimizationLevel.ORT_ENABLE_ALL
sess_options.intra_op_num_threads = 4
sess_options.execution_mode = ort.ExecutionMode.ORT_PARALLEL

# Create optimized session
session = ort.InferenceSession(
    "model.onnx",
    sess_options=sess_options,
    providers=['CPUExecutionProvider'],  # or 'CUDAExecutionProvider'
)
```

### Other Export Targets

| Target | Use Case | Tools |
|--------|----------|-------|
| **ONNX** | Cross-platform, server/desktop | onnxruntime |
| **TensorRT** | NVIDIA GPUs, maximum throughput | torch-tensorrt |
| **Core ML** | Apple devices (iOS, macOS) | coremltools |
| **TFLite** | Android, microcontrollers | TensorFlow Lite |
| **llama.cpp** | CPU inference, quantized LLMs | llama-cpp-python |
| **WebAssembly** | Browser-based inference | onnxruntime-web |

---

## Latency Optimization

### Measuring Latency

Before optimizing, measure properly:

```python
import time
import numpy as np

def benchmark_inference(model, input_ids, n_warmup=10, n_runs=100):
    """Benchmark model inference latency."""
    # Warmup (important for GPU kernel caching)
    for _ in range(n_warmup):
        with torch.no_grad():
            _ = model(input_ids)

    # Timed runs
    if torch.cuda.is_available():
        torch.cuda.synchronize()

    latencies = []
    for _ in range(n_runs):
        start = time.perf_counter()
        with torch.no_grad():
            _ = model(input_ids)
        if torch.cuda.is_available():
            torch.cuda.synchronize()
        latencies.append(time.perf_counter() - start)

    latencies = np.array(latencies) * 1000  # convert to ms
    print(f"Latency: {np.mean(latencies):.2f} ms "
          f"(p50: {np.percentile(latencies, 50):.2f}, "
          f"p99: {np.percentile(latencies, 99):.2f})")
    return latencies
```

### Key Optimization Techniques

**1. KV-Cache (revisited from Week 15)**

The most important inference optimization for autoregressive models. Cache the key-value pairs from previous tokens:

```python
# Without KV-cache: O(n^2) per generated token
# With KV-cache: O(n) per generated token

# For a 1000-token generation:
# Without cache: ~500,000 attention computations
# With cache:    ~1,000 attention computations
```

**2. Operator Fusion**

Combine multiple small operations into a single kernel launch:

```python
# Before fusion: 3 separate GPU kernel launches
x = layer_norm(x)       # kernel 1
x = linear(x)           # kernel 2
x = gelu(x)             # kernel 3

# After fusion: 1 GPU kernel launch
x = fused_ln_linear_gelu(x)  # kernel 1
```

ONNX Runtime and TensorRT perform this automatically when given the full graph.

**3. Speculative Decoding**

Use a small, fast "draft" model to generate candidate tokens, then verify them in batch with the large model:

```python
def speculative_decode(target_model, draft_model, prompt, n_draft=5):
    """Generate tokens faster using speculative decoding."""
    tokens = prompt.clone()

    while True:
        # Draft model generates n candidates quickly
        draft_tokens = []
        draft_input = tokens.clone()
        for _ in range(n_draft):
            logits = draft_model(draft_input)
            next_token = logits[:, -1, :].argmax(dim=-1, keepdim=True)
            draft_tokens.append(next_token)
            draft_input = torch.cat([draft_input, next_token], dim=1)

        # Target model verifies all candidates in ONE forward pass
        candidate = torch.cat([tokens] + draft_tokens, dim=1)
        target_logits = target_model(candidate)

        # Accept tokens where draft and target agree
        n_accepted = 0
        for i, draft_token in enumerate(draft_tokens):
            target_token = target_logits[:, len(tokens) + i - 1, :].argmax(dim=-1)
            if target_token == draft_token.squeeze():
                n_accepted += 1
            else:
                break

        tokens = candidate[:, :len(tokens) + n_accepted + 1]
        # ... (handle end-of-sequence, etc.)
```

**4. Batched Inference**

If serving multiple users, batch their requests together:

```python
# Sequential: 10 users * 50ms = 500ms total
# Batched:    10 users in one batch * 80ms = 80ms total (6x throughput improvement)

# Continuous batching: add new requests to a running batch as slots free up
```

---

## Putting It All Together: Deployment Pipeline

```python
class ModelServer:
    """Simple model serving with optimization."""

    def __init__(self, model_path, quantize=True, use_onnx=True):
        if use_onnx:
            self.session = ort.InferenceSession(model_path)
            self.inference_fn = self._onnx_inference
        else:
            self.model = torch.load(model_path)
            if quantize:
                self.model = torch.quantization.quantize_dynamic(
                    self.model, {torch.nn.Linear}, dtype=torch.qint8
                )
            self.model.eval()
            self.inference_fn = self._pytorch_inference

    def _onnx_inference(self, input_ids):
        return self.session.run(None, {"input_ids": input_ids})[0]

    def _pytorch_inference(self, input_ids):
        with torch.no_grad():
            return self.model(torch.tensor(input_ids)).numpy()

    def generate(self, prompt_tokens, max_tokens=100, temperature=1.0):
        tokens = list(prompt_tokens)
        for _ in range(max_tokens):
            input_array = np.array([tokens[-block_size:]])
            logits = self.inference_fn(input_array)
            next_logits = logits[0, -1, :] / temperature
            probs = softmax(next_logits)
            next_token = np.random.choice(len(probs), p=probs)
            tokens.append(next_token)
            if next_token == eos_token_id:
                break
        return tokens
```

---

## The Final Project

The capstone project brings together everything you have learned across all levels of this course. Choose a project that spans at least three levels and demonstrates mastery of the full stack.

### Project Requirements

Your capstone must include:

1. **A trained or fine-tuned model** (Level 3) — you must have modified a model's weights, either through pre-training, fine-tuning, or LoRA adaptation

2. **An agentic workflow** (Level 2) — the model must be part of a larger system with prompt engineering, tool use, or multi-step reasoning

3. **A deployed application** (Level 1) — a working product that someone can use, not just a notebook

4. **An optimization** (this week) — at least one technique from this week: quantization, ONNX export, latency measurement, or edge deployment

5. **An evaluation** (Week 17) — rigorous measurement of your system's performance with appropriate metrics

### Project Ideas

**Idea 1: Domain-Specific Code Assistant**
- Fine-tune a model on a specific codebase (your company's code, a framework)
- Build an agentic workflow that retrieves relevant context before generating
- Deploy as a VS Code extension or CLI tool
- Quantize for fast local inference

**Idea 2: Document Intelligence Pipeline**
- Pre-train or fine-tune on domain documents (legal, medical, technical)
- Build an agent that ingests documents, indexes them, and answers questions
- Deploy with a web interface
- Measure latency and optimize for sub-second responses

**Idea 3: Edge ML on Embedded Hardware**
- Distill or quantize a model to run on a Raspberry Pi (connecting back to Level 0)
- Build a voice or text interface
- Measure power consumption, latency, and accuracy trade-offs
- Compare edge inference quality to cloud inference

**Idea 4: Multi-Agent System with Custom Models**
- Fine-tune specialized models for different sub-tasks (summarization, classification, generation)
- Orchestrate them with an agent framework from Level 2
- Deploy the complete pipeline with observability
- Quantize each model and measure end-to-end latency

### Evaluation Criteria

| Criterion | Weight | What We Look For |
|-----------|--------|-----------------|
| **Technical Depth** | 30% | Evidence of understanding the full stack, not just surface-level usage |
| **Working System** | 25% | The project runs, produces correct results, and handles edge cases |
| **Optimization** | 20% | Measured latency, model size, or quality trade-offs with supporting data |
| **Evaluation** | 15% | Rigorous metrics, baseline comparisons, honest assessment of limitations |
| **Documentation** | 10% | Clear explanation of design decisions, architecture, and results |

### Deliverables

1. **Source code** in a Git repository with a clear README
2. **A working demo** — either deployed or reproducible locally
3. **An evaluation report** with metrics, baselines, and analysis
4. **A 10-minute presentation** explaining your design decisions and results

---

## Exercises

### Exercise 1: Quantization Comparison

Take your nanoGPT model from Week 15 and quantize it to float16, int8, and int4. For each:
- Measure model size on disk
- Measure inference latency (tokens per second)
- Measure generation quality (perplexity on a held-out set)
- Generate 20 samples and manually assess quality

Plot the Pareto frontier of quality vs. speed.

### Exercise 2: ONNX Export and Benchmark

Export your model to ONNX format and compare inference performance:
- PyTorch (float32) vs. ONNX Runtime (float32)
- ONNX Runtime (float32) vs. ONNX Runtime (quantized int8)
- CPU vs. GPU (if available)

What is the total speedup from the combination of ONNX + quantization?

### Exercise 3: Latency Budget

Define a latency budget for a real-time chat application:
- Time to first token: < 200ms
- Inter-token latency: < 50ms
- Total response time (100 tokens): < 5 seconds

Determine the largest model you can serve within this budget on your hardware. What optimizations are needed?

### Exercise 4: llama.cpp Deployment

Use [llama.cpp](https://github.com/ggerganov/llama.cpp) to run a quantized model on CPU:

```bash
# Convert model to GGUF format
python convert.py model/ --outtype f16

# Quantize to 4-bit
./quantize model-f16.gguf model-q4_0.gguf q4_0

# Run inference
./main -m model-q4_0.gguf -p "Once upon a time" -n 100
```

Measure tokens per second. How does 4-bit CPU inference compare to float16 GPU inference?

### Exercise 5: Capstone Prototype

Build a minimal prototype of your capstone project:
1. Choose your project idea
2. Implement the core model pipeline (training/fine-tuning + inference)
3. Add one optimization technique
4. Measure at least one metric
5. Write a 1-page design document for the full project

This prototype becomes the foundation for your final deliverable.

---

## Key Takeaways

- **Quantization is the highest-leverage optimization** — int8 gives ~4x compression with minimal quality loss; int4 gives ~8x with measurable but often acceptable loss
- **ONNX enables cross-platform deployment** — train in PyTorch, deploy anywhere
- **Measure before optimizing** — latency benchmarks must include warmup, use percentiles (p50, p99), and account for GPU synchronization
- **KV-cache is mandatory** for autoregressive generation — without it, inference is quadratically slow
- **The full stack matters** — a great model with poor deployment is worse than a good model deployed well
- **This course started with bare metal and ends with intelligence** — you now understand every layer from silicon to language model

---

## Resources

- [llama.cpp](https://github.com/ggerganov/llama.cpp) — the reference implementation for CPU-based LLM inference
- [ONNX Runtime documentation](https://onnxruntime.ai/docs/) — comprehensive guide to ONNX optimization
- [GPTQ paper](https://arxiv.org/abs/2210.17323) — Accurate Post-Training Quantization for Generative Pre-trained Transformers
- [Speculative Decoding paper](https://arxiv.org/abs/2211.17192) — Fast Inference from Transformers via Speculative Decoding
- [MLPerf Inference benchmarks](https://mlcommons.org/en/inference-datacenter-31/) — industry-standard ML performance benchmarks

---

<p style="font-size: 0.9rem;">
  <a href="/level-3/week-17/">← Week 17: Fine-tuning & Evaluation</a> · <a href="/">Back to Schedule</a>
</p>
