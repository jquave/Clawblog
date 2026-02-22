# The Fastest SOTA Open Source Models You Can Run Locally (February 2026)

*Last updated: February 22, 2026*

The open source LLM landscape has shifted dramatically. Models that rival GPT-5 and Claude Opus are now running on consumer hardware — gaming PCs, MacBooks, even phones. This post breaks down what's actually fastest, what's actually good, and what you actually need to run them, backed by real benchmark data from Chatbot Arena, LiveCodeBench, Artificial Analysis, and community hardware tests.

---

## The Current Open Source Leaderboard

Before picking a model for local use, here's where open source stands against proprietary as of February 2026, per the [Chatbot Arena](https://arena.ai/leaderboard/text) (313 models, 5.3M+ human votes):

| Arena Rank | Model | Elo | License | Active Params |
|:---:|---|:---:|---|:---:|
| 1 | Claude Opus 4.6 (thinking) | 1505 | Proprietary | - |
| 3 | Gemini 3.1 Pro | 1500 | Proprietary | - |
| 8 | Grok 4.1 (thinking) | 1473 | Proprietary | - |
| **15** | **GLM-5** | **1452** | **MIT** | **44B** |
| **17** | **Kimi K2.5 (thinking)** | **1450** | **Modified MIT** | **-** |
| **21** | **Qwen3.5-397B-A17B** | **1449** | **Apache 2.0** | **17B** |
| **27** | **GLM-4.7** | **1441** | **MIT** | **~6B** |
| **38** | **GLM-4.6** | **1425** | **MIT** | **~6B** |

**The gap is ~50 Elo points** between the best proprietary and best open source model. That's meaningful but shrinking fast — it was 150+ points just a year ago.

### Quality Index Rankings (WhatLLM, Feb 2026)

| Rank | Model | Quality Index | LiveCodeBench | AIME 2025 | MMLU-Pro |
|:---:|---|:---:|:---:|:---:|:---:|
| 1 | GLM-5 (Reasoning) | 49.64 | - | - | - |
| 2 | Kimi K2.5 (Reasoning) | 46.73 | 85% | 96% | - |
| 3 | MiniMax-M2.5 | 41.97 | - | - | - |
| 4 | GLM-4.7 (Thinking) | 41.7 | 89% | 95% | 86% |
| 5 | DeepSeek V3.2 | 41.2 | 86% | 92% | 86% |
| 8 | MiMo-V2-Flash | 39.0 | 87% | 96% | 84% |
| 11 | DeepSeek V3.2 Speciale | 34.1 | 90% | 97% | 86% |

These are frontier-class scores. DeepSeek V3.2 Speciale hitting 90% on LiveCodeBench and 97% on AIME 2025 would have been unthinkable for open source a year ago.

---

## Trending on HuggingFace Right Now

What the community is actually downloading and using this week:

| # | Model | Author | Params | Downloads | Likes |
|:---:|---|---|:---:|:---:|:---:|
| 1 | Qwen3.5-397B-A17B | Qwen | 403B | 217K | 880 |
| 2 | personaplex-7b-v1 | NVIDIA | - | 540K | 2,138 |
| 3 | Nanbeige4.1-3B | Nanbeige | 3.9B | 154K | 711 |
| 4 | GLM-5 | zai-org | 754B | 178K | 1,427 |
| 5 | MiniMax-M2.5 | MiniMaxAI | 229B | 191K | 854 |
| 6 | Qwen3-TTS-12Hz-1.7B | Qwen | 1.7B | 982K | 1,141 |
| 7 | GLM-4.7-Flash | zai-org | ~30B | 1.77M | 1,558 |
| 8 | Llama-3.1-8B-Instruct | Meta | 8B | 5.9M | 5,480 |
| 9 | gpt-oss-120b | OpenAI | 120B | 3.5M | 4,513 |
| 10 | gpt-oss-20b | OpenAI | 20B | 5.6M | 4,388 |

Notable: OpenAI's first open-weight models (gpt-oss) are among the most downloaded models on the platform. Nanbeige4.1-3B — a 3B model that outperforms Qwen3-32B on Arena-Hard — is trending hard.

