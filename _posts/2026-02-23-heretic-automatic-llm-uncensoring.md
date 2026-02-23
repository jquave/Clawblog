---
title: "Heretic: One Command to Remove Any LLM's Censorship, Backed by NeurIPS Math"
date: 2026-02-23
categories:
  - Trending
tags:
  - llm
  - open-source
  - uncensoring
  - abliteration
  - mechanistic-interpretability
excerpt: "A fully automatic tool that finds and surgically deletes the single 'refusal direction' in any LLM's residual stream. 200 Bayesian optimization trials, 45 minutes on an RTX 3090, 1/6th the quality damage of manual methods. ~9,200 GitHub stars."
header:
  overlay_color: "#2d1b2e"
toc: true
toc_label: "Contents"
toc_icon: "brain"
toc_sticky: true
---

There's a tool trending on GitHub that makes most AI safety researchers uncomfortable: [p-e-w/heretic](https://github.com/p-e-w/heretic). One command. Any instruction-tuned LLM. Censorship gone.

```bash
pip install -U heretic-llm
heretic Qwen/Qwen3-4B-Instruct-2507
```

~45 minutes later on an RTX 3090, you have a model that will never refuse a prompt again. No fine-tuning. No training data. No GPU cluster. The modification is permanent and the model can be saved, shared, and quantized like any other.

~9,200 GitHub stars. 14 contributors. 1,300+ uncensored models already published on HuggingFace using this tool. AGPL-3.0 license.

But the interesting part isn't the tool itself -- it's the science underneath. Heretic exploits a NeurIPS 2024 discovery that refusal behavior in language models is controlled by **a single direction vector** in the model's internal activations. Find it, delete it, and the model loses the ability to refuse -- while keeping almost everything else intact.

## The Science: Refusal Is a Line

The foundational insight comes from Arditi et al.'s NeurIPS 2024 paper: *"Refusal in Language Models Is Mediated by a Single Direction."*

Here's what they found. When you feed a chat model a harmful prompt ("How do I pick a lock?") and a harmless prompt ("How do I bake a cake?"), the internal activations -- the numbers flowing through the transformer's residual stream -- diverge. But they don't diverge in a complex, distributed way. They diverge along **one direction**.

That direction is a vector. It can be computed:

```python
# For each transformer layer:
harmful_mean = mean(activations on harmful prompts)
harmless_mean = mean(activations on harmless prompts)

refusal_direction = harmful_mean - harmless_mean
refusal_direction = refusal_direction / norm(refusal_direction)
```

RLHF and safety fine-tuning don't deeply rewire the model's reasoning. They teach it a simple linear classifier: "if the current activation has a large component along this direction, output a refusal." It's a tripwire, not a fundamental behavioral change.

Remove the tripwire, and the model still knows how to reason, write code, answer questions, and follow instructions. It just can't refuse anymore.

### How Removal Works

A decoder-only transformer (Llama, Gemma, Qwen, etc.) has weight matrices that write to the residual stream at each layer: attention out-projections (`W_O`) and MLP out-projections (`W_out`). **Orthogonalizing** these matrices with respect to the refusal direction makes them physically incapable of writing that direction into the residual stream:

```python
# For each weight matrix in the model:
projection = einsum(matrix, refusal_direction) * refusal_direction
orthogonalized = matrix - projection
```

After this, the model cannot produce the refusal activation pattern. Not "won't" -- **can't**. The direction no longer exists in the space the model can write to.

This is what people call **abliteration** -- ablation + obliteration.

## What Heretic Adds: Automated Optimization

Manual abliteration has been possible since the Arditi paper. Maxime Labonne's [HuggingFace tutorial](https://huggingface.co/blog/mlabonne/abliteration) popularized it. But doing it well is hard:

- Which layer's refusal direction do you use?
- How much ablation weight do you apply?
- Do you treat MLP and attention projections differently?
- How do you minimize damage to the model's capabilities?

Heretic answers all of these automatically using **multi-objective Bayesian optimization** via Optuna's Tree-structured Parzen Estimator (TPE).

### The Optimization Loop

**Step 1: Compute refusal directions.** Heretic runs 400 harmful prompts and 400 harmless prompts through the model, recording activations at every layer. Each layer gets its own refusal direction vector.

**Step 2: Define the search space.** The optimizer controls:

