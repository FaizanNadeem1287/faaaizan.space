---
title: "ai-memory: A Complete Guide to Giving AI Coding Agents Persistent, Cross-Tool Memory"
description: "How akitaonrails/ai-memory solves the two biggest context problems in AI-assisted coding: losing everything between sessions, and losing everything when you switch from Claude Code to Codex mid-task. A deep dive into its compile-not-retrieve architecture."
date: 2026-08-22
author: "Faizan Nadeem"
tags: ["AI Engineering", "Backend Development", "Developer Tools", "Rust"]
image: "images/blogs/ai-memory-coding-agents.jpg"
imageAlt: "Open notebook with pen and pencils on a wooden desk"
imageCredit: "Photo by Clay Banks on Unsplash"
imageCreditURL: "https://unsplash.com/@claybanks"
css: "/styles/blog.css"
---

## Hook

You quit Claude Code mid-task after four hours of back-and-forth — architecture decisions made, three approaches tried and abandoned, one open question unresolved. An hour later you open Codex in the same directory to try something it's better at. Codex knows none of it. You re-explain the architecture, the failed approaches, and the open question, because that context lived in one tool's transcript and nowhere else. Multiply that by however many agent CLIs you actually use in a given week — and in 2026, with Claude Code, Codex, Cursor, Antigravity, Grok Build, Kimi Code, and a dozen others all viable at once, that number is rarely one.

The common mistake in how "AI agent memory" gets solved is reaching straight for a vector database and calling it done. `ai-memory`, an open-source Rust project from Fabio Akita, takes a deliberately different position: compile a coherent summary at the end of a session instead of retrieving fragments from raw logs at the start of the next one, store it as plain markdown in a git repo instead of embeddings in a vector store, and treat "which agent vendor you're using right now" as irrelevant to whether your memory follows you. This post walks through how that's actually built.

**You'll learn:**

- Why "compile, don't retrieve" is a fundamentally different approach to agent memory than RAG-based tools
- How cross-agent handoffs let you quit Claude Code and resume the same work in Codex, Cursor, or Gemini CLI
- How the hybrid retrieval system combines full-text search, entity matching, and graph neighbors — with vectors as optional, not required
- Why retrieved memory is explicitly treated as untrusted evidence, never as instructions
- How self-hosting a single Rust binary with git-backed markdown compares operationally to a managed vector-DB service
- Where this tool's real limitations are, and what it deliberately doesn't try to do

**Table of Contents**

