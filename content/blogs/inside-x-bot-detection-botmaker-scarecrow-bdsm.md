---
title: "Inside X's Bot Detection: A Complete Guide to How botmaker, scarecrow, and bdsm Actually Flag Inauthentic Accounts"
description: "X just open-sourced the account-scoring pipeline behind its For You feed. Here's how agatha, bdsm, and user-cred-v2 score accounts, how scarecrow and botmaker turn those scores into labels, and how visibility filtering decides what you never see."
date: 2026-08-16
author: "Faizan Nadeem"
tags: ["AI Engineering", "Trust & Safety", "Backend Development", "System Design"]
image: "images/blogs/x-algorithm-bot-detection.jpg"
imageAlt: "Abstract network graph of connected nodes representing account relationship and trust data"
imageCredit: "Photo by Conny Schneider on Unsplash"
imageCreditURL: "https://unsplash.com/@choys_"
css: "/styles/blog.css"
---


On August 13, 2026, X pushed an update to its open-source `x-algorithm` repository that most coverage reduced to one headline: the exact ranking weights behind the For You feed are now public. That's a real story, but it's not the interesting one. Buried in the same update is the actual account-scoring pipeline X uses to decide which accounts look inauthentic — models with names like `agatha`, `user-cred-v2`, and, genuinely, `bdsm`. That's the literal directory name in the source tree, and it does exactly what its neighbors do: reads signals about an account and produces a score that feeds downstream into whether your posts get shown at all.

The common mistake in most explainers of systems like this is treating "bot detection" as one model with a yes-or-no verdict. It isn't, here or almost anywhere at this scale. What X actually built — and just made fully inspectable — is a layered pipeline: independent models scoring an account from different angles, a rule engine that turns those scores into labels, and a separate filtering system that decides, per post and per viewer, whether to show it, hide it behind a tap-through, or drop it outright. This post walks through how that pipeline fits together.

**You'll learn:**

- What X's account-scoring pipeline actually covers, and why it's several independent models instead of one classifier
- How `agatha`, `bdsm`, and `user-cred-v2` each score an account from a completely different signal
- How `scarecrow` and its `botmaker` rule engine turn raw model scores into enforceable labels
- How visibility filtering decides, per post and per viewer, to allow, interstitial, or drop content
- Why ranking and safety are deliberately kept as two separate systems in this architecture
- What's honestly still withheld from the public repo, and why that's a defensible tradeoff

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

### What This Pipeline Actually Covers

X's `x-algorithm` repository splits into two paths that run almost entirely independently. The **Request Path** handles a single feed request in real time: retrieve candidates, rank them, filter them, return a timeline. The **Labeling Path** runs continuously in the background, completely separate from any one user's request — accounts and posts get scored and labeled day and night, regardless of who's scrolling. Bot and inauthentic-account detection lives almost entirely in the Labeling Path, and that's the first thing worth understanding: the system that decides an account looks like a bot isn't triggered by you loading your timeline. It already ran, days or weeks earlier, and simply reads back a stored answer when your feed gets assembled.

Within the Labeling Path, account scoring specifically comes from three models working on entirely different signals: `agatha` reads how other people react to an account's posts, `bdsm` reads the account's own behavior over time, and `user-cred-v2` reads the account's position in the follow-and-engagement graph. None of them alone is "the bot detector." Together, they feed a rule engine that decides what label, if any, an account or post gets.

### Why This Architecture Is a High-Value Problem to Get Right

- **The stakes cut both ways.** A false positive silences a real account; a false negative lets a coordinated network manipulate what hundreds of millions of people see. Neither failure mode is acceptable at this scale, which is exactly why no single model carries the whole decision.
- **This is now the most inspectable system of its kind in the industry.** Meta, TikTok, and YouTube have published papers and high-level descriptions of their ranking systems; none have put a working, buildable codebase covering both ranking and safety on GitHub. That changes what "trust us" means for a platform this size.
- **The design choices here are a genuinely useful reference**, independent of X specifically, for anyone building trust-and-safety systems: how to combine multiple weak signals, where to draw the line between real-time and background processing, and how much of your own detection logic to make public without handing attackers a manual.

