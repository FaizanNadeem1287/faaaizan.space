---
title: "UUIDs Explained: A Complete Guide to UUID Versions, Collision Odds, and Primary Key Performance"
description: "What UUIDs actually are, the real difference between v1, v4, v5, and v7, whether UUIDs can really collide, and why UUID v4 is the wrong default for a database primary key in 2026."
date: 2026-08-03
author: "Faizan Nadeem"
tags: ["Python", "Database Design", "Backend Development", "UUID"]
image: "images/blogs/uuid-guide.jpg"
imageAlt: "Hexadecimal identifier strings representing unique database keys"
imageCredit: "Photo by Artturi Jalli on Unsplash"
imageCreditURL: "https://unsplash.com/@artturijalli"
css: "/styles/blog.css"
---

Ask most backend developers what a UUID is and you'll get the same answer: "a random string that's basically unique, so I don't have to think about it." That's the common mistake — treating a UUID as a single, interchangeable thing instead of a family of different generation strategies with genuinely different tradeoffs. There isn't one UUID. There's a random one, a timestamp-based one, two deterministic hash-based ones, and — as of the finalized RFC 9562 spec — newer time-sortable versions that most teams still aren't using by default, even though they arguably should be.

This gap matters more than it looks like it should. Pick the wrong version for a primary key and you won't get an error — you'll get a database that quietly gets slower to write to as it grows, because nobody thought about how a 128-bit random number interacts with how B-tree indexes actually work. This post covers what's really encoded in a UUID, how each version builds its bits, the actual math behind whether two UUIDs can collide, and why "just use `uuid4()`" is exactly the kind of one-check thinking that causes production pain a year later.

**You'll learn:**

- What's actually encoded in the 128 bits of every UUID — including the version and variant nibbles you can read at a glance
- The real difference between UUID v1, v4, v3/v5, and the newer time-sortable v6/v7/v8
- The actual math behind UUID collision probability, and why real-world duplicate UUIDs almost never come from that math at all
- Why random UUIDs (v4) quietly hurt B-tree index performance when used as primary keys
- Why UUIDv7 is becoming the default choice for primary keys, and how to generate one in Python
- How to migrate an existing UUID-keyed schema without a risky big-bang rewrite

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

### What a UUID Actually Is

A UUID (Universally Unique Identifier) is a 128-bit value, usually written as 32 hex characters split into five groups in an 8-4-4-4-12 pattern:

```
550e8400-e29b-41d4-a716-446655440000
```

Two of those hex digits aren't random or timestamp data at all — they're metadata describing the UUID itself. The **version** nibble tells you how it was generated (timestamp-based, random, hash-based, or newer time-ordered schemes), and the **variant** field tells you which layout standard is in use — almost everything you'll encounter follows the standard laid out in RFC 9562, the 2024 update that formally added the newer versions on top of the original RFC 4122. That's the elegant part of the design: you can look at any UUID and know its "recipe" from a couple of characters, with no lookup table required.

### Why Choosing the Right Version Is a High-Value Decision

- **UUIDs show up everywhere** — primary keys, session tokens, request IDs, S3 object keys, idempotency keys — so a wrong assumption about one use case tends to get copy-pasted into every other one.
- **The wrong version as a primary key has a real, measurable performance cost.** Random UUIDs inserted into a B-tree index don't just "work a little slower" — they can meaningfully increase insert latency and index size at scale, for reasons that have nothing to do with collision risk.
- **Not every use case wants randomness.** Some need reproducibility (the same input always yielding the same ID), and reaching for the wrong tool there creates duplicate records instead of preventing them.

## The Complete Architecture

```
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
   │        │    │   │        │
 time/random   version  variant   time/random/node
              (how built)  (layout std)
```

The guiding principle: the version nibble is a UUID's own built-in instruction label for how every other bit in it should be interpreted. Nothing about a UUID's uniqueness or sortability is inherent to "being a UUID" — it's entirely a function of which version-specific recipe generated it.

