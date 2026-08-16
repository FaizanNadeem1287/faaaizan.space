---
title: "Needle: A Complete Guide to Running Function-Calling AI on Tiny Devices"
description: "How Cactus Compute's Needle 2 packs tool calling into a single 14MB, 45M-parameter binary — the Simple Attention Network architecture, grammar-constrained decoding, confidence gating, and what it actually takes to run structured AI on a phone, watch, or robot."
date: 2026-08-16
author: "Faizan Nadeem"
tags: ["AI Engineering", "Edge AI", "Backend Development", "LLM"]
image: "images/blogs/needle-tiny-device-ai.jpg"
imageAlt: "Macro close-up of a computer chip on a printed circuit board"
imageCredit: "Photo by Alexandre Debiève on Unsplash"
imageCreditURL: "https://unsplash.com/@alexkixa"
css: "/styles/blog.css"
---


A post about a 26-million-parameter open-source model has been circulating on LinkedIn: Needle, from Cactus Compute, distilled from Gemini for function calling on phones and wearables. It's a genuinely interesting project — but the repository has already moved past that version. Cactus shipped Needle 2 within the last few days: 45 million parameters, compressed into a single 14MB binary, built on a new architecture recipe they call a Simple Attention Network. The core idea holds and gets sharper: it's not a smaller chatbot, it's a fundamentally different kind of model, one that structurally cannot produce a rambling free-text answer even if you wanted it to.

The common mistake in how this class of model gets covered is treating it as "GPT but tiny." It isn't, and the ways it isn't are the actually interesting engineering decisions: every token the model emits is constrained by a grammar compiled from your own function schemas, every response carries a calibrated confidence score you can act on programmatically, and an off-topic query doesn't get a made-up answer — it gets an explicit, structured refusal. This post walks through how that's built, and what building on top of it actually looks like.

**You'll learn:**

- Why tool calling is a fundamentally different problem from open-ended generation, and why that lets the model be this small
- How Needle 2's Simple Attention Network architecture differs from a standard transformer
- How byte-level grammar-constrained decoding guarantees valid structured output instead of hoping for it
- How confidence gating and tool retrieval work, and why they matter for production reliability
- How to fine-tune Needle on your own tools with LoRA, and deploy the result to the same lightweight engine
- The real limitations of a model this specialized, and where it fits versus a general-purpose LLM

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

### What Needle Actually Covers

Needle solves exactly one problem: given a query and a set of declared tools, decide which tool to call and what arguments to fill in — or explicitly decline if nothing declared applies. Performing an action and extracting structured data from text turn out to be the same operation under the hood; the only difference is what you declare as the "tool." This is a deliberately narrow scope, and that narrowness is the entire point. Tool calling is fundamentally a retrieval-and-assembly task — match a query to a tool name, pull argument values out of the input, emit valid JSON — not open-ended reasoning. A model built specifically for that job doesn't need the enormous general-knowledge capacity a chatbot needs, which is exactly why 45 million parameters, compressed to a 14MB binary, can hold its own against models many times its size on this specific task.

### Why This Is a High-Value Problem to Solve Well

- **On-device AI has real constraints a cloud model never has to meet.** RAM measured in tens of megabytes, no guaranteed network connection, and battery budgets that make a 1B+ parameter model a non-starter on a watch or a pair of glasses.
- **Latency-sensitive features shouldn't be reaching for a cloud LLM at all.** A round trip to a hosted model for "turn off the lights" is both slower and less private than a model that already lives on the client and answers instantly.
- **Structural guarantees matter more than raw capability for this task.** A tool-calling system that occasionally emits malformed JSON or hallucinates a parameter isn't just inconvenient — it's a broken integration. Needle's whole design is built around making that class of failure structurally impossible rather than just statistically rare.

## The Complete Architecture

```
Tool schemas ──▶ Grammar compiler (byte-level)
                         │
Query + schemas ──▶ Simple Attention Network ──▶ constrained decode ──▶ structured call
                                                                              │
                                                                       execute locally
                                                                              │
                                                                    result fed back in
                                                                              │
                                                                        next turn / final answer
```