## The Complete Architecture

```
LABELING PATH (continuous, background)
Content Understanding (agatha, bdsm, user-cred-v2, grox, ...)
      │
      ▼
Labeling Rules (scarecrow + botmaker, abuse-enforcement-service)
      │
      ▼
Storage
      │
      ▼
REQUEST PATH (per feed request)
Candidate Sources → Scoring/Ranking → Visibility Filtering (reads Storage) → Post-Selection Filters
      │
      ▼
Ranked For You Timeline
```

The guiding principle, stated explicitly in X's own design notes: ranking and visibility are separate systems, reading separate inputs, making separate decisions. Ranking decides the *order* posts appear in; visibility filtering decides whether a post can appear *at all*. Conflating the two — letting a relevance score double as a safety score — is exactly the failure mode this architecture is built to avoid.

## Core Layers Explained

### 1. `agatha` — Scoring an Account by How Others React

**What it is:** An offline batch process that labels an account based on how other people respond to its posts — blocks, reports, and spam reports, measured relative to favorites — plus derived labels for spam-suspension risk and adult content.

**Why it matters:** This is a reaction-based signal — it looks at how real people who encounter an account's content respond to it, not at what the account does. A post that reliably draws blocks and reports relative to favorites is a strong, human-sourced signal, independent of anything the model can infer from content alone.

**Production tip:** Reaction-based signals need real exposure and real human responses before a pattern emerges, which means `agatha` alone can't catch a bad account on its first post.

### 2. `bdsm` — Scoring an Account by Its Own Behavior Over Time

**What it is:** A model that reads the sequence of actions an account takes over time — not any single action in isolation — to identify signs of inauthentic or abusive behavior.

**Why it matters:** This catches what reaction-based and content-based signals structurally can't. A single post, taken alone, can look completely ordinary. A *sequence* of actions — posting cadence, timing patterns, the rhythm of follows and engagements — can look nothing like how a person uses the platform, even when every individual action passes a content check. That's exactly why this is its own separate system rather than folded into a per-post classifier: per-post models are blind to pattern, and pattern is often the strongest signal an automated account leaves behind.

**Production tip:** Because this is about pace and pattern, not content, it can flag an account before it posts anything individually rule-breaking — valuable, but it means the threshold needs care, since legitimate power users can have unusually high-frequency, patterned usage too.

### 3. `user-cred-v2` — Scoring an Account by Its Position in the Graph

**What it is:** Runs PageRank over the follow graph and engagement edges, converting the resulting mass into a per-account credibility score.

**Why it matters:** This is a structural signal, independent of what an account posts or how it behaves moment to moment. An account genuinely embedded in real, mutually-engaged relationships accumulates graph-based credibility that's expensive to fake — scripting posts is easy, but manufacturing a convincing position inside a real social graph is much harder to fake at scale.

**Production tip:** Graph-based trust is excellent at catching networks that mostly engage with each other and stay disconnected from genuine clusters — but it's slow to update for a brand-new, legitimate account. That's exactly why it's one signal among three, not the whole story.

### 4. `scarecrow` and `botmaker` — Turning Scores Into Labels in Real Time

**What it is:** `scarecrow` reacts to events as they happen, using `botmaker` as its embedded rule engine — a purpose-built language, compiler, and runtime for rules of the shape: on this event, if these conditions hold, apply this label. The rules themselves live in a separate `botmaker-rules` directory that `scarecrow` loads.

**Why it matters:** This is the layer that connects model output to consequence. `agatha`, `bdsm`, and `user-cred-v2` each produce scores; `scarecrow` watches live events and decides, using rules written against those scores, whether a label actually gets applied right now.

**Production tip:** A dedicated rule language rather than hardcoded conditionals means rule changes don't require a full redeploy — a real advantage when rules need to adapt quickly to new evasion patterns.

### 5. `abuse-enforcement-service` — Acting on Scores Directly

**What it is:** A separate system that reads model scores about an account — not live events — and labels the account or its posts, issues a challenge, or suspends it.

**Why it matters:** This runs in parallel to `scarecrow`, on the same model outputs, but through a different mechanism: score-driven rather than event-driven. Two independent paths acting on the same signals is redundancy by design — a gap in one system's coverage doesn't automatically become a gap in enforcement.

