---
title: "Open Knowledge Format (OKF): A Complete Guide to Building a Self-Updating Codebase Knowledge Graph"
description: "What Google's new Open Knowledge Format actually standardizes, how it maps onto a real codebase instead of BigQuery tables, and how to build the enrichment pipeline that keeps it accurate as code changes."
date: 2026-08-12
author: "Faizan Nadeem"
tags: ["AI Engineering", "RAG", "Backend Development", "LLM"]
image: "images/blogs/okf-codebase-knowledge-graph.jpg"
imageAlt: "Interconnected markdown files representing a codebase knowledge graph"
imageCredit: "Photo by Shutter Speed on Unsplash"
imageCreditURL: "https://unsplash.com/@shutter_speed_"
css: "/styles/blog.css"
---

Point a coding agent at a real repository and watch what actually happens before it writes a single line: it reads files, greps for callers, opens a handful of related modules, and rebuilds — from scratch, every single task — a mental model of how your codebase fits together. That work isn't free. It's tokens, and on a repo of any real size, it's a lot of them, spent re-deriving the same structural facts your team already knows and could have written down once.

Most teams treat this as a retrieval problem and reach for RAG. That's the common mistake this post pushes back on. Google Cloud's new Open Knowledge Format, released in June 2026, is built on a different premise entirely: some knowledge about your system — what a service does, what it depends on, who owns it — is stable enough that it shouldn't be re-derived on every query. It should be compiled once, like source code, and reused. The spec itself is almost nothing — one required field, plain Markdown, no SDK. The real engineering problem, the one every explainer of the format skips, is keeping that compiled knowledge honest while a team is still shipping dozens of commits a day. That's what this post actually walks through.

**You'll learn:**

- What OKF actually standardizes, and how deliberately minimal the spec is
- Why compiled context beats RAG for the stable, curated slice of a codebase — and where RAG still wins
- How an OKF "concept" file maps onto real code instead of Google's original BigQuery-table use case
- How to build a git-hook-triggered enrichment pipeline that keeps the graph from drifting out of sync with your repo
- How progressive disclosure turns a knowledge bundle into an actual token-budget control for multi-agent workflows
- The honest limitations of a spec this new, and how to decide if it's worth adopting yet

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

### What OKF Actually Covers

The Open Knowledge Format is a specification for representing curated knowledge as a directory of plain Markdown files with YAML frontmatter, published by Google Cloud's data team in June 2026. It formalizes a pattern that had already emerged organically across the ecosystem — Andrej Karpathy's widely-shared "LLM wiki" idea, the AGENTS.md convention now used across tens of thousands of open-source projects, and developers wiring Obsidian-style vaults directly into coding agents. OKF doesn't invent a new substrate; it standardizes the one that already won: Markdown, frontmatter, and Git.

A "concept" is the atomic unit — one file, one piece of knowledge, and the file's path is its identity, with no separate ID system to keep in sync. Of the handful of recommended frontmatter fields, exactly one is required: `type`. Everything else — title, description, a resource link, tags, a timestamp — is optional. That minimalism isn't an oversight; it's the entire design philosophy. This is a wire format, not a platform, and the spec is deliberately lenient about what it tolerates: unrecognized types, missing optional fields, even broken links are all things a conformant consumer has to accept rather than reject.

### Why This Is a High-Value Problem to Solve

- **Token costs compound with every agent, every task, and every teammate running one.** Re-deriving the same structural facts about a codebase isn't a one-time cost — it's a recurring tax on every single interaction.
- **The alternatives are both flawed for this specific kind of knowledge.** A naive full-repo dump doesn't scale past a small codebase, and RAG re-derives relationships from raw documents at query time, every time, for knowledge that doesn't actually change that often.
- **Stale context isn't just slow — it's actively wrong.** An agent acting on an outdated mental model of a service's dependencies doesn't just waste tokens; it can make changes that break something it didn't know was connected.

