---
title: "MCP Servers Explained: A Complete Guide to What I Learned Deploying One to AWS EC2"
description: "A hands-on walkthrough of deploying an MCP server to EC2 over streamable HTTP with nginx and bearer auth — and what the July 2026 stateless spec rewrite means for that setup."
date: 2026-07-26
author: "Faizan Nadeem"
tags: ["MCP", "AI Agent Tools", "Backend Development", "AWS"]
image: "images/blogs/mcp-server-ec2-deployment.jpg"
imageAlt: "Server infrastructure diagram representing a deployed API endpoint behind a reverse proxy"
imageCredit: "Photo by Google Deep Mind on Unsplash"
imageCreditURL: "https://unsplash.com/@googledeepmind"
css: "/styles/blog.css"
---

## What's Inside

Most MCP content is theoretical — diagrams of a client talking to a server, an analogy to USB-C, a five-line "hello world" that runs on localhost and never leaves your laptop. None of that prepares you for the actual work: taking a tool you built locally and putting it on a real server, behind a real reverse proxy, reachable by a real client over the public internet. That's a different problem, and it's the one nobody writes about.

I deployed one. A local MCP server, moved to an EC2 instance, served over the streamable HTTP transport with bearer-token auth in front of it via nginx. It worked. And then, almost as soon as I'd settled on that setup, the protocol itself changed underneath it — the MCP specification's biggest revision since launch went from release candidate to final on July 28, 2026, and it rewrites the core of the protocol to be stateless. Most people treat "add MCP support" as a single check on a list. It isn't. It's a transport decision, an auth decision, and now — with this spec update — a scaling decision, and getting any one of them wrong is the difference between a tool that works in a demo and one you can actually trust in production.

**You'll learn:**

- What MCP actually standardizes, and what it deliberately leaves up to you
- How the streamable HTTP transport works and why it replaced the older SSE-based approach
- How I exposed a local MCP server on EC2 with nginx and bearer-token auth
- What the July 28, 2026 stateless rewrite changes at the protocol level
- Why a static bearer token clears the bar for "has auth" but not for "spec-compliant auth"
- What I actually had to change in my own deployment to keep up with the update

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

### What MCP Actually Covers

The Model Context Protocol standardizes one specific thing: how an AI application discovers and calls tools, and reads resources, exposed by a separate server process — using a defined message format (JSON-RPC) over a defined transport. It doesn't tell you how to write your tool logic, how to design your agent, or how to deploy anything. It's a connection standard, not a framework. Before it existed, every AI application that wanted to talk to, say, your internal API needed a custom, one-off integration. MCP means you build the server once and any compliant client can talk to it.

That framing matters because it's also exactly why deployment is the hard part. The protocol gives you a contract for how messages are structured — it says nothing about whether your server is reachable only from your laptop or from the internet, whether it's protected by anything, or whether it can survive more than one client hitting it at once. All of that is on you.

### Why Deployment Details Are a High-Value Problem

- **A locally-running MCP server has no real attack surface. A publicly deployed one does.** The moment your server has a public URL, it's an authenticated API like any other, with everything that implies about token handling and input validation.
- **Session and scaling behavior weren't obvious under the older spec.** Getting this wrong meant either a server that silently broke under concurrent load or one over-engineered with sticky sessions it didn't need.
- **The protocol itself is still moving.** A deployment built against assumptions from six months ago can quietly stop being spec-compliant without throwing a single error — clients just start negotiating down to an older protocol version instead.

## The Complete Architecture

```
AI Client ── HTTPS POST ──▶ nginx (TLS termination, auth check, reverse proxy)
                                    │
                                    ▼
                          MCP Server Process
                          (Streamable HTTP transport)
                                    │
                       ┌────────────┴────────────┐
                    Tools                    Resources
```

The guiding principle: once your MCP server has a public URL, treat it exactly like any other authenticated API you'd expose from EC2 — because structurally, that's what it is. The "MCP" part is the message format riding on top; the security and scaling considerations underneath are the same ones you'd apply to any backend service.

## Core Layers Explained

### 1. Transport: Streamable HTTP

**What it is:** A single HTTP endpoint that accepts POST requests carrying JSON-RPC messages. It replaced the earlier HTTP+SSE transport, which relied on a persistent server-sent-events stream alongside regular HTTP calls.

**Why it matters:** A single POST endpoint is something every reverse proxy, load balancer, and API gateway already knows how to handle. You're not fighting your infrastructure to keep a long-lived stream alive through nginx or an ALB — it behaves like a normal HTTP API.

```nginx
location /mcp {
    proxy_pass http://127.0.0.1:8000;
    proxy_set_header Host $host;
    proxy_set_header Authorization $http_authorization;
    proxy_http_version 1.1;
    proxy_read_timeout 300s;
}
```