## Core Layers Explained

### 1. UUID1 — Timestamp + Node

**What it is:** Combines the current timestamp with a node identifier (historically a MAC address) and a clock sequence to guard against clock rollbacks.

**Why it matters:** Uniqueness here comes from structure, not probability — two machines generating a UUID1 at the same instant still get different values because the node identifier differs.

**Production tip:** UUID1 embeds a real timestamp and potentially a real hardware identifier. Treat it as a mild privacy and security leak if you're exposing these IDs externally, and avoid it for anything user-facing.

### 2. UUID4 — Almost Entirely Random

**What it is:** 122 bits of pure randomness (the other 6 are fixed for version and variant) — no timestamp, no machine identifier, nothing to reverse-engineer.

```python
import uuid
uuid.uuid4()
# UUID('a3b8f9d2-1c4e-4b7a-9f2d-6e8c1a0b3d5f')
```

**Why it matters:** This is why UUID4 is the default for API tokens and idempotency keys — it reveals nothing about when or where it was created.

**Production tip:** UUID4 is an excellent choice for tokens. It's a much weaker choice as a database primary key than most people assume — more on exactly why in the primary-key section below.

### 3. UUID3 and UUID5 — Deterministic, Hash-Based

**What it is:** Not random at all. Feed in a namespace and a name, hash them (MD5 for v3, SHA-1 for v5), and the same namespace-plus-name always produces the same UUID.

```python
uuid.uuid5(uuid.NAMESPACE_DNS, "example.com")
# deterministic — identical input always yields identical output
```

**Why it matters:** This is genuinely useful when you need a stable, reproducible ID — for example, a consistent identifier derived from a URL or an external system's record ID — without needing a lookup table to check whether you've seen that input before.

**Production tip:** Prefer UUID5 over UUID3 for new code. SHA-1 (v5) is a stronger hash than MD5 (v3), and there's rarely a reason to pick the older, weaker option unless you're matching an existing system's output.

### 4. UUID6, UUID7, and UUID8 — Time-Sortable UUIDs

**What it is:** The newer versions formalized in RFC 9562. UUID7 in particular combines a Unix timestamp with random bits after it, producing IDs that are unique *and* sort naturally by creation time. UUID6 is a reordered variant of UUID1 designed for the same database-friendly sorting; UUID8 is a user-defined layout for custom schemes.

```python
# Python 3.13+
uuid.uuid7()
# UUID('01928a47-3b30-7c5e-9d1a-f0b8c4a7e923')
```

On older Python versions, the `uuid-utils` package (Rust-backed, and a drop-in for the standard library type) adds `uuid7()` support along with a meaningful speed boost even for `uuid4()`.

**Why it matters:** Time-ordered IDs cluster in a B-tree index instead of scattering randomly across it, which is the single biggest practical reason UUIDv7 is displacing v4 as the 2026 default for anything that becomes a primary key. PostgreSQL 18 ships native `uuidv7()` generation, and MySQL 8.4 added helper support — the ecosystem has caught up to the spec.

**Production tip:** If a UUID is ever going to be a primary key or a clustered index, default to v7 unless you have a specific reason not to. If it's purely an external token that never touches an index, v4 is still perfectly fine.

### 5. UUID as a Primary Key: Why Random UUIDs Hurt B-Tree Performance

**What it is:** The performance cost of using UUID4 as a primary key isn't about collisions — it's about how relational databases physically store primary keys. PostgreSQL, MySQL, and SQLite all use a B-tree structure for primary key indexes, and B-trees are optimized for values that arrive in roughly sorted order: new rows append to the rightmost page, which stays "hot" in memory and keeps writes fast and localized.