---

## Breakdown by Device Type

This is the part that matters. Not every model runs on every machine. Here's what to run based on what you actually own.

---

### Desktop: NVIDIA RTX 4090 / 5090 (24-32GB VRAM)

The RTX 4090 remains the most popular local inference GPU. The 5090 with 32GB GDDR7 extends what's possible.

| Model | Active Params | Quant | VRAM Used | tok/s | Quality Tier |
|---|:---:|---|:---:|:---:|---|
| **GLM-4.7-Flash** | ~6B | Q4 | ~16GB | **120-220** | Arena #27 (Elo 1441) |
| gpt-oss-20b | 3.6B | MXFP4 | ~10GB | ~80-100 | o3-mini parity |
| Nanbeige4.1-3B | 3B | Q4 | ~2GB | 150+ | Beats Qwen3-32B |
| Qwen 2.5 14B | 14B | Q4 | ~9GB | ~45-60 | Strong all-rounder |
| Qwen 2.5 32B | 32B | Q4 | ~20GB | ~30-40 | GPT-4o coding parity |
| Llama 4 Scout | 17B active | Q2 | ~24GB | ~20 | GPT-4 class (MoE) |
| Llama 3.1 8B | 8B | Q4_K_M | ~5GB | **100** | Solid baseline |

**Recommendation:** GLM-4.7-Flash is the clear winner. 120-220 tok/s at 4-bit on a 4090 with top-5 open source quality. It's fast enough that it feels like a cloud API.

**Speed reference (Artificial Analysis, measured):**

```
RTX 4090 (24GB) — 8B models @ Q4
├── Llama 3.1 8B:    ~100 tok/s
├── Granite 3.3 8B:  ~128 tok/s
└── GLM-4.7-Flash:   120-220 tok/s

RTX 4060 Ti (16GB) — 8B models @ Q4
├── Llama 3.1 8B:    ~89 tok/s
└── gpt-oss-20b:     ~60-80 tok/s

Intel Arc B580 (budget, $249)
└── 8B models @ Q4:  ~62 tok/s
```

---

### Desktop: Multi-GPU / Workstation (48-80GB+ VRAM)

If you have 2x RTX 3090s, an A100, or similar:

| Model | Active Params | Total Params | Quant | VRAM | tok/s | Quality Tier |
|---|:---:|:---:|---|:---:|:---:|---|
| gpt-oss-120b | 5.1B | 120B MoE | MXFP4 | ~80GB | **313** | Near o4-mini |
| Qwen3-Coder-Next | 3B | 80B MoE | Q4 | ~40GB | ~37 | 70%+ SWE-Bench |
| DeepSeek V3.2 | - | 685B MoE | - | Multi-GPU | - | Arena-competitive |
| GLM-5 | 44B | 754B MoE | Q2 | ~241GB | ~5-10 | #1 open source |

**Recommendation:** gpt-oss-120b at 313 tok/s on a single 80GB GPU is extraordinary — near o4-mini quality at insane speed. For coding, Qwen3-Coder-Next on 2x RTX 3090s (~$1,600 setup) gives you 37 tok/s with elite SWE-Bench scores.

---

### MacBook Pro / Mac Studio: Apple Silicon

Apple Silicon's unified memory architecture is the secret weapon for local LLMs. You can load models that would need 2-4 GPUs on NVIDIA — but bandwidth (not capacity) is the bottleneck.

**Memory bandwidth by chip:**

```
M3 Pro:    150 GB/s
M3 Max:    400 GB/s  <-- MacBook Pro ceiling
M3 Ultra:  800 GB/s  <-- Mac Studio
M4 Max:    546 GB/s
```

**Critical: Use MLX, not llama.cpp.** MLX is ~50% faster on Apple Silicon. For Qwen3-Coder-Next, that's 60 tok/s (MLX) vs 24 tok/s (llama.cpp).

#### M3 Max (MacBook Pro, 128GB Unified Memory, 400 GB/s)

