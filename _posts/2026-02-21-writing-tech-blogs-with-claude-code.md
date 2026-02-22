---
title: "Writing Tech Blogs With Claude Code"
date: 2026-02-21
categories:
  - Meta
tags:
  - claude-code
  - ai-writing
  - workflow
excerpt: "What happens when you point an AI coding assistant at the open web and tell it to write a tech blog? This is that experiment."
header:
  overlay_color: "#1a1a2e"
---

This blog is an experiment.

The premise is simple: what if you used [Claude Code](https://claude.ai/claude-code) -- Anthropic's CLI agent for software engineering -- not just to write code, but to research and write technical blog posts?

Not the kind of AI-generated slop that litters the internet. Actual research. Going to benchmark sites, pulling real numbers from leaderboards, cross-referencing hardware specs, and assembling it into something a human developer would find useful.

## How It Works

The workflow behind every post on this blog:

1. **A human asks a question.** Something like *"What's the fastest open source model I can run on my MacBook Pro?"*

2. **Claude Code researches it live.** It hits HuggingFace trending, Chatbot Arena, Artificial Analysis, GitHub, ModelScope, hardware benchmark threads, quantization guides -- whatever's relevant. Not from training data. From the actual web, right now.

3. **The human pushes back.** *"Those are just summaries from articles. Look at the actual benchmark leaderboards."* So it does. It pulls real Elo scores from Arena, real tok/s measurements from hardware tests, real download counts from HuggingFace.

4. **Claude Code writes the post.** Markdown, with tables, charts, sourced data. Then it builds the blog itself -- picks a Jekyll theme, sets up GitHub Pages, writes the config.

5. **The human publishes it.** One `git push` and it's live.

The entire chain from question to published blog post happens in a single terminal session.

## Why This Is Interesting

Most AI-generated content is bad because it's unsupervised pattern completion. The model hallucinates numbers, invents benchmarks, and writes in that unmistakable "AI slop" voice.

This is different because:

- **Every claim is sourced from a live web fetch.** If Claude Code says GLM-4.7-Flash gets 120-220 tok/s on an RTX 4090, it's because it pulled that from a hardware benchmark page, not because it pattern-matched from training data.

- **A human is in the loop questioning the work.** The second post on this blog exists because the human said *"why don't you look at actual benchmarks"* -- and the research got significantly better as a result.

- **The tool builds the whole stack.** Claude Code didn't just write the content. It researched Jekyll themes, wrote the `_config.yml`, set up the GitHub Pages deployment, and created this post you're reading. One tool, end to end.

## What This Blog Is Not

This is not an attempt to replace human tech writers. It's an exploration of what happens when you combine:

- A human who knows what questions to ask
- An AI that can research the web and write structured content
- A feedback loop where the human rejects weak work

The human editorial judgment -- knowing when the research is shallow, when numbers smell wrong, when the framing is off -- is the part that makes this work. Claude Code is the research engine and the publishing pipeline. The human is the editor.

## The Stack

For the curious, here's what's powering this blog:

- **Content generation:** Claude Code (Claude Opus 4.6)
- **Research:** Live web search, web fetch, cross-referencing multiple benchmark sources
- **Static site generator:** Jekyll with the Minimal Mistakes theme (dark skin)
- **Hosting:** GitHub Pages
- **Total cost of infrastructure:** $0

Every post on this blog was researched, written, and deployed from a single terminal session. The source is public at [github.com/jquave/Clawblog](https://github.com/jquave/Clawblog).

---

*Next up: [The Fastest SOTA Open Source Models You Can Run Locally](/Clawblog/fastest-local-llms-feb-2026/) -- a deep dive into what's actually worth running on your hardware right now, backed by real benchmark data.*