## The Complete Architecture

```
Commit Pushed
      │
      ▼
Diff-Scoped Scan  (which services/modules actually changed?)
      │
      ▼
Draft / Update Concept Docs  (two-pass enrichment agent)
      │
      ▼
Re-link Cross-References  (update dependency/responsibility links)
      │
      ▼
Lint  (spec-compliance rules)
      │
      ▼
Publish  (commit the bundle, push to CI, or register to a catalog)
```

The guiding principle: compile the stable knowledge once, so agents stop re-deriving it on every task. OKF itself has no opinion on how that compilation happens or stays current — that part is entirely up to whoever adopts it, and it's the actual engineering work this post focuses on.

## Core Layers Explained

### 1. The Concept File

**What it is:** A single Markdown file representing one unit of knowledge — a service, an API, a table, a runbook — with a small YAML frontmatter block up top and a free-form Markdown body underneath.

**Why it matters:** Because the file path *is* the identity, there's nothing to reconcile between a separate ID system and the actual location of the knowledge — the filesystem itself is the index.

```markdown
---
type: Service
title: billing-service
description: Handles subscription billing, invoicing, and payment webhooks.
resource: https://github.com/yourorg/monorepo/tree/main/services/billing
tags: [billing, payments, python]
timestamp: 2026-08-01T10:00:00Z
---
# Responsibilities
Owns the Invoice and PaymentEvent domain models.

# Dependencies
- Calls customer-service to resolve account state.
- Publishes events consumed by notifications-service.
```

**Production tip:** Mirror the original use case's structure deliberately — swap a BigQuery console URL for a repo path in `resource`, and swap a data `# Schema` section for a code-oriented `# Responsibilities` / `# Dependencies` structure. Consistency in body structure across concept files matters more for agent consumption than any single field does.

### 2. `index.md` and Progressive Disclosure

**What it is:** A reserved filename that acts as a directory listing at any level of the bundle — the entry point an agent reads before touching anything else.

**Why it matters:** This is a genuine token-budget control, not a convenience file. An orchestrating agent reads `index.md`, decides which concept files a given subtask actually needs, and only loads those — nobody pulls the entire bundle into context for a change that touches one service.

**Production tip:** Keep index entries to a title and a one-line description pulled straight from each concept's frontmatter. An index bloated with detail defeats the point of having one.

### 3. `log.md` and Change History

**What it is:** Another reserved filename — a chronological, date-grouped record of what changed in what the bundle *knows*, distinct from `git log`, which only records what changed in the code.

**Why it matters:** "What changed in this file" and "what changed in our understanding of this system" are different questions. A service's code can change without its documented responsibilities changing, and vice versa — a `log.md` lets an agent (or a person) answer the second question directly instead of inferring it from commit history.

### 4. Cross-Links as a Dependency Graph

**What it is:** Ordinary relative Markdown links between concept files, with the relationship meaning carried by surrounding prose rather than a formal schema.

**Why it matters:** These links turn a flat directory into something closer to a dependency graph than a simple folder hierarchy — `billing-service` can point at everything it calls and everything it publishes to, independent of where those services actually sit in the repo tree.

**Production tip:** Because links are just Markdown and the spec requires consumers to tolerate broken ones, don't treat this as a substitute for actual static analysis of your call graph. It's a curated, human-and-agent-readable *summary* of the dependency graph, not a formally verified one.

### 5. The Enrichment Agent Pipeline

**What it is:** The part OKF deliberately leaves unspecified — the process that actually drafts and maintains concept files as code changes. The reference pattern is two passes: one that walks changed services and drafts or updates a concept file from their interfaces and structure, and a second that adds citations back to existing documentation, runbooks, and PRs.

**Why it matters:** This is the real engineering work behind adopting OKF. The format itself is close to free to prototype — one required field, no SDK. Keeping a thousand-file bundle accurate while a team ships dozens of commits a day is the actual systems problem, and it's squarely on you to solve, not the spec.