### 6. Visibility Filtering — The Actual Gate

**What it is:** For every post and viewer combination, `visibility-filtering` returns one of three answers: **ALLOW**, **INTERSTITIAL** (a tap-through, used for adult or graphic media), or **DROP**. It reads the labels produced by everything above, plus the viewer's own blocks, mutes, and settings.

**Why it matters:** This is the only system that decides whether a post exists in a feed at all. Two rule sets run here: general rules for everyone, and a second set that applies only when a post is recommended to someone who doesn't follow the author — those out-of-network-only rules can only drop, tuned for high recall precisely because the same post stays visible to an actual follower.

**Production tip:** Evaluation stops at the first rule that returns drop — worth ordering the most confident conditions earliest, both for correctness and to avoid wasting compute on rules that never get reached.

### 7. Under the Hood — Closing the Loop With Transparency

**What it is:** A per-account reporting tool, paired with this release, aggregating the visibility-impacting labels applied to a person's own account and posts over time.

**Why it matters:** Publishing code answers "how does this system work in general," not "did it do something to *my* account." Under the Hood closes that gap — someone can see which labels landed on their account and, since the labeling code is public, trace a label back to roughly which system produced it.

## End-to-End Walkthrough

Trace a single account through the full pipeline over time:

1. **An account is created** and starts posting and engaging normally, drawing no particular attention.
2. **Content Understanding runs continuously in the background**, off the request path. `bdsm` starts building a read on the account's action sequence, `user-cred-v2` recomputes its graph-based score as edges accumulate, and `agatha`'s batch jobs track how people respond to its posts.
3. **The posting cadence starts looking automated** — rapid, repetitive, patterned in a way a person's usage typically isn't. `bdsm`'s sequence read crosses into inauthentic-behavior territory.
4. **`scarecrow`, watching live events, matches a `botmaker` rule** referencing that rising score and applies a label.
5. **In parallel, `abuse-enforcement-service`**, reading the same model score rather than the triggering event, independently crosses its own threshold and issues a challenge or suspension.
6. **The resulting labels are written to storage.**
7. **The next time any viewer's feed is assembled**, `visibility-filtering` reads those labels and evaluates its rules in order — the first drop ends evaluation, and the post never reaches that viewer's candidates.
8. **Ranking already happened independently of this decision.** The post may have scored well on relevance; visibility filtering vetoes it anyway, because the two systems answer different questions from different inputs.
9. **If the account believes this was a mistake**, Under the Hood shows exactly which labels are applied — and because the labeling code is public, anyone can trace a label back to roughly which system produced it.

## Special Cases

**Out-of-network-only drop rules.** Some rules apply only to posts recommended to a viewer who doesn't follow the author — a deliberately high-recall net cast on the discovery surface, where a false positive costs a stranger one missed post rather than silencing an account for its real audience. The identical post reaches followers untouched.

**Rules intentionally withheld from the public repo.** X has been direct about this: some `botmaker` rules, and the specific LLM prompts used by `grox`, aren't published, to reduce the risk of the code being used as a guide to circumvent the system. Full transparency and effective enforcement pull in opposite directions at the margins, and this is where X drew the line.

**Interstitial versus drop from the same pipeline.** The same labeling machinery feeds both outcomes — the difference is which rule fires and what category it's tied to. Adult or graphic content tends toward interstitial; inauthentic-behavior labels tend toward drop.

## Scaling & Production Challenges

**Keeping expensive analysis off the request path.** Sequence modeling, graph PageRank, and batch reaction analysis are all genuinely expensive operations — running all three on every account for every feed request would be untenable at this scale. Doing it continuously in the background and reading a cached, stored result at request time is what makes the economics work; reporting suggests the candidate pool alone narrows from several hundred million daily posts to roughly a few thousand per user before ranking even begins.

**Candidate isolation keeps scores cacheable.** A deliberate ranking-layer decision ensures a candidate's score doesn't depend on which other candidates are in the same batch — the same principle applies to account labels, which need to mean the same thing regardless of which request reads them back.