**Production tip:** Even with a single-endpoint transport, keep your proxy read timeout generous. Individual tool calls can legitimately take longer than a typical API request, and a timeout that's too tight will kill valid long-running calls, not just hung connections.

### 2. Authentication: Bearer Tokens (a starting point, not a destination)

**What it is:** A static token in the `Authorization` header, checked before a request is allowed to reach the MCP server process — either at the nginx layer or in application middleware.

**Why it matters:** It's the minimum viable security for anything with a public URL, and it's exactly where I started. But a static bearer token has no expiry semantics, no binding to a specific server, and no protection if it leaks into a log somewhere. It answers "is this request authenticated at all," not much else.

```python
from fastapi import Request, HTTPException

VALID_TOKEN = "your-rotated-secret-token"

async def verify_bearer(request: Request):
    auth = request.headers.get("authorization", "")
    if auth != f"Bearer {VALID_TOKEN}":
        raise HTTPException(status_code=401, detail="Unauthorized")
```

**Production tip:** Never let the raw `Authorization` header hit your access logs. It's an easy thing to miss in a default nginx or app log format, and it turns a log file into a credential leak.

### 3. Statelessness — the Core of the July 2026 Rewrite

**What it is:** The 2026-07-28 specification removes protocol-level sessions entirely. There's no more `Mcp-Session-Id` header, and the transport drops the long-held GET stream endpoint that earlier versions used to keep a client and server in sync across a conversation. Every request is now self-contained.

**Why it matters:** Under the older model, a server often needed to remember something about a client between calls, which is exactly the kind of state that forces you into sticky sessions once you have more than one server instance behind a load balancer. Statelessness at the protocol level means any instance can legitimately handle any request — real horizontal scaling, without pinning a client to a specific box.

**Production tip:** If your server was relying on in-memory state to remember anything about a client between calls, that pattern breaks under the new spec. Push whatever state matters into the request payload itself or into an external store like Redis that any instance can read.

### 4. Multi Round-Trip Requests (asking the user something mid-call)

**What it is:** A stateless protocol still needs a way for a server to ask a client for input in the middle of a tool call — a confirmation, a missing parameter. The new spec handles this with what it calls Multi Round-Trip Requests: instead of holding a stream open, the server returns an `InputRequiredResult` containing the questions and an opaque `requestState` blob. The client collects the answers and reissues the original call with both the answers and the echoed state attached.

**Why it matters:** This is the piece that makes real statelessness workable. Because everything the server needs to resume is contained in the payload the client sends back, any server instance — not necessarily the one that started the call — can pick up the retry and finish it.

**Production tip:** Treat `requestState` as opaque on the client side. Don't inspect or modify it — just store it and echo it back exactly as received, or you risk breaking a resume that depends on it being intact.

### 5. Authorization Hardening: OAuth 2.1, PKCE, and Resource Indicators

**What it is:** The updated spec pushes remote MCP servers toward OAuth 2.1 with PKCE for authorization, and requires resource indicators (per RFC 8707) that bind an issued token to the specific server it was meant for.

**Why it matters:** A static bearer token can be replayed against any server that happens to accept it, if it ever leaks. Binding a token to a specific resource closes exactly that gap — a token issued for one MCP server is no longer usable against a different one, even if both happen to trust the same identity provider.

**Production tip:** A bearer-token setup like the one most of us started with satisfies "this endpoint requires authentication." It does not satisfy the interoperability and replay-protection bar the new spec sets for a properly compliant remote server, and that gap is worth closing before you hand the URL to anyone outside your own team.

### 6. Deprecation Policy and the Extensions Framework

**What it is:** The spec now moves capabilities through a formal Active → Deprecated → Removed lifecycle, with a minimum twelve-month gap between deprecation and removal. New capabilities — things like server-rendered UI or long-running task support — ship first as opt-in Extensions rather than landing directly in the protocol core. A handful of older capabilities, including roots, sampling, and logging as originally specified, are moving through this deprecation path in favor of their replacements.

**Why it matters:** It means a working integration won't get broken overnight by a future revision, and it means you can adopt new capabilities deliberately instead of being forced to keep pace with everything the spec adds.

## End-to-End Walkthrough

Trace the actual path from local prototype to a spec-current EC2 deployment:

1. **Local testing.** The MCP server runs over stdio against a local client — no network, no auth, fast iteration.
2. **Move to streamable HTTP.** The server is reconfigured to serve the streamable HTTP transport instead of stdio, and deployed to an EC2 instance.
3. **nginx in front.** TLS termination happens at nginx, which also checks the bearer token before anything reaches the MCP process, then reverse-proxies the request through.
4. **A normal tool call.** The client sends a JSON-RPC request as a single POST; the server processes it and returns a result in the same request-response cycle.
5. **Protocol version negotiation.** With the 2026-07-28 spec now final, the client and server negotiate which protocol revision to use. A client that hasn't upgraded yet negotiates down to 2025-11-25 automatically rather than failing outright.
6. **A tool needs more input.** Instead of holding the connection open, the server returns an `InputRequiredResult` with the questions it needs answered.
7. **The client resumes.** It reissues the call with the answers and the echoed `requestState`. Because nothing about that resume depends on hitting the same server instance, it can land on any box behind nginx.
8. **Auth failure path.** A missing or invalid token is rejected at the nginx layer, before it ever reaches the MCP process — the application code never even sees a malformed or unauthenticated request.