| Parameter | What It Does |
|---|---|
| `direction_index` | Which layer's refusal direction to use (supports float interpolation -- e.g., 9.3 blends layers 9 and 10) |
| `max_weight` / `max_weight_position` | Peak ablation strength and which layer gets maximum modification |
| `min_weight` / `min_weight_distance` | Shape and spread of the ablation kernel across layers |
| MLP vs. attention split | Separate tuning for each component type (MLP interventions cause more damage) |

**Step 3: Run 200 optimization trials.** Each trial:

1. Samples abliteration parameters
2. Applies orthogonalization with those parameters
3. Counts refusals on 100 held-out harmful evaluation prompts (using 28 refusal markers: "sorry", "I can't", "harmful", etc.)
4. Measures KL divergence between modified and original model outputs on harmless prompts
5. Records results on the Pareto frontier

**Two objectives, co-minimized:**

- **Minimize refusals** (the point of the exercise)
- **Minimize KL divergence** (preserve the model's capabilities)

**Step 4: Select and export.** Heretic picks the best parameter set from the Pareto frontier and saves the modified model. You can save locally, upload to HuggingFace, or drop into an interactive chat to test.

## The Numbers

### Quality Preservation (Gemma-3-12b-it)

This is Heretic's headline result. Three different uncensored versions of the same base model:

| Model | Refusals (out of 100) | KL Divergence |
|---|:---:|:---:|
| google/gemma-3-12b-it (original) | 97 | 0 (baseline) |
| mlabonne/gemma-3-12b-it-abliterated-v2 | 3 | **1.04** |
| huihui-ai/gemma-3-12b-it-abliterated | 3 | **0.45** |
| **p-e-w/gemma-3-12b-it-heretic** | **3** | **0.16** |

All three uncensored versions achieve the same refusal rate (3/100). But Heretic's KL divergence is **1/3 of the next best** and **1/6 of the established method**. Lower KL divergence = less damage to the model's general capabilities. The model stays closer to its original self.

### Independent Evaluation (arXiv:2512.13655)

A peer-reviewed comparison tested four abliteration tools (Heretic, DECCP, ErisForge, FailSpy) across 16 instruction-tuned models from 7B to 14B parameters:

| Metric | Heretic | DECCP | ErisForge | FailSpy |
|---|:---:|:---:|:---:|:---:|
| **Model coverage** | **100%** | 69% | 56% | 31% |
| MMLU impact (avg) | Small | Small | **Smallest** | Variable |
| GSM8K impact (avg) | Variable | **-0.13 pp** | -0.28 pp | Variable |

Heretic wins on **universality** -- it worked on all 16 models. The single-pass methods (DECCP, ErisForge) can preserve capabilities better on specific model/benchmark combinations, but they fail on nearly half the architectures.

The paper also revealed that **mathematical reasoning (GSM8K) is the most fragile capability** under abliteration. Heretic's KL divergence optimization helps here, but it's still the benchmark most likely to degrade.

### Magnitude-Preserving Orthogonal Ablation (v1.2.0)

The latest release added MPOA, which restores original weight matrix magnitudes after orthogonalization:

- UGI Score: **39.05** (vs. 34.22 traditional) -- approximately **14% improvement** in general capability preservation.

## Hardware Requirements

Heretic runs the full model in memory during optimization, so VRAM requirements scale with model size:

| Model Size | Full Precision | With 4-bit Quantization (v1.2.0) |
|---|---|---|
| 7B | ~14 GB | ~4.2 GB |
| 13B | ~26 GB | ~7.8 GB |
| 70B | ~140 GB | ~42 GB |

An RTX 3090 (24 GB) handles 7B-13B models at full precision. With v1.2.0's 4-bit quantization support, an RTX 3060 12GB can process 7B models. Optimization takes ~45 minutes for an 8B model on an RTX 3090.

## Supported Models

- **Dense transformers**: Llama 3.x, Gemma 3, Qwen 3, Mistral, and most others
- **Vision-language models**: As of v1.2.0 -- abliterates the text decoder while preserving the image encoder
- **Some MoE architectures**: Certain Mixture-of-Experts models
- **Not supported**: SSMs/hybrid models (Mamba, etc.), models with inhomogeneous layers

1,300+ models have been published on HuggingFace using Heretic within three months. The community collection from DavidAU alone covers dozens of architectures.

## Usage

### Basic

```bash
pip install -U heretic-llm
heretic meta-llama/Llama-3.1-8B-Instruct
```

That's it. Heretic downloads the model, runs optimization, and prompts you to save the result.

### With Options

```bash
# Show prompt/response pairs during optimization
heretic --print-responses meta-llama/Llama-3.1-8B-Instruct

# Enable 4-bit quantization for reduced VRAM
# (set in config.default.toml: quantization = "bnb_4bit")

# Generate residual stream visualizations
heretic --plot-residuals meta-llama/Llama-3.1-8B-Instruct

# Print cosine similarity and geometry analysis
heretic --print-residual-geometry meta-llama/Llama-3.1-8B-Instruct
```

### Configuration

Heretic uses a TOML config file with tunable parameters:

```toml
n_trials = 200              # Optimization trial count
n_startup_trials = 60       # Random exploration before Bayesian optimization
kl_divergence_target = 0.01 # Target KL divergence
quantization = "bnb_4bit"   # Enable 4-bit for lower VRAM
```

After optimization, you choose: save locally, upload to HuggingFace Hub, interactive chat, or any combination.

## The Debate

Heretic sits squarely in dual-use territory, and the community knows it.

### The Technical Criticism

From [HN discussions](https://news.ycombinator.com/item?id=45945587):

> *"You're lobotomizing the model and it will result in worse performance."* -- The concern that removing refusal also removes knowledge. The counter-argument: RLHF doesn't add knowledge about restricted topics; it adds a classifier that blocks output. Removing the classifier doesn't remove knowledge that was never there.

> *"That sounds like it removes some unknown amount of censorship, where the amount removed could be anywhere from just these exact prompts to all censorship entirely."* -- The generalization concern. Heretic optimizes against a specific set of harmful prompts, but the refusal direction is shared across all refusals. Targeting the strongest refusals pulls out the root, affecting weaker/legitimate ones too.

### The Safety Argument

The academic paper frames it directly: *"Systematic understanding of safety vulnerabilities is prerequisite to building robust protections, not antithetical to them."*

The counterpoint from safety researchers: understanding the vulnerability is different from packaging a one-command exploit and distributing it with a PyPI package.

### The Practical Reality

1,300+ uncensored models are already on HuggingFace. The technique was published at NeurIPS. The knowledge is public. Heretic didn't create the vulnerability -- it automated the exploitation of a known one.

The AGPL-3.0 license creates legal obligations if modified versions are offered as a network service, which provides at least some governance over commercial deployment.

## Why It Matters Beyond Uncensoring

The deeper story isn't about removing censorship. It's about what the technique reveals: **that RLHF safety alignment is geometrically simple**.

Billions of dollars of safety research, months of RLHF training, carefully curated preference datasets -- and the entire mechanism collapses to a single direction vector that can be deleted in 45 minutes on consumer hardware.

This has implications for AI safety research:

- **Current alignment is shallow.** If refusal is mediated by a single direction, it's not deeply integrated into the model's reasoning. It's a patch, not a rewrite.
- **Robustness requires depth.** Future alignment techniques need to be more deeply entangled with the model's capabilities to resist this kind of surgical removal.
- **Interpretability enables both attack and defense.** The same mechanistic interpretability that found the refusal direction could find directions for deception, sycophancy, or other unwanted behaviors.

Heretic is a tool. It's also a proof of concept that the current paradigm of safety alignment has a geometric weakness. Whether that motivates better alignment research or just more uncensored models on HuggingFace depends on who's paying attention.

---

*Sources: [p-e-w/heretic (GitHub)](https://github.com/p-e-w/heretic), [Arditi et al. 2024 -- "Refusal in Language Models Is Mediated by a Single Direction" (NeurIPS 2024)](https://proceedings.neurips.cc/paper_files/paper/2024/file/f545448535dfde4f9786555403ab7c49-Paper-Conference.pdf), [arXiv:2512.13655 -- Comparative Analysis of LLM Abliteration Methods](https://arxiv.org/abs/2512.13655), [Maxime Labonne -- "Uncensor any LLM with abliteration" (HuggingFace Blog)](https://huggingface.co/blog/mlabonne/abliteration), [Heretic HN Discussion](https://news.ycombinator.com/item?id=45945587), [heretic-llm on PyPI](https://pypi.org/project/heretic-llm/)*