**Why it matters:** A random UUID4 lands in a random page of that index on every single insert. At scale, that means page splits, index fragmentation, and cache thrashing — the working set of "hot" pages the database needs in memory grows to cover the whole index instead of just its tail, and inserts get measurably slower and the index measurably larger than the equivalent sequential-key table. Public benchmarks on tables in the ten-million-row range have shown UUID4 inserts running roughly three times slower than auto-increment integer keys, with the resulting index around 40% larger — and that gap only widens as the table keeps growing, because the fragmentation compounds rather than staying constant.

In MySQL specifically, this cost is even sharper than it sounds, because InnoDB's primary key *is* the clustered index — every other index on the table stores a copy of the primary key value, so a bloated, fragmented primary key index doesn't just slow down primary key lookups, it inflates every secondary index built on top of it too.

**Production tip:** This is exactly the gap UUID7 closes — because it carries a timestamp prefix, it behaves like a sequential key for indexing purposes while still being generated with zero coordination across services. Switching an existing table is a schema migration, not a config flag, so budget for it as a deliberate project rather than a one-line fix.

### 6. Alternatives Worth Knowing: TSID and ULID

**What it is:** Time-sortable ID formats that aren't UUIDs at all — TSID is typically a compact 64-bit value, and ULID is a 128-bit, UUID-compatible format with a similar timestamp-plus-randomness design to UUID7.

**Why it matters:** If you want the smallest possible index footprint and don't need UUID's specific 128-bit format, TSID is worth evaluating as a 2026-era alternative. If you want to stay UUID-compatible while still gaining index locality, UUID7 or ULID both fit that need.

## End-to-End Walkthrough

Trace a new user signup through ID generation, storage, and what actually happens at insert time:

1. **Request received.** A new user signs up; the application needs to generate a primary key for the new row before or during the insert.
2. **ID generated.** With a UUID4 strategy, `uuid.uuid4()` produces a fully random 128-bit value. With a UUID7 strategy, `uuid.uuid7()` produces a value with a leading timestamp component.
3. **Row inserted.** The database's B-tree index for the primary key receives the new value. A UUID4 lands in an effectively random page of the tree; a UUID7 lands at (or very near) the rightmost, already-hot page, alongside every other recently-inserted row.
4. **Index maintenance.** Under UUID4, this insert may trigger a page split if the random landing page is full. Under UUID7, the append-mostly pattern avoids that far more often.
5. **Read path.** A lookup by primary key works identically either way — this entire difference only shows up on the write path and in overall index size, not in single-row read performance.
6. **At scale.** Multiply this by millions of rows, and the UUID4 table's index has grown larger and more fragmented than the UUID7 table's would have, purely from the insertion pattern — nothing else about the schema changed.

## Special Cases

**Idempotent, reproducible resource IDs.** When you need the same external input to always map to the same internal ID — deduplicating records pulled from an external system, for instance — UUID5 is the right tool specifically because it isn't random. Reprocessing the same input never creates a duplicate record.

**External-facing IDs vs. internal foreign keys.** A common hybrid pattern: use an internal, fast, sequential key (an integer or a TSID) for foreign key relationships and index-heavy joins, and expose a separate UUID column as the external-facing identifier. This avoids leaking sequential IDs (an IDOR risk — guessing that `/orders/1002` exists because `/orders/1001` does) without paying the full B-tree cost everywhere internally.

**Offline-first and multi-writer systems.** This is UUID's original reason for existing: any node, service, or offline client can generate a globally valid ID with zero coordination and no round-trip to a central sequence. That property doesn't go away with v7 — you get the coordination-free generation and better index behavior at the same time.

## Scaling & Production Challenges

**Index bloat compounds as tables grow.** The gap between a random-UUID-keyed table and a sequential or time-ordered one widens with table size, not just row count — fragmentation gets worse, not better, the longer a table has been taking random-order inserts.

**Migrating an existing UUID4 schema without downtime.** Add a new UUID7 column, backfill it for existing rows, dual-write both columns during a transition window, then cut reads and writes over once the backfill is verified complete — a live primary-key format change is not something to attempt as a single migration step.

