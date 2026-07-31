---
layout: post
title: Build Your Own Claude Code — Running Entirely on Your Own Machine
categories: ai, local-llm, coding-agents, ollama, agent-skills, mcp
excerpt: Models, harnesses, skills, and agents the four layers of a local coding agent in 2026
---

If you have ever looked at Claude Code and thought *"I want this, but with the model running on my own hardware"* — that is a much more achievable goal in mid-2026 than it was a year ago. The pieces are all open now, and more importantly, they are cleanly separated into layers you can swap independently.

This post walks through those layers, gives concrete model recommendations by hardware tier, and lays out a staged implementation path. It also covers the part most guides skip: what you actually have to tune when the model is a 30B running on your laptop instead of a frontier model behind an API.

---

## The four layers

The single most useful mental shift is to stop thinking of "an AI coding tool" as one thing. It is four things stacked on top of each other.

### 1. The model

The brain. This is what runs in Ollama, llama.cpp, or vLLM. Its entire job is narrow: take text in, produce text out — including tool calls emitted as structured JSON.

That is all it does. It does not read your files. It does not run your tests. It asks for those things to happen.

### 2. The harness

The body. This is the program that runs the loop:

> send prompt → model requests a tool call → harness executes it (read file, run bash, apply edit) → result goes back into context → repeat until done

Claude Code *is* a harness. Cursor's agent mode is a harness. So are OpenCode, Aider, and Cline. The model is a pluggable component inside it.

Here is the part people underestimate: **the harness determines more of the day-to-day experience than the model does.** Context compaction strategy, how diffs get applied, permission prompts, how gracefully it recovers from malformed JSON — all harness concerns. A great model in a bad harness feels worse than a decent model in a good one.

### 3. Skills

On-demand procedural knowledge. A skill is a folder containing a `SKILL.md` file — YAML frontmatter with a `name` and `description`, followed by markdown instructions.

Anthropic released the format as an open standard in December 2025, and the spec now lives at agentskills.io under the stewardship of the Agentic AI Foundation. Adoption was fast: the same skill folder works across Claude Code, Codex CLI, Gemini CLI, Cursor, GitHub Copilot, and dozens of other hosts without modification.

The design principle that makes skills different from "a big system prompt" is **progressive disclosure**, which works in three stages:

| Stage | What loads | Cost |
|---|---|---|
| Startup | Name + description only | ~30–100 tokens per skill |
| Trigger | Full `SKILL.md` body | Recommended under 5,000 tokens |
| Execution | Files in `references/`, `scripts/`, `assets/` | Only what is actually opened |

The practical guidance from the spec: keep `SKILL.md` under 500 lines, number your steps, and push anything needed less than ~20% of the time into a reference file.

### 4. Agents and subagents

A subagent is a separate harness instance with its own system prompt, its own tool set, and — critically — **its own context window**.

The point is isolation, not parallelism. A `code-reviewer` subagent can read fifty files, form an opinion, and return three paragraphs. The fifty files never touch your main conversation. On a local model with a realistically usable context budget, this matters far more than it does with a frontier model.

### The supporting layer: MCP

Model Context Protocol is the standard that lets a harness talk to external tools — databases, issue trackers, code intelligence servers. If a skill is *knowledge*, an MCP server is *capability*.

---

## Where your existing tools fit

If you already use tools from the current ecosystem, they slot into these layers cleanly — and because the skill format is an open standard, **they are portable to whatever harness you end up with**.

- **Ponytail** — a pure skill (layer 3). It injects a "lazy senior developer" decision ladder that forces the agent to stop before writing code and ask whether the feature needs to exist at all, whether the standard library already handles it, whether a native platform feature covers it, or whether an installed dependency does the job.
- **Graphify** — a skill *and* an MCP server (layers 3 and 5). It parses your repository on-device with tree-sitter, writes a knowledge graph to local files under `graphify-out/`, and exposes query tools over MCP so the agent traces structure instead of grepping. No embeddings, no vector database, no cloud call.
- **Domain skills** (Spring Boot, Kubernetes, whatever your stack is) — layer 3, and the highest-leverage thing you own. These encode *your* conventions, and a local model needs that guidance more than a frontier model does.

---

## Model recommendations by hardware tier

This is entirely a function of your memory budget. The general rule of thumb is roughly 1 GB per billion parameters at full precision, though mixture-of-experts models break that math in your favor.

| RAM / VRAM | Model | Notes |
|---|---|---|
| 16 GB | `gpt-oss:20b`, or a dense 14B-class coder | Weak agentically. Fine for single-file edits and completion. |
| 24–32 GB | **`qwen3-coder:30b`** | 30B total / 3.3B active MoE, ~19 GB at Q4_K_M, 256K context. The current sweet spot. |
| 24 GB, want a benchmarked agent | `devstral:24b` | 46.8% on SWE-Bench Verified, Apache 2.0, ~14 GB. |
| 48–64 GB+ | **Qwen3-Coder-Next** | 80B total / 3B active per token, 256K native context. Needs roughly 46 GB of RAM, VRAM, or unified memory at 4-bit. |