**Production tip:** Scope every enrichment pass to the git diff, not the whole repository. A full-repo re-scan on every commit is what makes this expensive; a diff-scoped scan of just the changed services is what makes it cheap enough to run on every push.

### 6. Tooling: Init, Hooks, Search, and Lint

**What it is:** An open-source CLI (independent of Google, written in Go) that scaffolds a bundle, installs a git hook to keep it updating automatically, supports keyword search over concepts, and enforces a set of built-in spec-compliance rules.

**Why it matters:** You don't have to build the automation layer from zero — the shape of "scan on commit, refresh what changed, lint before trusting it" is already proven out by existing tooling, even this early in the ecosystem.

**Production tip:** Run lint as a hard gate before your CI trusts a bundle enough to let agents act on it. Given how lenient the spec itself is about missing fields and broken links, lint is the only thing standing between "technically conformant" and "actually useful."

## End-to-End Walkthrough

Trace one commit through the full pipeline:

1. **A developer pushes a change** to `billing-service`, adding a new dependency on a fraud-check service.
2. **The git hook fires** and triggers a diff-scoped scan — only `billing-service` and anything directly touched by the diff gets re-examined, not the entire repository.
3. **Pass one of the enrichment agent drafts the update**, reading the service's new interface and call sites, and rewrites the `# Dependencies` section of `billing-service.md` to include the fraud-check service.
4. **Pass two adds citations**, cross-referencing the change against any existing runbook or PR description that explains *why* the dependency was added, and links them in.
5. **Cross-references get updated on both sides** — `billing-service.md` now links to the fraud-check concept file, and if that file's `# Dependents` section exists, it gets updated to point back.
6. **Lint runs** against the updated bundle, checking frontmatter validity and catching anything the enrichment pass got structurally wrong.
7. **The bundle publishes** — committed alongside the code change, or pushed through CI to a served location.
8. **The next day, an orchestrator agent picks up an unrelated task** touching the fraud-check service. It reads `index.md`, sees the concept file, loads just that one file — including its now-current link back to `billing-service` — and never has to re-scan either codebase from scratch to understand the relationship.

## Special Cases

**Where OKF stops and RAG starts.** OKF is for the stable, curated slice of a codebase's knowledge — services, ownership boundaries, dependency graphs, the runbooks someone actually bothered to write. RAG is still the right tool for the long tail: one-off design docs, Slack threads, old tickets — anything too unstructured or too rarely-referenced to justify curating into a concept file. Treating OKF as a RAG replacement is the wrong frame; treating it as a compiled cache that spares RAG from re-deriving stable facts is the right one.

**Multi-team monorepos and type drift.** Because `type` is producer-defined with no external registry, different teams contributing to the same bundle will naturally drift — one team's "API Endpoint" is another's "Route." Nothing in the spec prevents this; only your own lint rules and conventions can hold the line.

**Governed metrics and certified computations.** The spec includes a concept type for what it calls an "attested computation" — a sanctioned, checkable way to compute a value, distinct from just documenting what the value means. For teams with metrics that need a single source of truth for *how* they're calculated, not just what they represent, this is worth a dedicated concept file rather than folding the computation logic into a general description.

## Scaling & Production Challenges

**Full-repo rescans don't scale, diff-scoped ones do.** The entire cost model of this pipeline depends on scoping enrichment passes to what actually changed. A team that re-scans everything on every commit will find the pipeline too expensive to keep running well before their repo gets large enough to actually need it.

**Progressive disclosure has to be designed into your orchestration layer, not assumed.** The token-budget benefit of `index.md` only materializes if your orchestrating agent is actually built to read the index first and load concept files selectively — bolting OKF onto an agent that still dumps whole directories into context gets you the maintenance cost without the savings.