**Cross-service ID generation without coordination.** In a distributed system with multiple services writing independently, any UUID version still solves the "no central authority" problem that motivated UUIDs in the first place — the version choice only changes index performance and sortability, not the coordination-free generation guarantee.

**Where real duplicate UUIDs actually come from.** The theoretical collision math (below) is essentially never the cause of a real production duplicate. Actual incidents trace back to weak or predictable random number generators, processes that fork and unintentionally share a seed, broken concurrency where shared state gets reused incorrectly across workers, or simple implementation bugs — not the underlying 128-bit space being too small.

## Code Examples

```python
import uuid

uuid.uuid4()      # random — 122 bits of entropy
uuid.uuid1()      # timestamp + node
uuid.uuid7()      # Python 3.13+: timestamp-prefixed, sortable
uuid.uuid5(uuid.NAMESPACE_DNS, "example.com")  # deterministic, SHA-1
```

For versions not yet in your Python's stdlib, or a meaningful speed boost even on `uuid4()`:

```python
# pip install uuid-utils
import uuid_utils as uuid

uuid.uuid7()  # drop-in compatible with stdlib UUID objects
```

A minimal PostgreSQL schema reflecting the v7-as-primary-key pattern:

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuidv7(),
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now()
);
```

## Common Pitfalls

**Mistake: treating "UUID" as one interchangeable thing.** Reaching for whatever `uuid()` helper is closest to hand without considering which version fits the use case. **Solution:** know whether you need randomness (v4), reproducibility (v5), or sortability (v7) before you generate anything.

**Mistake: defaulting to UUID4 for every primary key.** This is the single most common source of "why did our writes get slower as the table grew" incidents. **Solution:** use UUID7 (or a hybrid internal/external key design) for anything that becomes a clustered or primary-key index.

**Mistake: worrying about the birthday-math collision odds instead of real causes.** UUID4's collision probability is astronomically low — you'd need on the order of 2.7 quintillion generated values before even a 50% chance of one collision. **Solution:** if you do see a duplicate in production, audit your random number generator and your process/concurrency model — that's where real duplicates actually come from, not the math.

**Mistake: using UUID3 for new code.** MD5 is a weaker hash than SHA-1 with no upside for new systems. **Solution:** default to UUID5 unless you're specifically matching an existing UUID3-based system.

**Mistake: storing UUIDs as 36-character text everywhere.** This wastes storage and index space compared to the native binary representation most databases support. **Solution:** use your database's native UUID type (stored as 16 bytes) rather than a plain string column, when available.

## Production Best Practices

- **Match the version to the use case, not habit.** Random for tokens, deterministic for reproducible IDs, time-ordered for anything that becomes an index.
- **Default new primary keys to UUID7, not UUID4.** The index-locality benefit is close to free and compounds in your favor as tables grow.
- **Separate internal keys from external identifiers when it matters.** A fast internal key plus a UUID-based external identifier avoids both the IDOR risk of sequential IDs and the full indexing cost of random UUIDs everywhere.
- **Store UUIDs in their native binary type.** Sixteen bytes beats thirty-six characters at any real scale.
- **If you see a duplicate, look at your generator, not the math.** Real-world collisions come from weak randomness or broken concurrency almost every time — treat it as an implementation bug to hunt down, not evidence that UUIDs "aren't really unique."

## Wrapping Up

A UUID isn't a single thing you can reason about generically — it's a family of different bit-generation strategies, each with a specific job, and picking the wrong one for a primary key is exactly the kind of decision that looks free until your table has ten million rows in it. The math around collisions is a solved, negligible concern; the real decision that matters is version, not uniqueness. Default to v4 for tokens, v5 for reproducible IDs, and v7 for anything that becomes a primary key, and most of the "why is our database slower than it used to be" questions never come up in the first place.

Are you still defaulting to `uuid4()` for primary keys, or have you already made the switch to something time-ordered?