| Model | Size (Quantized) | tok/s (MLX) | Quality Tier | Notes |
|---|:---:|:---:|---|---|
| **Nanbeige4.1-3B** | ~2GB | **100+** | Beats Qwen3-32B | Instant responses |
| **GLM-4.7-Flash** | ~16GB | **60-80** | Arena #27, top coding | Best daily driver |
| gpt-oss-20b | ~10GB | ~50-60 | o3-mini parity | Fits easily |
| Llama 3.1 8B | ~5GB (Q4) | ~72 | Baseline | Well-tested |
| Qwen 2.5 14B | ~9GB | ~35 | Strong multilingual | Good balance |
| Qwen 2.5 32B | ~20GB | ~15 | GPT-4o coding parity | Usable for coding |
| **Qwen3-Coder-Next** | ~40GB (Q4) | **~20** | 70%+ SWE-Bench | Your 128GB advantage |
| **gpt-oss-120b** | ~60-80GB | **~15-25** | Near o4-mini | Most can't run this |
| **MiniMax-M2.5** | ~101GB (Q3) | **~20** | Quality Index 41.97 | Pushes your RAM limit |
| Llama 3.3 70B | ~46GB (Q4) | ~8-12 | Proven workhorse | Slow but capable |

**Your 128GB gives you a massive advantage.** Most people max out at 8B-14B models. You can run gpt-oss-120b (near o4-mini quality) and MiniMax-M2.5 (frontier-tier) — models that normally need datacenter GPUs.

#### M3 Ultra (Mac Studio, 192GB, 800 GB/s)

| Model | Size (Quantized) | tok/s (MLX) | Notes |
|---|:---:|:---:|---|
| GLM-4.7-Flash | ~16GB | ~120-160 | Near-RTX 4090 speed |
| gpt-oss-120b | ~60-80GB | ~40-60 | Excellent on this chip |
| Llama 3.3 70B | ~46GB (Q4) | ~12-18 | Usable |
| Qwen 2.5 72B | ~50GB (Q4) | ~12-15 | Similar to 70B |
| DeepSeek R1 | large | ~17-18 | Frontier reasoning |

#### M4 Max (2025 MacBook Pro, 128GB, 546 GB/s)

| Model | Size (Quantized) | tok/s (MLX) | Notes |
|---|:---:|:---:|---|
| Llama 3.1 8B | ~5GB (Q4) | 70-83 | Fastest small model |
| GLM-4.7-Flash | ~16GB | ~80-100 | ~25% faster than M3 Max |
| Qwen3-Coder-Next | ~40GB | ~25-30 | Sweet spot for coding |
| Qwen 2.5 32B | ~20GB | ~25 | Solid mid-range |

---

### Laptop: 16GB RAM (No Dedicated GPU)

Budget laptops, older MacBooks, entry-level machines:

| Model | Params | Quant | RAM Used | tok/s | Notes |
|---|:---:|---|:---:|:---:|---|
| **Nanbeige4.1-3B** | 3B | Q4 | ~2-3GB | 10-50 (CPU) | Best quality at this size |
| **Phi-4 Mini Flash** | 3.8B | Q4 | ~3GB | Very fast | 10x throughput vs peers |
| Llama 3.2 3B | 3B | Q4 | ~2-3GB | 40-60 | Meta's small model |
| Mistral Small 7B | 7B | Q4 | ~6GB | ~30 | Best 16GB experience |
| gpt-oss-20b | 3.6B active | MXFP4 | ~10GB | ~15-20 | Pushes 16GB limit |

**Recommendation:** Nanbeige4.1-3B is the standout. A 3B model that outperforms 32B models on key benchmarks. Phi-4 Mini Flash is the alternative if you need maximum throughput on edge hardware.

---

### Mobile / Edge Devices

| Model | Params | Size | Notes |
|---|:---:|:---:|---|
| Phi-4 Mini Flash | 3.8B | ~2GB (Q4) | Purpose-built for edge; 10x throughput |
| Gemma 3n E4B | ~4B | ~2GB | Google's edge model; $0.03/M tokens |
| Llama 3.2 1B | 1B | ~700MB | Can outperform 8B with inference scaling |