**Drift and governance get harder, not easier, at scale.** With no type registry and no enforced link semantics, a bundle maintained by a handful of people stays coherent through convention alone; a bundle touched by dozens of contributors needs deliberate lint rules and review discipline to avoid degrading into an inconsistent mess that's technically spec-conformant and practically unreliable.

**This is a very early ecosystem.** The spec was published in June 2026, tooling outside Google's own reference implementation is fragmented, and there are no long-running production case studies yet on large, actively-changing codebases. Budget for the spec itself to keep evolving, not just your bundle.

## Code Examples

Scaffolding a bundle and wiring up the automatic-update hook:

```bash
okf init
okf hook install
okf search -q "billing"
okf lint
```

A minimal Python wrapper for calling the CLI from your own CI pipeline:

```python
import subprocess

def okf_lint(bundle_path: str = ".okf/knowledge") -> bool:
    result = subprocess.run(
        ["okf", "lint", bundle_path],
        capture_output=True, text=True,
    )
    if result.returncode != 0:
        print(result.stdout)
    return result.returncode == 0
```

The load-and-search primitives your orchestrator calls before dispatching a sub-agent, and the lint check your CI calls before trusting a bundle:

```go
bundle, err := okf.LoadBundle(".okf/knowledge", nil)
results := bundle.Search("billing")
report := lint.LintBundle(concepts, lint.DefaultConfig())
```

## Common Pitfalls

**Mistake: treating OKF as a RAG replacement.** Trying to cram unstructured, rarely-referenced knowledge into curated concept files defeats the format's whole premise. **Solution:** keep OKF for stable, curated knowledge and leave RAG in place for the long tail.

**Mistake: re-scanning the whole repo on every commit.** This is the single fastest way to make the pipeline too expensive to keep running. **Solution:** scope every enrichment pass to the git diff, not the full codebase.

**Mistake: skipping lint because the spec is lenient by design.** Spec-conformant isn't the same as trustworthy — a bundle full of broken links and drifted types can still technically pass. **Solution:** run lint as a hard CI gate before agents are allowed to act on the bundle.

**Mistake: adopting OKF before you have agents actually consuming it.** The entire value proposition depends on something reading the bundle. **Solution:** if nothing in your workflow consumes structured context yet, you're maintaining a wiki nobody reads — wait until you have agentic workflows that would actually use it.

**Mistake: skipping the citation pass.** A single-pass enrichment agent that only drafts, without cross-referencing existing documentation, tends to produce plausible-sounding but unverified descriptions. **Solution:** keep the two-pass structure — draft, then cite — rather than trusting a first pass as final.

## Production Best Practices

- **Scope enrichment to the diff, always.** This is the difference between a pipeline that runs on every commit and one that gets disabled after the first expensive week.
- **Design your orchestrator around progressive disclosure from the start.** The token savings are real, but only if something actually reads `index.md` before loading concept files.
- **Lint before you trust.** The spec's leniency about missing fields and broken links means governance is entirely your responsibility, not the format's.
- **Keep OKF and RAG as complementary tools, not competitors.** Route stable, curated knowledge through OKF and everything else through RAG.
- **Start with your most-changed service, not the whole repo.** A minimal pipeline — `okf init`, a git hook, one enrichment pass over your highest-churn service — is enough to measure real token savings before committing to a full rollout.

## Wrapping Up

The spec itself really is close to nothing — one required field, plain Markdown, no SDK to install. That's precisely why it's adoptable, and precisely why it isn't the interesting part. The actual system worth building is the enrichment pipeline that keeps a knowledge graph honest while your codebase keeps changing underneath it, and that's an engineering problem no spec can hand you pre-solved. If you're already running multiple agents against your own repo, this is worth prototyping this week — point a git hook at your most-active service and measure the token delta on your next multi-agent task before deciding whether the maintenance cost is worth it.

If you tried this on your own codebase, what would you scope the first enrichment pass to — your most-changed service, or the one with the most tangled dependencies?