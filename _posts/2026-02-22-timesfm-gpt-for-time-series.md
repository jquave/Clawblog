---
title: "TimesFM: Google's 'GPT for Time Series' Is Quietly Dominating Forecasting"
date: 2026-02-22
categories:
  - Trending
tags:
  - time-series
  - google-research
  - foundation-models
  - forecasting
  - open-source
excerpt: "A 200M parameter model that predicts the future of any time series -- stock prices, server load, weather, sales -- with zero training. It's trending on GitHub this week and most people haven't heard of it."
header:
  overlay_color: "#1b2838"
toc: true
toc_label: "Contents"
toc_icon: "chart-line"
toc_sticky: true
---

There's a repo trending on GitHub this week that most ML engineers have never heard of. It's not an LLM. It's not a coding agent. It's not a chatbot.

It's [google-research/timesfm](https://github.com/google-research/timesfm) -- a **decoder-only transformer trained on 100 billion real-world time points** that does zero-shot time series forecasting. You feed it historical data -- stock prices, server metrics, sensor readings, sales figures, anything with timestamps -- and it predicts the future. No training. No fine-tuning. No feature engineering.

It just works. And it's currently **#1 on the GIFT-Eval zero-shot forecasting benchmark**, beating Amazon Chronos, Lag-Llama, and every other time series foundation model.

+1,259 GitHub stars this week. 9,400 total. Apache 2.0 license.

## What Is a Time Series Foundation Model?

The same idea behind GPT, but for numbers over time instead of words in sequence.

Traditional time series forecasting requires you to:

1. Collect historical data
2. Choose a model (ARIMA, Prophet, N-BEATS, etc.)
3. Feature engineer (seasonality, trend decomposition, exogenous variables)
4. Train on your specific dataset
5. Tune hyperparameters
6. Hope it generalizes

TimesFM skips steps 2-6. It was pretrained on 100 billion time points from diverse real-world sources (Google Trends, Wikimedia pageviews, financial data, synthetic augmentations) and learned general temporal patterns -- seasonality, trends, changepoints, noise structures -- that transfer to any new time series.

You give it a history. It gives you a forecast. That's it.

```python
import torch
from timesfm import TimesFm

model = TimesFm.load("google/timesfm-2.5-200m-pytorch")

# Your data -- any time series
history = [120, 135, 142, 128, 155, 163, 149, 171, 180, 168]

# Predict the next 5 points
forecast = model.forecast(history, horizon=5)
```

## Why This Matters

Time series forecasting is everywhere. Demand planning, capacity provisioning, energy grid management, financial modeling, anomaly detection, healthcare monitoring. It's one of the most important practical ML problems, and historically one of the most painful.

The pain comes from the fact that every dataset is different. A model trained on retail sales data doesn't transfer to server CPU metrics. You need domain expertise, custom pipelines, and ongoing maintenance for every use case.

TimesFM breaks this pattern the same way GPT broke the pattern for NLP: **one pretrained model, many tasks, zero fine-tuning**.

## The Numbers (v2.5)

TimesFM has been through three major versions. The current one, v2.5, is the one that's turning heads:

| Metric | TimesFM 1.0 | TimesFM 2.0 | TimesFM 2.5 |
|---|:---:|:---:|:---:|
| Parameters | 200M | 500M | **200M** |
| Context length | 512 | 2K | **16K** |
| GIFT-Eval rank (zero-shot) | - | Competitive | **#1** |
| Probabilistic output | No | No | **Yes (quantile head)** |
| Covariate support | No | Yes | **Yes (XReg)** |

Read that table again. They **cut the parameter count by 60%** (500M to 200M), **increased context 8x** (2K to 16K), and **moved from competitive to #1 on the benchmark**. Smaller, longer context, better accuracy. That's rare.

### Benchmark Results

