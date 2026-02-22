---
title: "gh-aw: GitHub's Experiment to Replace YAML CI/CD With Plain English"
date: 2026-02-23
categories:
  - Trending
tags:
  - github
  - ci-cd
  - agentic
  - automation
  - github-actions
excerpt: "GitHub built a system where you write CI/CD automation in Markdown, not YAML, and an AI agent figures out the rest. 7,400 commits, 254 releases, 100+ workflows running in production. Here's how it works."
header:
  overlay_color: "#0d1117"
toc: true
toc_label: "Contents"
toc_icon: "robot"
toc_sticky: true
---

There's a repo at [github/gh-aw](https://github.com/github/gh-aw) with 7,448 commits, 254 releases, and 40+ contributors. It's trending on GitHub this week with 3,500+ stars. It's MIT licensed. And most developers haven't heard of it.

GitHub Agentic Workflows lets you write CI/CD automation in plain English Markdown. An AI agent -- Copilot, Claude, or Codex -- reads your instructions, reasons about your repo, and takes action. The Markdown gets compiled into a hardened GitHub Actions workflow with security guardrails baked in.

Instead of this:

```yaml
name: Issue Triage
on:
  issues:
    types: [opened]
jobs:
  triage:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Analyze issue
        uses: some-ai-action@v1
        with:
          model: gpt-4
          prompt: |
            Analyze this issue and add appropriate labels.
            If it's a bug, add "bug". If it's a feature request,
            add "enhancement". If it's unclear, ask for details.
      - name: Add labels
        uses: actions-ecosystem/action-add-labels@v1
        # ... 40 more lines of YAML
```

You write this:

```markdown
---
on:
  issues:
    types: [opened]
permissions: read-all
safe-outputs:
  add-comment:
  add-label:
---

# Issue Triage

When a new issue is opened:

1. Read the issue title and body
2. Classify it as bug, feature request, question, or unclear
3. Add the appropriate label
4. If the issue is unclear or missing reproduction steps,
   post a polite comment asking for more details
5. If it looks like a duplicate, link to the existing issue
```

Then run `gh aw compile` and you get a locked-down `.lock.yml` that runs in GitHub Actions with sandboxed execution, read-only defaults, and write operations restricted to only what you explicitly allowed.

## How It Actually Works

The system has three layers:

### 1. The Workflow File (Markdown)

A `.md` file in `.github/workflows/` with YAML frontmatter and a natural language body. The frontmatter defines:

- **`on:`** -- What triggers the workflow (issue opened, PR created, cron schedule, slash command)
- **`permissions:`** -- What the agent can read (defaults to read-only)
- **`safe-outputs:`** -- The only write operations allowed (create issue, add comment, create PR, add label, etc.)

The body is plain English instructions. The agent reads them, looks at the repo context (the issue that was opened, the PR that failed, etc.), and acts.

### 2. The Compiler (`gh aw compile`)

This is the critical piece. The compiler transforms your Markdown into a hardened GitHub Actions YAML file (`.lock.yml`) with:

- **SHA-pinned dependencies** -- No supply chain attacks via unpinned actions
- **Network isolation** -- The Agent Workflow Firewall (AWF) controls egress
- **Tool allowlists** -- The agent can only use tools you've explicitly permitted
- **Input sanitization** -- Issue titles and PR bodies are cleaned before the agent sees them
- **Compile-time validation** -- Catches permission mismatches before deployment

You commit both the `.md` and the `.lock.yml`. The Markdown is what humans read and edit. The lock file is what Actions executes.

### 3. The Agent Runtime

When the workflow triggers, a containerized environment spins up inside GitHub Actions. The AI agent (Copilot, Claude, or Codex) receives:

- The compiled instructions from your Markdown
- The context (the issue that was opened, the PR diff, the CI logs, etc.)
- Access to the tools you allowed (read files, search code, run commands)
- Permission to produce only the safe-outputs you specified

The agent reasons about the situation and acts. If it needs to create an issue, it goes through the safe-output pipeline, which sanitizes the content before writing.

## What 100+ Production Workflows Look Like