**If you have a 64 GB Mac, an RTX 5090, or a multi-GPU box, Qwen3-Coder-Next is the pick.** Alibaba trained it specifically for agentic work — large-scale executable task synthesis, real environment interaction, and reinforcement learning — which produced exactly the behaviors a Claude Code–style harness depends on: long-horizon reasoning, reliable tool use, and recovery after a command fails.

### An honest note on benchmarks

Published SWE-bench numbers for Qwen3-Coder-Next are wildly inconsistent. Across sources I found 44.3% (on SWE-Bench Pro), 58.7%, and 70.6% (both claiming SWE-Bench Verified) — most of it in SEO-driven content with no methodology disclosed.

Treat the model as "roughly in the neighborhood of a previous-generation frontier model" and stop there. For routine Spring Boot and Kubernetes work, the gap to a hosted frontier model is rarely visible. For gnarly multi-file refactors, it still is.

Models like GLM-5.2 and Kimi K3 score higher on open-weight leaderboards, but they need datacenter-scale hardware. They are downloadable, not local.

---

## Pick a harness — don't write one yet

OpenCode (~165k stars, MIT) has become the de facto open-source Claude Code equivalent: provider-agnostic, first-class local model support, and a polished terminal UI. Other serious options include Crush, Cline, Aider, Goose, OpenHands, and Pi.

My advice: **run OpenCode first and read its internals while you use it.** If you still want to build your own harness afterward, you will have a working reference instead of a blank file.

---

## Implementation, in five stages

### Stage 1 — Model and harness (one afternoon)

```bash
ollama pull qwen3-coder:30b        # or qwen3-coder-next if you have the RAM
ollama serve                       # OpenAI-compatible API at localhost:11434/v1

npm install -g opencode-ai
opencode                           # point the provider config at Ollama
```

### Stage 2 — Install your skills

```bash
uv tool install graphifyy
graphify opencode install          # registers the /graphify skill

# Ponytail: plugin install where supported, or copy the rules file
pi install git:github.com/DietrichGebert/ponytail
```

For your own domain skills, drop the folder into your harness's skill directory — `~/.config/opencode/skills/` globally, or `.opencode/skills/` per project. The only hard requirement is a `SKILL.md` with `name` and `description` in the frontmatter.

### Stage 3 — Index the codebase

```bash
graphify .     # writes graph.json, GRAPH_REPORT.md, graph.html to graphify-out/
```

This is the highest-impact step for a local model, and here is why: the 256K context window is real, but **a small model degrades long before that window fills**. Attention quality falls off a cliff well ahead of the token limit. A structural graph query costs a few hundred tokens and returns a precise answer. Repeated grepping costs tens of thousands and returns noise.

### Stage 4 — Tune for a local model

This is the stage almost every guide skips, and it is where most local setups fail.

Local models are far more fragile at tool calling than frontier models. Concretely:

- **Cut your tool count.** Stay under roughly a dozen active tools. A model with 3B active parameters starts mis-selecting once the list gets long.
- **Cut your active skill count.** Skills with similar-sounding descriptions cause routing misses. The description is the entire interface the model uses to decide whether to activate a skill — treat it as an API surface, not a comment.
- **Use the recommended sampling parameters.** For Qwen3-Coder-Next: `temperature=1.0`, `top_p=0.95`, `top_k=40`.
- **Keep llama.cpp and Ollama current.** Tool-call parsing for these models has been through several rounds of fixes. Running a stale build produces mysterious looping and malformed calls that look like model problems but aren't.
- **Lower your context window to 32K–64K if memory is tight.** A short context full of signal beats a 256K context that is half garbage. This is a feature, not a compromise.

### Stage 5 — Write your own harness (optional)

The minimal loop is about 300 lines: maintain a `messages[]` array, POST to `/v1/chat/completions` with a `tools[]` schema, parse `tool_calls` from the response, execute them, append the results, repeat.

The loop is not the hard part. The hard parts are:

- **Permissions and sandboxing** — what can the agent run without asking?
- **Context compaction** — what do you summarize and what do you drop when the window fills?
- **Diff application** — applying an edit that fails cleanly rather than corrupting a file.
- **Malformed output recovery** — local models emit broken JSON more often than you would like. Your retry strategy is load-bearing.

Solve those four and you have something genuinely usable. Skip them and you have a demo.

---

## Why bother

Three reasons, in descending order of how often they actually matter:

1. **Privacy.** Proprietary code, client work, anything under NDA — a local model is the only option that keeps the code on your machine.
2. **Cost.** No per-token bill, regardless of how much you generate or how many loops the agent burns.
3. **Offline and unmetered.** No rate limits, no outages, no dependency on someone else's capacity planning.

The honest tradeoff is capability. A local setup will not match a frontier model on the hardest tasks. But the fraction of daily work where the difference is visible has shrunk dramatically over the past year — and the layered architecture means when a better open-weight model ships next quarter, you change one line of config.

That portability is the real win. Your skills, your graph, your harness configuration, your conventions — none of it is locked to a vendor.