**Two independent enforcement paths on the same signals.** Running `scarecrow` (event-driven) and `abuse-enforcement-service` (score-driven) in parallel, off the same model outputs, costs real engineering complexity — but it means a gap in one path's timing doesn't silently become a gap in enforcement overall.

**Balancing recall differently by surface.** Stricter, higher-recall rules apply only to out-of-network recommendations, while in-network delivery to followers stays untouched — concentrating the most aggressive filtering on the surface where a platform actively introduces a stranger's content, rather than applying it uniformly everywhere.

## Code Examples

The following are illustrative reconstructions of the concepts described above, not verbatim source from the repository — some of the actual rule logic is deliberately unpublished, as noted above.

A simplified illustration of what a sequence-based account scorer conceptually evaluates:

```python
def score_action_sequence(actions: list[dict], window_hours: int = 24) -> float:
    recent = [a for a in actions if a["hours_ago"] <= window_hours]
    if len(recent) < 5:
        return 0.0
    intervals = [b["ts"] - a["ts"] for a, b in zip(recent, recent[1:])]
    regularity = 1.0 - (stdev(intervals) / (mean(intervals) + 1e-6))
    return min(regularity * len(recent) / 100, 1.0)
```

A conceptual `botmaker`-style rule, expressed as pseudocode rather than the actual rule language:

```
ON post_published
IF account.bdsm_score > 0.85 AND account.user_cred_score < 0.2
THEN apply_label(account, "likely_inauthentic")
```

A simplified illustration of the first-drop-wins evaluation order in visibility filtering:

```python
def evaluate_visibility(post, viewer, rules: list[callable]) -> str:
    for rule in rules:
        result = rule(post, viewer)
        if result == "DROP":
            return "DROP"
    return "ALLOW"
```

## Common Pitfalls

**Mistake: treating bot detection as one model with one score.** A single classifier capturing reaction patterns, behavior sequences, and graph position all at once blurs signals that are genuinely independent. **Solution:** score each dimension separately and combine them downstream in the rule layer, not upstream in the model.

**Mistake: letting a relevance score double as a safety score.** It's tempting to fold "is this good content" and "is this a bad actor" into one number. **Solution:** keep ranking and visibility as genuinely separate systems reading separate inputs.

**Mistake: running expensive account-level analysis on the request path.** This is the fastest way to make a detection system too slow to deploy at scale. **Solution:** do the heavy analysis continuously in the background and keep the request-time read cheap.

**Mistake: publishing every rule to maximize transparency.** It sounds trustworthy, but it also hands anyone motivated a precise map of what to avoid triggering. **Solution:** be explicit about what's withheld and why, rather than implying completeness you don't have.

**Mistake: treating any single flagged action as sufficient evidence.** One unusual post, alone, is weak evidence of anything. **Solution:** build for pattern over time — the entire reason a sequence-based model exists as its own system.

## Production Best Practices

- **Separate your signals before you combine them.** Reaction-based, behavior-based, and graph-based signals catch different failure modes — collapsing them too early loses information a rule layer downstream could use.
- **Keep safety and ranking as distinct systems.** They answer different questions and should be allowed to disagree — a highly relevant post can still be one you choose not to show.
- **Do expensive scoring continuously, not on the request path.** Background computation plus a cheap, cached read is what keeps a layered detection system viable at scale.
- **Build redundant enforcement paths where the stakes are high.** Independent systems acting on the same signals, through different mechanisms, catch each other's blind spots.
- **Be honest about what you withhold, and why.** A credible transparency effort names its limits explicitly rather than implying completeness it doesn't have.

## Wrapping Up

The headline framing — "X published its ranking weights" — undersells what actually shipped here. The genuinely notable part is a full, inspectable account-scoring and enforcement pipeline: independent models reading reactions, behavior sequences, and graph position, a purpose-built rule engine turning those scores into labels, and a visibility layer that keeps the safety decision entirely separate from the relevance decision. Whatever you make of the platform it's built for, the architecture itself is a genuinely useful reference for anyone building trust-and-safety systems of their own.

If you've read through the actual repository yourself, what part of the pipeline surprised you most — the account-level models, or how much of the rule logic is deliberately left out?