GitHub isn't just building this -- they're running it on themselves. The [Peli's Agent Factory](https://github.github.com/gh-aw/blog/2026-01-12-welcome-to-pelis-agent-factory/) blog post reveals 100+ agentic workflows operating inside the `github/gh-aw` repo and across internal GitHub repositories.

Here's a sampling by category:

### Issue & PR Management

| Workflow | What It Does |
|---|---|
| **Issue Triage** | Auto-classifies and labels incoming issues |
| **Contribution Guidelines Checker** | Reviews PRs for compliance with contributing rules |
| **Grumpy Reviewer** | On-demand opinionated code review by a "grumpy senior dev" |
| **AI Moderator** | Detects and moderates spam, link spam, and AI-generated content |

### CI/CD Intelligence

| Workflow | What It Does |
|---|---|
| **CI Doctor** | Monitors CI runs, investigates failures automatically |
| **CI Coach** | Analyzes CI workflows for speed and cost optimization |
| **PR Fix** | Analyzes failing CI checks and implements fixes |

### Code Quality

| Workflow | What It Does |
|---|---|
| **Code Simplifier** | Simplifies recently modified code for clarity |
| **Duplicate Code Detector** | Finds duplicate patterns, suggests refactoring |
| **Daily Test Coverage Improver** | Adds tests to under-tested areas |
| **Daily Performance Improver** | Benchmarks and optimizes hot paths |
| **Daily Malicious Code Scan** | Scans recent changes for suspicious patterns |

### Documentation

| Workflow | What It Does |
|---|---|
| **Daily Doc Updater** | Updates docs based on recent code changes and merged PRs |
| **Glossary Maintainer** | Maintains project glossary from codebase changes |
| **Link Checker** | Finds and fixes broken links daily |
| **Documentation Unbloat** | Reduces verbose docs while maintaining clarity |

### Meta-Workflows

| Workflow | What It Does |
|---|---|
| **Q - Workflow Optimizer** | Analyzes and optimizes other agentic workflows |
| **Repo Assist** | Daily assistant: triages issues, fixes bugs, creates activity summaries |
| **Daily Backlog Burner** | Systematically works through and reduces issue backlog |

The meta-workflow concept is particularly interesting -- agents that monitor and improve other agents.

## The Security Model

This is where gh-aw distinguishes itself from "just run Claude in a GitHub Action." The security approach is paranoid by design:

**Read-only by default.** The agent can read your repo, issues, PRs, and CI logs. It cannot write anything unless you explicitly allow it.

**Safe-outputs are the only write path.** You whitelist specific operations:

```yaml
safe-outputs:
  create-issue:        # Can create new issues
  add-comment:         # Can comment on issues/PRs
  create-pull-request: # Can open PRs
  add-label:           # Can add labels
```

Everything else is denied. The agent can't push to branches, merge PRs, delete issues, or modify repository settings.

**Sandboxed execution.** The agent runs in an isolated container. The [Agent Workflow Firewall (AWF)](https://github.com/github/gh-aw) controls network egress -- the agent can't phone home to arbitrary servers.

**MCP Gateway.** Model Context Protocol calls are routed through a centralized gateway for access management and auditing.

**Human approval gates.** Critical operations can require human approval before the agent proceeds.

**Compile-time validation.** The `gh aw compile` step catches permission mismatches and unsafe configurations before they ever reach production.

This layered approach is the answer to the obvious question: "Why would I let an AI agent loose on my repo?" You're not. You're letting it operate within a very specific sandbox with very specific capabilities.

## The Skeptics Have Points

The Hacker News reception wasn't all positive. Some valid concerns:

**"The YAML + Markdown hybrid is comically awful."** Fair criticism. The frontmatter is still YAML. You haven't eliminated YAML -- you've just moved most of the logic into English. Whether that's better depends on whether your automation is primarily conditional logic (YAML is fine) or reasoning-heavy (Markdown is better).

**"This adds AI overhead to every CI run."** True. Every workflow invocation calls an LLM. That's slower and more expensive than a deterministic bash script. The value proposition only works for tasks that genuinely benefit from reasoning -- triage, code review, documentation -- not for `npm test && npm build`.

**"Automated refactoring PRs will spam the repo."** A legitimate concern. The daily workflows (Code Simplifier, Doc Updater, Test Coverage Improver) will generate PRs that someone has to review. Without careful tuning, this becomes noise rather than signal. GitHub's own experience with 100+ workflows probably taught them a lot about this.

**"This is a research demo, not a product."** GitHub labels it a "research demonstrator" from GitHub Next. But 254 releases and 7,448 commits suggest it's far more mature than typical research projects. The question is whether it graduates to a first-party GitHub feature.

## When to Use It (and When Not To)

**Use gh-aw when:**

- The task requires **reasoning about context** -- understanding what an issue is asking, whether a PR follows conventions, why CI is failing
- You'd otherwise write a fragile script that handles 5 cases and breaks on the 6th
- The output is low-risk (comments, labels, new issues) rather than high-risk (direct code changes, deployments)
- You want automation that **adapts** rather than follows rigid rules

**Don't use gh-aw when:**

- The task is deterministic -- `run tests`, `build docker image`, `deploy to staging`
- Speed matters -- LLM inference adds seconds to minutes of latency
- Cost matters at scale -- every invocation calls an LLM API
- You need guaranteed, reproducible behavior -- agents are probabilistic

The sweet spot: **triage, analysis, and low-stakes maintenance**. Let the agent classify issues, review PRs for obvious problems, keep docs updated, and flag CI failures. Keep your actual build/test/deploy pipeline in regular Actions YAML.

## Getting Started

```bash
# Install the CLI extension
gh extension install github/gh-aw

# Initialize in your repo
gh aw init

# Create a workflow (interactive)
gh aw create

# Or manually create .github/workflows/my-workflow.md
# Then compile it
gh aw compile

# Test locally
gh aw run my-workflow
```

The [sample pack](https://github.com/githubnext/agentics) has 25+ ready-to-use workflow templates you can drop into any repo.

## The Bigger Picture

gh-aw represents a specific bet: **that natural language is a better interface than YAML for describing intent-driven automation**.

Traditional CI/CD is imperative -- you specify exact steps. Agentic workflows are declarative -- you specify desired outcomes. The agent figures out the steps.

This mirrors the shift happening across software engineering. We went from writing assembly (exact instructions) to writing Python (high-level intent). Now we're going from writing YAML (exact CI steps) to writing English (high-level automation goals).

Whether this specific implementation succeeds is less important than the direction it points. The gap between "describe what you want" and "get a working automation pipeline" is collapsing. gh-aw is one of the most serious attempts to close it, backed by the company that owns the CI/CD platform it runs on.

7,448 commits. 254 releases. 100+ workflows in production. Whatever this is, it's not a toy.

---

*Sources: [github/gh-aw (GitHub)](https://github.com/github/gh-aw), [Official Documentation](https://github.github.com/gh-aw/), [GitHub Next: Agentic Workflows](https://githubnext.com/projects/agentic-workflows/), [Sample Workflows Pack](https://github.com/githubnext/agentics), [Peli's Agent Factory](https://github.github.com/gh-aw/blog/2026-01-12-welcome-to-pelis-agent-factory/), [InfoQ: GitHub Agentic Workflows](https://www.infoq.com/news/2026/02/github-agentic-workflows/), [YUV.AI Analysis](https://yuv.ai/blog/gh-aw)*