On the [GIFT-Eval benchmark](https://github.com/SalesforceAIResearch/gift-eval) (the standard for zero-shot time series forecasting):

| Model | MASE (Point Accuracy) | CRPS (Probabilistic) | Parameters |
|---|:---:|:---:|:---:|
| **TimesFM 2.5** | **#1** | **#1** | 200M |
| Amazon Chronos | Competitive | Competitive | 710M |
| Lag-Llama | Mid-tier | Mid-tier | 52M |
| Moirai | Competitive | Competitive | 311M |

TimesFM achieves the best accuracy with fewer parameters than its closest competitor (Chronos at 710M is 3.5x larger).

## What Can You Actually Use It For?

### Demand Forecasting

```python
# Predict next 30 days of product sales
sales_history = load_daily_sales("product_123", days=365)
forecast = model.forecast(sales_history, horizon=30, freq="D")
```

### Server Capacity Planning

```python
# Predict CPU usage for the next 24 hours
cpu_metrics = get_prometheus_data("cpu_usage_percent", hours=168)
forecast = model.forecast(cpu_metrics, horizon=24, freq="H")
```

### Financial Modeling

```python
# With exogenous variables (interest rates, market indices)
prices = get_stock_prices("AAPL", days=500)
covariates = get_macro_indicators(days=500)  # Fed rate, VIX, etc.
forecast = model.forecast(prices, horizon=30, xreg=covariates)
```

### Anomaly Detection

```python
# Compare actual vs. predicted -- large deviations = anomalies
predicted = model.forecast(historical_data, horizon=1)
actual = get_latest_reading()
if abs(actual - predicted) > threshold:
    alert("Anomaly detected")
```

### Probabilistic Forecasting

This is the v2.5 killer feature. Instead of just a point prediction ("sales will be 150 units"), you get prediction intervals:

```python
# Returns quantiles: 10th, 20th, 30th, ..., 90th percentile
quantile_forecast = model.forecast(
    history, horizon=30,
    quantiles=[0.1, 0.25, 0.5, 0.75, 0.9]
)
# quantile_forecast.q50 = median prediction
# quantile_forecast.q10 = pessimistic scenario
# quantile_forecast.q90 = optimistic scenario
```

This is critical for real-world decision-making. "Sales will be between 120 and 180 units with 80% confidence" is far more useful than "sales will be 150 units."

## How It Compares

| Feature | TimesFM 2.5 | Amazon Chronos | Lag-Llama | Prophet | ARIMA |
|---|:---:|:---:|:---:|:---:|:---:|
| Zero-shot | Yes | Yes | Yes | No | No |
| Parameters | 200M | 710M | 52M | N/A | N/A |
| Context length | 16K | 512 | 32 | Unlimited | Limited |
| Quantile output | Native | Via sampling | Via sampling | Via sampling | No |
| Covariates (XReg) | Yes | No | No | Yes | Yes |
| BigQuery integration | Yes | No (SageMaker) | No | No | No |
| Training required | No | No | No | Per-dataset | Per-dataset |
| License | Apache 2.0 | Apache 2.0 | Apache 2.0 | MIT | Varies |

The standout advantages:

- **16K context** -- TimesFM can look at 16,000 historical data points. Chronos maxes out at 512. For weekly data, that's 300+ years of history vs. ~10 years.
- **Native covariates** -- You can feed in external variables (weather, holidays, economic indicators) that affect your forecast. Chronos and Lag-Llama can't do this.
- **Native quantile head** -- Purpose-built 30M parameter head for probabilistic output, versus the sample-then-aggregate approach others use.
- **BigQuery integration** -- If your data is in BigQuery, you can forecast directly with SQL. No Python pipeline needed.

## Running It Locally

TimesFM 2.5 is 200M parameters -- tiny by LLM standards. It runs comfortably on a CPU, and flies on any GPU.

### Installation

```bash
pip install timesfm[torch]
```

### Basic Usage

```python
import timesfm

# Load the model (downloads ~400MB on first run)
model = timesfm.TimesFm(
    hparams=timesfm.TimesFmHparams(
        per_core_batch_size=32,
        horizon_len=128,
        backend="gpu",  # or "cpu"
    ),
    checkpoint=timesfm.TimesFmCheckpoint(
        huggingface_repo_id="google/timesfm-2.5-200m-pytorch"
    ),
)

# Forecast
import numpy as np
history = np.array([your_time_series_data])
point_forecast, quantile_forecast = model.forecast(
    [history],
    freq=[0],  # 0 = auto-detect frequency
)
```

### Hardware Requirements

| Setup | Performance | Notes |
|---|---|---|
| CPU (any modern) | Usable | Fine for batch processing |
| Single GPU (any) | Fast | 200M params fits in <1GB VRAM |
| Apple Silicon (MLX) | Fast | JAX backend works well |
| Google BigQuery | Managed | `CREATE MODEL ... OPTIONS(model_type='TIMESFM')` |

No quantization needed. No multi-GPU setup. No MLX vs. llama.cpp debate. It's 200M params. It just runs.

## The Bigger Picture

TimesFM represents something important: **foundation models moving beyond text and images into structured data**.

We've had GPT for text (2018-present), DALL-E/Stable Diffusion for images (2022-present), and now TimesFM for time series. The pattern is the same each time:

1. Pretrain a transformer on massive diverse data
2. The model learns transferable patterns
3. Zero-shot performance on new tasks matches or beats task-specific models
4. The entire workflow simplifies dramatically

Time series was one of the last holdouts because the data is inherently heterogeneous -- a temperature reading has nothing in common with a stock price, structurally. TimesFM's contribution is showing that a transformer can learn temporal patterns abstract enough to transfer across domains.

The implications are significant for any team running forecasting pipelines. Instead of maintaining separate models for each metric, each product, each region -- you run one model. Instead of retraining when patterns shift -- the foundation model adapts via its context window. Instead of hiring a time series specialist -- your data engineer writes a few lines of Python.

## Why It's Trending This Week

TimesFM 2.5 shipped in September 2025, but the GitHub trending spike is happening now -- February 2026. The likely reason: practical adoption is catching up to the research.

The model has been integrated into [Google BigQuery](https://cloud.google.com/bigquery/docs/reference/standard-sql/bigqueryml-syntax-create-time-series) as a native model type. Enterprise teams are discovering they can replace entire forecasting pipelines with a single SQL statement. Word is spreading.

19 contributors. Apache 2.0. Available on [HuggingFace](https://huggingface.co/google/timesfm-2.5-200m-pytorch) and [PyPI](https://pypi.org/project/timesfm/). No API key needed. No cloud dependency. Just `pip install` and forecast.

---

*Sources: [google-research/timesfm (GitHub)](https://github.com/google-research/timesfm), [Google Research Blog: Decoder-only Foundation Model for Time-Series Forecasting](https://research.google/blog/a-decoder-only-foundation-model-for-time-series-forecasting/), [TimesFM 2.5 on HuggingFace](https://huggingface.co/google/timesfm-2.5-200m-pytorch), [GIFT-Eval Benchmark](https://github.com/SalesforceAIResearch/gift-eval), [MarkTechPost: TimesFM-2.5 Analysis](https://www.marktechpost.com/2025/09/16/google-ai-ships-timesfm-2-5-smaller-longer-context-foundation-model-that-now-leads-gift-eval-zero-shot-forecasting/)*