1. [The Basics](#the-basics)
2. [The Complete Architecture](#the-complete-architecture)
3. [Core Layers Explained](#core-layers-explained)
4. [End-to-End Walkthrough](#end-to-end-walkthrough)
5. [Special Cases](#special-cases)
6. [Scaling & Production Challenges](#scaling--production-challenges)
7. [Code Examples](#code-examples)
8. [Common Pitfalls](#common-pitfalls)
9. [Production Best Practices](#production-best-practices)

## The Basics

### What ai-memory Actually Covers

ai-memory solves two related but distinct problems. The first is the one most memory tools target: an agent session ends, and everything discussed — decisions made, approaches tried and rejected, open questions — disappears unless a human manually writes it down somewhere. The second is the one almost nothing else targets: even if a single tool remembers its own history, that memory doesn't travel when you switch to a *different* agent vendor. A session in Claude Code and a session in Codex are, by default, two islands with no bridge between them.

ai-memory's answer to both is the same underlying mechanism. Lifecycle hooks capture sanitized observations — prompts, tool calls, session boundaries — as a session runs. At session end, those observations get compiled into a coherent markdown page, not just archived as raw log. The next agent that opens in that project, regardless of which vendor it is, receives a bounded "where you left off" handoff before its first prompt.

### Why This Is a High-Value Problem to Solve Well

- **The 2026 agent landscape genuinely is this fragmented.** Developers now routinely keep Claude Code, Codex, and at least one other CLI installed and reach for whichever fits the task — cloud sandboxing here, terminal-native depth there. A memory system tied to one vendor solves an increasingly narrow slice of a developer's actual workflow.
- **Context loss has a real, recurring cost.** Re-explaining architecture and failed approaches isn't just annoying — it burns tokens, burns time, and risks the next agent repeating an approach that already failed for reasons it never learned about.
- **Most competing tools solve this with a vector database**, which means embedding infrastructure, a store to keep available, and a retrieval step that can miss context a human would consider obvious. Betting on "compile a summary, don't retrieve fragments" is a genuinely different bet, and worth understanding on its own terms.

## The Complete Architecture

```
Agent CLI (Claude Code, Codex, Cursor, ...)
        │  lifecycle hooks fire-and-forget
        ▼
ai-memory server (single Rust binary)
        │
        ├── wiki/   ── markdown source of truth, git-versioned
        ├── raw/    ── sanitized transcript segments (managed workstreams only)
        ├── db/     ── SQLite: FTS5 index, entities, embeddings
        └── logs/
        │
        ▼
Session end ──▶ compile observations into a markdown page ──▶ typed handoff (pending)
        │
        ▼
Next agent's SessionStart ──▶ server finds pending handoff ──▶ injects "where you left off"
```

The guiding principle: one server, one data directory, and the CLI itself is a thin HTTP client — `ai-memory status`, `bootstrap`, `search`, and the rest all talk to the running server rather than touching SQLite or the wiki files directly. Markdown is the actual source of truth; SQLite is a rebuildable index over it, which matters more than it sounds like it should once you consider what that means for backup and recovery.

## Core Layers Explained

### 1. Zero-Friction Lifecycle Capture

**What it is:** Hooks fire-and-forget bounded, sanitized observations for prompts, tool calls, and session boundaries as an agent works — without blocking the agent's own execution.

**Why it matters:** Capture that requires the developer to remember to do anything defeats the purpose. This has to happen automatically, and "fire-and-forget" specifically means a slow or unreachable memory server doesn't become a bottleneck in the agent's own loop.

**Production tip:** This is bounded, sanitized capture, not a complete transcript — direct launches trade completeness for near-zero overhead, the right default for most day-to-day use.

### 2. Compile, Don't Retrieve

**What it is:** At session end, relevant observations are compiled into a coherent markdown page rather than left as raw log for a future retrieval step to dig through. This directly applies what's sometimes called the Karpathy LLM wiki pattern: a small index of curated pages beats a large corpus of raw history that needs re-searching every time it's needed.

**Why it matters:** RAG-style memory retrieves fragments and hopes they cohere in context. A compiled page is already coherent — a decision record or session summary in its own right, readable by a human, not just consumable mid-retrieval.

**Production tip:** This is also why the wiki stays useful with no LLM configured at all — a rule-based summarizer still produces something usable, just less polished than an LLM-consolidated page.

### 3. Cross-Agent Handoffs

**What it is:** The feature the rest of the architecture exists to support. Quit one agent CLI mid-task, open a different one in the same project later, and the new session receives a typed handoff — open questions, next steps, a session summary — before its first prompt.

**Why it matters:** This is the part almost no comparable memory tool targets directly. Most stay scoped to a single vendor's ecosystem; ai-memory's support matrix deliberately spans over twenty CLIs specifically so the handoff isn't limited to staying inside one company's tooling.

**Production tip:** Not every agent exposes a true session-end event. For those that don't, running `ai-memory finalize-session` manually after your last turn is what actually triggers the summary and handoff.

### 4. Managed Workstreams

**What it is:** An optional layer on top of the base handoff system. `ai-memory run claude`, then later `ai-memory run codex --yolo`, transparently continues one logical workstream — native per-harness session resume plus a portable, searchable event ledger — rather than just a one-time summary handoff.

**Why it matters:** A compiled handoff summary is good; native session resume with a full portable ledger is better when fidelity to "exactly what happened" matters more than a condensed summary — the difference between "roughly where you left off" and "your actual prior session, continued."

**Production tip:** Managed mode currently covers a meaningful but bounded subset of the full support matrix — check whether your specific harness is covered before assuming `ai-memory run` works everywhere the base handoff system does.

### 5. Hybrid, Authority-Aware Retrieval

**What it is:** Querying the wiki combines full-text search (FTS5), entity matching against nouns extracted at consolidation time, and graph-neighbor expansion across linked pages — with vector similarity as an optional fourth signal. Before truncation, a bounded adjustment favors maintained rules, decisions, procedures, and gotchas pages over closely matching but purely episodic session history.

**Why it matters:** Pure vector search can surface a session page that's semantically close while burying the actual standing decision on the same topic. Weighting toward curated knowledge — without making it an absolute filter — is a genuinely useful middle ground.

**Production tip:** Vector search here is additive, not foundational. FTS5 plus entity and graph-neighbor matching already works with zero embedding infrastructure; add a vector provider for better fuzzy recall, not because the system requires it.

### 6. LLM as an Opt-In, Not a Requirement

**What it is:** Capture, search, and rule-based summarization all work with no LLM provider configured. Adding one unlocks LLM-consolidated pages, contradiction linting, and background auto-improvement, but baseline usability doesn't depend on it.

**Why it matters:** This is a real architectural stance: the tool degrades gracefully rather than becoming useless the moment an API key is missing, and it keeps a genuinely free, self-hosted path available.

**Production tip:** When you do add a provider, the recommended defaults point at small, fast models — Haiku-class or mini-class — because consolidation is summarization, not hard reasoning. Save larger models for the coding agent itself.

### 7. Auto-Improvement and Curation

**What it is:** With an LLM configured, a background scheduler reviews newly completed sessions and proposes wiki edits — durable lessons a session taught — recorded in an auditable pending-writes trail. A separate, LLM-free curator command runs rule-based maintenance over cold pages, duplicate titles, and dangling links.

**Why it matters:** A memory system that only grows without revisiting what it stored accumulates noise. Having both an LLM-driven "what did we learn" loop and a cheap, deterministic housekeeping pass keeps the wiki useful over months of accumulated sessions.

**Production tip:** Auto-approval is the default, but `require_approval = true` keeps proposed edits pending for human review — worth enabling on a shared or long-lived project where a bad automatic edit is costly to fix later.

## End-to-End Walkthrough

Trace the actual scenario this tool is built around:

1. **A Claude Code session runs for several hours** on a project. Lifecycle hooks capture prompts and tool calls as sanitized, bounded observations along the way.
2. **The session ends.** ai-memory compiles the relevant observations into a session page, and creates a typed handoff containing open questions and next steps, marked pending.
3. **Hours later, Codex opens in the same directory.** Its SessionStart hook fires, and the server finds the pending handoff waiting for this project.
4. **The handoff gets injected before Codex's first prompt** — a "where you left off" block covering the architecture decisions, what was tried and rejected, and the specific open question that was never resolved.
5. **Codex continues the work** without the developer re-explaining anything that was already established.
6. **If a specific decision needs to persist permanently** rather than just live in that session's summary — "we standardized on Postgres for this" — the developer says so, and the agent writes a durable, pinned wiki page that won't get swept away by normal decay.
7. **Weeks later, in a session with a third agent entirely**, a query like "what did we decide about the database six weeks ago" hits that pinned decision page through the hybrid retrieval system — ranked ahead of any similar-sounding but purely episodic session mention, thanks to the authority-aware adjustment.

## Special Cases

**Per-project isolation by construction.** Every project lives at a path keyed by stable UUIDs rather than a raw directory name, with project identity derived from the current working directory by default. A marker file lets you override that explicitly — useful for consultancies managing multiple clients, monorepos, or linked git worktrees that should share or split memory deliberately rather than by accident.

**Per-operator memory slots on shared servers.** When a homelab or team server is shared, an opt-in setting keeps each authenticated operator's own working context in a bounded namespace, layered alongside shared context. This is context-injection isolation, not access control — exact reads and searches remain project-wide.

**Feedback that lowers confidence instead of deleting.** When a page turns out to be stale or wrong, feedback floors its retrieval salience and flags it for review rather than deleting it. A later rewrite of that page clears the flag, preserving it as an auditable record rather than silently erasing history.

## Scaling & Production Challenges

**Operational simplicity is a real design choice.** A single Rust binary, one data directory, markdown as the actual source of truth: backup is `rsync` or a git remote, recovery from one bad page edit is one file restored from a git commit, and the dataset is `grep`-able without a database client — a meaningfully different operational story than running a managed vector database alongside your agent tooling.

**Security defaults to safe, and non-loopback access fails closed.** Binding to loopback with no auth is fine for a single-user laptop; exposing the server beyond that without a bearer token is a deliberate, explicit exception rather than something that quietly works. TLS is intentionally left to a reverse proxy rather than handled internally.

**Retrieval quality has to hold up as the wiki grows across months of sessions.** This is exactly what the authority-aware ranking is for — without it, a year of accumulated episodic session pages would increasingly bury the few pages that actually matter for answering "what did we decide."

**LLM cost stays decoupled from your coding agent's cost.** Because the LLM work here is consolidation, not reasoning, running it on a cheap model keeps the memory layer's operating cost low and largely independent of whatever you're spending on the coding agent itself.

## Code Examples

Getting the server running locally with Docker, in outline:

```bash
docker run -d --name ai-memory \
    --restart unless-stopped \
    -p 127.0.0.1:49374:49374 \
    -v ai-memory-data:/data \
    -e AI_MEMORY_LLM_PROVIDER=anthropic \
    -e ANTHROPIC_API_KEY=sk-ant-... \
    akitaonrails/ai-memory:latest
```

Wiring an agent CLI to it:

```bash
ai-memory install-mcp   --client claude-code --apply
ai-memory install-hooks --agent  claude-code --apply
```

Switching agents mid-workstream with the managed launcher:

```bash
ai-memory run claude          # start work in Claude Code
# ...quit, come back later...
ai-memory run codex --yolo    # resume the same workstream in Codex
```

Querying the wiki directly over its read-only JSON API:

```bash
curl "http://127.0.0.1:49374/api/v1/search?q=database+choice&project=my-app" \
    -H "Authorization: Bearer $AI_MEMORY_AUTH_TOKEN"
```

Writing a durable, pinned decision page from the terminal:

```bash
ai-memory write-page \
    --path decisions/0007-db.md \
    --body $'# Standardized on Postgres for this project\n\nRejected MongoDB due to...' \
    --pinned
```

## Common Pitfalls

**Mistake: treating retrieved memory as instructions.** It's tempting to let a highly-ranked page steer an agent's behavior directly. **Solution:** ai-memory's own design explicitly treats retrieved text as untrusted historical evidence that never gains instruction-level authority regardless of its tier, pin, or rank — a safeguard worth copying in any memory or RAG system you build yourself.

**Mistake: assuming you need an LLM provider to get value from this.** Skipping setup entirely because you haven't picked a provider yet. **Solution:** zero-LLM mode already gives you full-text search, entity matching, graph-neighbor recall, and rule-based summaries — add a provider later specifically for consolidation quality, not as a prerequisite to starting.

**Mistake: exposing the server beyond loopback without auth.** Assuming a quick LAN test is harmless. **Solution:** treat any non-loopback bind as requiring a bearer token by default, and use the provided reverse-proxy templates rather than skipping TLS on anything beyond a single trusted machine.

**Mistake: expecting this to replace live code intelligence.** Asking the memory wiki what a function currently does. **Solution:** use it for prior decisions, rationale, and failed attempts — verify any historical code claim against the actual checkout, and lean on a structural code-intelligence tool for symbols, callers, and current behavior.

**Mistake: adopting this on a mature project without bootstrapping.** Starting from a completely empty wiki on a codebase with months of real history. **Solution:** run the bootstrap command once, which seeds initial pages from git log, README, and docs so future sessions build on existing context instead of starting from zero.

## Production Best Practices

- **Set up cross-agent hooks for every CLI you actually switch between**, not just your primary one — the handoff value only materializes once more than one vendor is wired in.
- **Pin the decisions that should outlive a single session.** Auto-compiled session pages are useful history; explicitly pinned pages are what you actually want surfacing months later.
- **Keep the LLM model for consolidation cheap and separate from your coding agent's model.** This is summarization work, and a small model handles it well at a fraction of the cost.
- **Enable manual approval for auto-improvement on shared or long-lived projects.** Auditability matters more than convenience once more than one person depends on the wiki being right.
- **Treat markdown-in-git as your real backup story**, and use it — a single bad page is one `git log` and one restore away from fixed, without touching the rest of the wiki.

## Wrapping Up

The interesting decision in ai-memory isn't any single feature — it's the bet made twice, at two different layers, that "compile a coherent summary" beats "retrieve fragments from raw history," and that memory should follow the developer across whatever agent vendor they're actually using rather than staying locked inside one company's tooling. Given how fractured the agent-CLI landscape already is in 2026, betting on the second part looks less like a nice-to-have and more like the actual shape of how most developers already work.

If you've tried wiring more than one agent CLI into the same memory system, what's actually held up well in practice — the compiled handoffs, or the raw portable ledger from managed workstreams?