The guiding principle: the model's only degrees of freedom are the ones the grammar allows. Correctness is enforced by construction — a byte-level grammar compiled from your declared schemas constrains every token during decoding — not by hoping a much larger, much slower model happens to behave.

## Core Layers Explained

### 1. The Simple Attention Network Architecture

**What it is:** Needle 2's dense small-model recipe, described in Cactus's own paper (arXiv:2607.18363). In place of a standard feed-forward block, it uses a Hadamard-transform-based MLP, grouped-query attention, an "engram" key-value memory built from hashed n-gram tables, and multi-lane hyper-connections combining several residual streams. The routing between components is normalized through Sinkhorn iteration rather than a simple softmax.

**Why it matters:** Every one of these choices trades a piece of the "general capability" a standard transformer spends parameters on for efficiency specific to this task. The original 26M Needle went further still, using pure attention and gating with no feed-forward layers at all — a bet that tool calling doesn't need the kind of associative "knowledge storage" an FFN typically provides. Needle 2 reintroduces a lightweight MLP, but a deliberately cheap one built on a fixed, weight-free Hadamard transform rather than a dense matrix.

**Production tip:** Don't expect this architecture to generalize to open-ended chat quality — it's optimized specifically for the retrieval-and-assembly shape of tool calling, and that specialization is exactly why it's small enough to run in 28MB of RAM.

### 2. Byte-Level Grammar-Constrained Decoding

**What it is:** Every tool schema you declare gets compiled into a grammar that constrains what tokens the model is even allowed to emit at each decoding step. The model cannot produce malformed JSON, an unrecognized tool name, or a value that violates a declared constraint — not because it was trained not to, but because the invalid tokens are simply unavailable during decoding.

**Why it matters:** This converts "the model should return valid JSON" from a probabilistic hope into a guarantee. Constraints like value ranges, regex patterns, string lengths, and enums — anything expressible in the schema — get compiled directly into the grammar, so an argument outside your declared bounds literally cannot be generated.

**Production tip:** Push validation into your schema wherever possible rather than checking it after the fact in application code. A `Literal["heat", "cool", "auto"]` or a numeric range declared up front means the model can't emit an invalid value in the first place — there's nothing to catch downstream.

### 3. Confidence Gating

**What it is:** Every response carries a confidence score that's the minimum of two independent signals: a calibrated head that scores the full prompt plus the produced call, and the raw decoding probability of the call tokens. Both have to agree for a high score.

**Why it matters:** This gives you a single, principled number to build a real escalation policy around: act locally above your chosen threshold, and re-ask or route to a larger cloud model below it. The failure mode this is designed to produce is escalation, not silent wrong execution.

**Production tip:** Tune the threshold per use case, not globally — a smart-home toggle can tolerate a lower bar than something like a payment or a message send, where a wrong low-confidence call executing silently is a much worse outcome than asking again.

### 4. Tool Retrieval for Large Catalogs

**What it is:** With five or fewer declared tools, everything renders directly into context. Past that, a built-in retrieval head embeds each tool schema once, embeds the query every turn, and narrows the field to the five highest-scoring tools — with the decoding grammar rebuilt over just that subset.

**Why it matters:** An unselected tool isn't just unlikely to get called — it's genuinely unreachable for that turn, since the grammar itself only permits the retrieved subset. This is what lets a device declare a large tool catalogue without either blowing the context budget or slowing decoding down trying to consider all of it every turn.

**Production tip:** Persist the tool embeddings to disk rather than recomputing them every session. They're keyed by a fingerprint over the schema set and the model, so an unchanged catalogue loads instantly and only a genuinely changed schema forces re-embedding.

### 5. Bounded Memory Regardless of Conversation Length

**What it is:** A 256-token sliding window handles the running conversation, while the declared tools are pinned in place as fixed key-value "sinks" rather than sliding out of context.

**Why it matters:** Total memory use stays close to 28MB no matter how long a conversation runs — a hard requirement for anything actually deployed on a wearable or embedded device, where memory that grows unbounded with usage simply isn't viable.

### 6. System Facts, Not System Instructions