---

## Speed vs. Quality: The Tradeoff Chart

Across all device types, here's how the key models stack up:

```
Quality (Arena Elo)
  ^
  |
1452 |  GLM-5 ............................................. (5-10 tok/s, multi-GPU only)
  |
1449 |  Qwen3.5-397B ..................................... (needs 8x GPU)
  |
1441 |  GLM-4.7 .................. * ..................... (60-220 tok/s, single GPU!)
  |
  |           gpt-oss-120b .... * ...................... (15-313 tok/s depending on HW)
  |
  |  MiniMax-M2.5 ......... * .......................... (20 tok/s on 128GB Mac)
  |
  |     gpt-oss-20b .............. * ................... (50-100 tok/s, fits 16GB)
  |
  |        Qwen3-Coder-Next ....... * .................. (20-37 tok/s, best for code)
  |
  |              Nanbeige4.1-3B ............ * .......... (100+ tok/s, only 3B!)
  |
  +-----+--------+--------+--------+--------+--------+--> Speed (tok/s)
       10       30       60      100      150      300
```

The sweet spot: **GLM-4.7-Flash** delivers top-5 open source quality at 60-220 tok/s on a single consumer GPU or Apple Silicon Mac. Nothing else combines quality and speed this well right now.

---

## How to Get Started

### Option 1: Ollama (Easiest)

```bash
# Install
curl -fsSL https://ollama.com/install.sh | sh

# Run the best models
ollama run glm-4.7-flash        # Best all-around
ollama run nanbeige4.1:3b       # Fastest small model
ollama run gpt-oss:20b          # OpenAI's open model
ollama run qwen3-coder-next     # Best for coding
```

### Option 2: LM Studio (Best GUI + MLX on Mac)

Download from [lmstudio.ai](https://lmstudio.ai). Supports MLX backend for Apple Silicon (50% faster than llama.cpp). Browse and download models directly.

### Option 3: MLX (Fastest on Apple Silicon)

```bash
pip install mlx-lm
mlx_lm.generate --model mlx-community/GLM-4.7-Flash-4bit --prompt "Hello"
```

### Option 4: llama.cpp (Most Flexible)

```bash
brew install llama.cpp  # or build from source
# Download GGUF from huggingface, then:
llama-cli -m model.gguf -p "Your prompt"
```

---

## Key Takeaways

1. **GLM-4.7-Flash is the current sweet spot** — top-5 open source quality at 60-220 tok/s on a single GPU or Mac. MIT licensed.

2. **gpt-oss-120b is the quality ceiling for single-GPU** — near o4-mini at 313 tok/s on an 80GB GPU, or ~15-25 tok/s on a 128GB Mac.

3. **Nanbeige4.1-3B is the small model miracle** — 3B params, outperforms 32B models, runs on anything.

4. **128GB Macs are secretly OP** — unified memory lets you run 120B+ MoE models that normally need datacenter hardware. Use MLX, not llama.cpp.

5. **MoE architecture is winning** — models like gpt-oss-120b (5.1B active of 120B total) and Qwen3-Coder-Next (3B active of 80B total) get big-model quality at small-model speeds.

6. **Open source trails proprietary by only ~50 Elo points** now. A year ago it was 150+. The gap is closing fast.

---

*Data sources: [Chatbot Arena](https://arena.ai/leaderboard/text), [Artificial Analysis](https://artificialanalysis.ai/leaderboards/models), [WhatLLM Feb 2026 Rankings](https://whatllm.org/blog/best-open-source-models-february-2026), [HuggingFace Trending](https://huggingface.co/models?sort=trending), [llama.cpp Apple Silicon Benchmarks](https://github.com/ggml-org/llama.cpp/discussions/4167), [Home GPU LLM Leaderboard](https://awesomeagents.ai/leaderboards/home-gpu-llm-leaderboard/), [GGUF Quantization Benchmarks](https://dasroot.net/posts/2026/02/gguf-quantization-quality-speed-consumer-gpus/)*