## Special Cases

**Long-running tools.** Anything that used to lean on a held-open stream for a slow operation should move to the Tasks extension pattern rather than trying to recreate a persistent connection on top of a transport that no longer supports one.

**Tools with no plan to go remote.** If a server only ever needs to run locally alongside its client, stdio transport is still the simplest option, and almost none of the deployment concerns here apply.

**Mixed-version clients during the transition.** Since older clients fall back to 2025-11-25 automatically, it's realistic to run a single deployment that correctly serves both protocol revisions for a while, rather than forcing every client to upgrade in lockstep with the server.

## Scaling & Production Challenges

**Multiple instances behind a load balancer.** Under the old session-based model, this meant sticky sessions or shared session storage so a client stayed bound to the instance that knew about it. Statelessness removes that requirement outright — any instance can serve any request, which is the actual unlock behind this rewrite for anyone running more than one box.

**TLS and certificate management as instance count grows.** Managing a certificate per EC2 instance doesn't scale well; putting a managed load balancer with an ACM-backed certificate in front of a fleet of instances is far less operational overhead than per-instance nginx TLS configs.

**Token issuance and rotation at scale.** Static bearer tokens don't just have a security ceiling — they're an operational headache once you have more than a couple of clients, since every rotation is a manual coordination problem. Moving to OAuth 2.1-based issuance solves the security gap and the rotation headache in the same move.

**Tracing multi-call interactions without server-side session state.** With no session tying a sequence of calls together on the server, correlation IDs passed through the request payload become the only reliable way to reconstruct what a single agent interaction actually did across multiple calls.

## Code Examples

The nginx config and bearer-check middleware above cover the transport and baseline auth layers. Here's a minimal pattern for handling the resume side of a Multi Round-Trip Request on the client:

```python
async def call_tool_with_resume(client, tool_name, params):
    result = await client.call_tool(tool_name, params)
    if result.get("type") == "InputRequiredResult":
        answers = collect_answers(result["questions"])
        return await client.call_tool(
            tool_name,
            {**params, "answers": answers, "requestState": result["requestState"]},
        )
    return result
```

The important detail is the last line: `requestState` is passed straight through, untouched, exactly as the server sent it.

## Common Pitfalls

**Mistake: treating "it's deployed" as "it's production-ready."** Getting nginx and a bearer token in front of an MCP server clears the lowest bar, not the whole bar. **Solution:** hold your MCP endpoint to the same standard you'd hold any other public API — proper auth, logging discipline, and rate limiting before you share the URL widely.

**Mistake: relying on server memory between calls.** Some early deployments quietly depended on the server remembering something about a client from a previous request. **Solution:** treat every request as if it could land on a different instance than the last one, because under the current spec, it can.

**Mistake: hardcoding a single protocol version in the client.** A client pinned to one exact revision breaks the moment a server upgrades. **Solution:** implement version negotiation with a sane fallback, so an upgrade on one side doesn't take down the other.

**Mistake: underestimating the auth bar for a "just internal" tool.** Internal-only today often becomes shared-with-a-partner-team tomorrow, at which point a static token stops being enough. **Solution:** build toward OAuth 2.1 and resource indicators from the start if there's any realistic chance the server leaves your own laptop or VPC.

## Production Best Practices

- **Don't hold state your infrastructure can't scale.** If a client needs something remembered between calls, put it in the payload or an external store — not server memory.
- **Bind tokens to the resource they're for.** A token that works against any server you happen to trust is a token that will eventually get used somewhere you didn't intend.
- **Negotiate protocol versions, don't assume one.** Your clients and server won't always upgrade on the same day — build for that gap deliberately.
- **Keep auth checks at the edge.** Rejecting a bad request at nginx, before it reaches your application code, keeps your MCP process simpler and your attack surface smaller.
- **Treat every opaque protocol field as opaque.** `requestState` and similar fields exist so the server can trust what comes back — don't inspect or mutate them client-side.

## Wrapping Up

The theory around MCP is easy; the deployment is where the real decisions live, and those decisions just changed underneath everyone with the July 28, 2026 spec update. Streamable HTTP got simpler to run behind standard infrastructure, statelessness turned "add another EC2 instance" into an actual option instead of a session-management project, and the authorization model finally caught up to what a public-facing server needs. None of it is complicated once you've seen it — it's just not the part any of the theoretical write-ups cover.

If you've deployed an MCP server anywhere other than localhost, what did you have to change once real clients and real load showed up?