**What it is:** An optional system turn carries environment state — date, locale, device type, battery level, and similar recognized keys — strictly as facts the model can reason from, never as instructions that steer its behavior.

**Why it matters:** This is a deliberate security-adjacent design choice. Because the model is trained to treat this field as data rather than commands, text placed there doesn't get a chance to override or redirect the model's actual behavior — closing off a class of prompt-injection-style manipulation that a more permissive "system prompt" design would leave open.

**Production tip:** Resolve relative time references ("tomorrow at 7") only work when a `date:` fact is actually supplied — without it, the model passes the phrase through rather than guessing at an absolute time, which is the safer failure mode for anything time-sensitive.

### 7. LoRA Fine-Tuning on a Weights-Agnostic Engine

**What it is:** Needle fine-tunes with LoRA against a frozen base model, then merges the adapter and quantizes the result into a single `.cact` file — a self-contained artifact that runs on the exact same inference engine as the base model, no separate build step required.

**Why it matters:** This makes specializing the model on your own tool catalogue genuinely cheap, and the output stays as simple to deploy as the original: one file, one engine, no recompilation.

**Production tip:** If you don't have labeled examples yet, the toolchain can synthesize training data directly from your tool schemas via an OpenRouter-backed generation step — a reasonable way to bootstrap a first fine-tune before you have real usage logs to learn from.

## End-to-End Walkthrough

Trace a request from query to executed action:

1. **Tools are declared**, either as decorated Python functions or as raw JSON schemas — describing a `set_lights` tool with `room`, `on`, and `brightness` parameters, for example.
2. **The grammar compiler builds a constrained decoding grammar** from those schemas the moment the tools are registered.
3. **A query arrives**: "dim the living room to 30."
4. **The model processes the query against the declared tools** and produces a call, with every token constrained by the compiled grammar — it's structurally impossible for it to emit a call to an undeclared tool or a malformed argument.
5. **The response includes the call itself, a short natural-language reasoning trace, and a confidence score** — something like `set_lights(room="living room", on=true, brightness=30)` at a high confidence.
6. **Confidence clears the configured threshold**, so the call executes locally without escalation.
7. **For a multi-step task** — say, "message Alex that I'm running late" — the model first calls a `search_for_contact` tool, the result (a contact ID) gets fed back into the next turn, and the model then calls `send_instant_message` using that returned ID, chaining calls the way a longer workflow requires.
8. **A final turn may answer in plain text** once all necessary calls are complete, returned as a distinct "respond" type with no further function calls attached.

## Special Cases

**Extraction is tool calling with one tool.** Declaring a single schema — a record shape like an invoice or a receipt — and passing text where a query would go returns the parsed fields as that schema's arguments. Because only one tool is declared, the grammar admits exactly one call of that name, so schema conformance is guaranteed rather than merely likely.

**Off-topic queries get a structured refusal, not a free-text answer.** If no declared tool can serve a request, the model returns an empty call. There's no fallback to conversational text — that's the entire contract for handling anything outside the declared scope, and it's what keeps the model's behavior predictable in an embedded integration.

**Optional arguments are omitted, not guessed.** If the input doesn't provide evidence for an optional field, it's left out of the call entirely rather than filled with a plausible-sounding guess — the same discipline applied to arguments that applies to whole off-topic queries.

## Scaling & Production Challenges

**Fine-tuning at fleet scale still has to fit the same tiny footprint.** A LoRA adapter merges back into a single quantized `.cact` file, which means specializing the model per customer, per device class, or per tool catalogue doesn't multiply your deployment complexity — every variant still runs on the identical lightweight engine.

**Confidence threshold tuning is an ongoing operational decision, not a one-time setting.** As your tool catalogue grows or usage patterns shift, the right escalation threshold for a given action can drift — treat it as a metric to monitor, not a constant to set once and forget.

**Retrieval quality matters more as your tool catalogue grows.** Past five tools, correctness depends on the retrieval head actually surfacing the right five candidates every turn — a catalogue with many near-duplicate or poorly-described tools will degrade retrieval accuracy well before it exhausts the model's raw capacity.

**This is a narrow specialist, and benchmarking it like a general model misses the point.** Needle trades wins with function-calling-focused models many times its size on single-shot tool-call accuracy — but that comparison is specifically about tool calling, not general reasoning or conversation, and treating strong function-calling benchmarks as evidence of broader capability would be a mistake.

## Code Examples

Declaring a tool with a decorator — the signature and docstring alone are enough for simple cases:

```python
import needle

@needle.tool
def get_weather(city: str):
    "Get the current weather for a city."
    return {"city": city, "temp_c": 27, "sky": "clear"}

agent = needle.Needle(tools=[get_weather])
print(agent.run("what's it like in Lagos right now?"))
```

Adding real constraints — ranges and patterns compiled directly into the decode grammar:

```python
from typing import Annotated

@needle.tool
def send_money(
    amount: Annotated[float, needle.Field(gt=0, le=10000)],
    to: Annotated[str, needle.Field(pattern=r"^@[a-z0-9_]+$")],
):
    "Send money to a handle."
    return {"sent": amount, "to": to}
```

Structured extraction with a typed result:

```python
from pydantic import BaseModel

class Invoice(BaseModel):
    vendor: str
    total: float
    due_date: str

invoice = needle.extract("Invoice from Acme Corp, $1,200.00, due 2026-09-01", Invoice)
```

Fine-tuning on your own tools, end to end:

```bash
needle generate-data --tools my_tools.json --num-samples 500 --output data.jsonl
needle finetune data.jsonl --epochs 3
needle build checkpoints/needle2.pkl --lora checkpoints/needle_lora.pkl --out my_needle.cact
```

## Common Pitfalls

**Mistake: treating Needle as a small chatbot.** It has no free-text fallback by design — any conversational use case outside tool calling or extraction is the wrong fit. **Solution:** route open-ended conversation to a general-purpose model and keep Needle scoped to structured actions.

**Mistake: declaring a large tool catalogue without accounting for retrieval.** Past five tools, only the top five retrieved candidates are reachable per turn. **Solution:** write clear, distinct tool descriptions so the retrieval head can actually differentiate between them.

**Mistake: ignoring the confidence field.** Taking whatever call comes back without checking confidence throws away the model's own signal about how sure it is. **Solution:** set a real per-action threshold and build an escalation path for anything below it.

**Mistake: putting behavioral instructions in the system facts field.** The model is trained to treat that field as data, not commands, so instructions placed there won't steer it. **Solution:** put behavior guidance in tool descriptions and schemas, where the model actually reads it as directive.

**Mistake: skipping fine-tuning for a highly specific tool ontology.** The base model is general-purpose within tool calling; a narrow or unusual catalogue may not be its strongest fit out of the box. **Solution:** LoRA fine-tuning on your own schemas is cheap enough to be worth doing before assuming accuracy is capped.

## Production Best Practices

- **Push constraints into the schema, not into post-hoc validation.** Ranges, patterns, and enums declared up front are enforced during generation, not caught after the fact.
- **Build a real escalation path around the confidence score.** Decide per action what "below threshold" should do — retry, ask a clarifying question, or hand off to a larger model.
- **Keep tool descriptions distinct and specific**, especially once you're relying on the retrieval head for a larger catalogue.
- **Treat the system facts field as data only.** Don't rely on it to steer behavior — that's not what it's trained for.
- **Fine-tune early if your tool catalogue is unusual.** A cheap LoRA pass on your own schemas is a small cost against the risk of shipping with base-model accuracy that isn't tuned to your domain.

## Wrapping Up

The real story here isn't "a 26M model can do function calling" — that framing actually undersells what's already shipped, since Needle 2 has moved past it with a purpose-built architecture, grammar-guaranteed structured output, and a confidence contract built for production escalation. The underlying idea is the one worth keeping: not every AI task needs a billion-parameter model, and once you know precisely what a model needs to do, spending its entire parameter budget on doing that one thing well beats spending most of it on general capability you'll never use.

Are you seeing more teams reach for specialized tiny models like this for the narrow, structured slices of their agent stack, or is the pull toward one big general-purpose model still winning out in practice?