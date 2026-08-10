---
title: "Twilio Telephony Terminology: A Complete Guide to SIP Trunking, DIDs, Porting, and A2P Compliance"
description: "Every core Twilio and telephony term explained in context — SIP trunking, DIDs, Caller ID, number porting, 10DLC, and STIR/SHAKEN — including the 2026 A2P registration rules most guides haven't caught up to yet."
date: 2026-08-10
author: "Faizan Nadeem"
tags: ["Twilio", "Telephony", "Backend Development", "SIP"]
image: "images/blogs/twilio-telephony-terminology.jpg"
imageAlt: "Phone network infrastructure diagram representing SIP trunking and carrier connectivity"
imageCredit: "Photo by Berkeley Communications on Unsplash"
imageCreditURL: "https://unsplash.com/@berkeleycommunications"
css: "/styles/blog.css"
---

The first time you open Twilio's console to wire up voice or SMS for a real product, the terminology hits you all at once — SIP trunk, DID, CNAM, 10DLC, STIR/SHAKEN, LNP — and most of it sounds like it belongs in a networking exam, not a backend integration. The common mistake is treating this as vocabulary to memorize once and move past. It isn't. Telephony has its own physical and regulatory reality underneath the API — a real PSTN network, real carriers, and real compliance frameworks that will silently block your traffic if you don't understand them, no matter how correct your code is.

This post covers the full map: what each term actually means, how the pieces connect into one real call or message flow, and — because this is the part most terminology guides skip entirely — what changed in A2P 10DLC compliance in 2026, since a setup that worked fine in 2024 can get your messages blocked outright today.

**You'll learn:**

- What SIP trunking actually is, and how Twilio's Elastic SIP Trunking replaces physical phone lines
- The difference between a DID, a toll-free number, a short code, and a 10DLC number — and when to use each
- How number porting (LNP) actually works, and why it can take days to weeks
- How a call or message really flows through TwiML, webhooks, and your own server
- What STIR/SHAKEN and A2P 10DLC compliance require in 2026, including the FCC's new consent rule
- The mistakes that get calls flagged as spam or messages silently blocked

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

### What Twilio Telephony Actually Covers

Twilio sits between your application and two very different underlying systems: the PSTN (the actual global voice network that landlines and mobile networks use) and the carrier messaging ecosystem that routes SMS between phone numbers. Your code never touches either of those directly — it talks to Twilio's API, and Twilio acts as the carrier, translating your instructions into real signaling on real telephone infrastructure. Everything in this glossary exists to describe some piece of that translation: how a call gets connected, how a number gets identified, how a message gets approved to send, and how a call gets proven legitimate.

### Why the Terminology Is a High-Value Problem to Understand

- **Get the numbers wrong and messages don't degrade — they get blocked outright.** Since February 2025, U.S. carriers block 100% of unregistered A2P 10DLC traffic. There's no partial delivery, no warning — it just doesn't arrive.
- **Compliance frameworks change on a schedule you don't control.** STIR/SHAKEN attestation and 10DLC registration requirements have shifted multiple times since launch, most recently with the FCC's one-to-one consent rule taking effect January 27, 2026.
- **Porting mistakes cause real downtime.** Moving a number between carriers incorrectly can mean a business phone number going dark for hours or days.

## The Complete Architecture

```
Your App ──▶ Twilio API / TwiML ──▶ Twilio Platform
                                          │
                         ┌────────────────┼────────────────┐
                         ▼                                  ▼
                  SIP Trunk / PSTN                  Carrier Messaging Network
                (voice calls, real numbers)         (SMS/MMS, 10DLC/toll-free/short code)
                         │                                  │
                         ▼                                  ▼
                  Caller's Phone                      Recipient's Phone
```

The guiding principle: Twilio is your carrier, not just your API. Every term in this guide describes either how the voice side (SIP, PSTN, DIDs) or the messaging side (10DLC, A2P, short codes) gets your traffic legitimately onto that real-world network.

## Core Layers Explained

### 1. SIP Trunking & the PSTN Handoff

**What it is:** SIP (Session Initiation Protocol) is the signaling language that sets up, manages, and tears down calls over IP — the "handshake" that says "I want to call you," "call accepted," "call ended." A SIP trunk is the virtual line carrying that signaling between your system and Twilio, replacing physical copper connections. Twilio's Elastic SIP Trunking product is Twilio acting as that carrier — you connect your PBX or SBC to Twilio over SIP instead of buying physical trunk capacity. The actual audio itself flows separately, over RTP (Real-time Transport Protocol), once the SIP signaling has set the call up. Eventually, calls to real phone numbers hit the PSTN — the physical global telephone network underneath all of this.

**Why it matters:** This is the layer that turned voice infrastructure from a hardware procurement problem into a software configuration problem — trunk capacity scales through an API call instead of a vendor contract.

**Production tip:** Twilio's Elastic SIP Trunks don't support the SIP REGISTER method. You configure Twilio's SIP signaling IP addresses as trusted peers on your end rather than authenticating via registration — miss this and your trunk simply won't connect.

### 2. DIDs, Caller ID, and CNAM

**What it is:** A DID (Direct Inward Dialing) number is a real phone number that routes straight to a specific endpoint without a human operator — when you buy a number from Twilio, that's a DID. Caller ID is the number shown to whoever receives a call; CNAM (Caller ID Name) is the registered name behind that number, which you can look up on inbound calls.

**Why it matters:** DIDs are the entry point for every inbound call your system handles, and Caller ID/CNAM control what the person on the other end actually sees — get this wrong and legitimate calls look like spam before anyone even answers.

**Production tip:** Enable CNAM lookup on a trunk only for the traffic that needs it — it's a per-lookup cost, and most high-volume internal routing doesn't need a name resolved on every inbound call.

### 3. Number Porting (LNP)

**What it is:** Local Number Portability is the process of moving an existing phone number from one carrier to another — say, from a legacy carrier into Twilio — without changing the number itself.

**Why it matters:** Porting isn't an API call. It's a carrier-to-carrier process involving paperwork, account verification, and approval, and it can take anywhere from a few days to several weeks depending on the number type and losing carrier.

**Production tip:** Never cancel service with the losing carrier before a port completes. A port that fails partway through with the old service already cancelled is how a business number ends up dark.

### 4. Number Types & Formats

**What it is:** Not all numbers are the same product. Toll-free numbers (800/888-style) put the cost on the receiving party and require separate verification. Short codes are 5–6 digit numbers built for high-volume SMS, requiring lengthy carrier approval, typically used for OTPs and marketing blasts. 10DLC (10-Digit Long Code) numbers let businesses send A2P messages from a standard-looking local number, under a carrier registration framework rather than the short-code approval process. Every number Twilio's API expects is in E.164 format: `+[country code][number]`, e.g. `+923001234567`.

**Why it matters:** Picking the wrong number type means picking the wrong approval process and the wrong throughput ceiling for your use case — a short code and a 10DLC number solve overlapping problems through completely different registration paths.

**Production tip:** Normalize every phone number to E.164 at the boundary of your system — the moment it enters your database or your API calls — rather than formatting it ad hoc wherever it's used. Inconsistent formatting is a common source of silent routing failures.

### 5. Call Control: TwiML, IVR, and Webhooks

**What it is:** TwiML is Twilio's XML-based markup language that tells Twilio what to do with a call or message — `<Say>`, `<Dial>`, `<Record>`, `<Gather>`. An IVR (the "press 1 for sales" system) is built using `<Gather>` or Twilio Studio. Webhooks are HTTP callbacks Twilio makes to your server when something happens — an incoming call, an SMS, a status change — which is how your backend controls call behavior dynamically instead of relying on static TwiML.

**Why it matters:** This is the layer where your actual application logic lives. Everything else in this guide gets a call or message *to* your system; TwiML and webhooks are what your system does with it.

**Production tip:** Design your webhook endpoint to respond fast. Twilio expects a timely TwiML response to continue the call — slow application logic belongs in a background job that updates state, not inline in the webhook handler itself.

### 6. Audio & Input: Codecs and DTMF

**What it is:** A codec is the audio compression format used for a call — PCMU/PCMA for standard quality, Opus for higher quality — and matters for SIP trunk bandwidth and configuration. DTMF (Dual-Tone Multi-Frequency) is the tone generated by pressing a keypad button, which is how IVR systems capture input like "press 1 for English."

**Why it matters:** Codec mismatches between your SBC and Twilio's trunk are a common source of one-way audio or call setup failures that have nothing to do with your application code at all.

### 7. Compliance & Verification: STIR/SHAKEN and A2P 10DLC in 2026

**What it is:** STIR/SHAKEN is the U.S. regulatory framework that cryptographically signs calls to verify Caller ID legitimacy and fight spoofing, with Twilio handling attestation levels (A, B, or C) on outbound calls. A2P (Application-to-Person) messaging is any text sent from a business or app to a consumer, and when it runs over a 10DLC number, it requires Brand and Campaign registration through The Campaign Registry (TCR) before you can send anything at all.

**Why it matters:** This is the part of the stack that has moved the most, and most existing write-ups on the topic are already out of date. Since February 1, 2025, U.S. carriers block 100% of unregistered 10DLC traffic outright. On top of that, 2026 brought Authentication+ verification requirements for public companies, mandatory reseller IDs when registering on behalf of another entity, stricter EIN-age matching rules, and a requirement that opt-in consent pages stay live and carrier-verifiable at all times. Most significantly, the FCC's one-to-one consent rule took effect January 27, 2026, tightening what counts as valid consent for A2P messaging considerably beyond the older shared-consent model many businesses had been relying on.

**Production tip:** If you registered a 10DLC campaign in 2023 or 2024 and haven't revisited it, don't assume it's still compliant — re-verify your opt-in flow and consent language against the current rules rather than assuming a past approval still holds.

## End-to-End Walkthrough

Trace a real inbound call, from the caller's phone to your application logic:

1. **Caller dials your DID.** The call enters the PSTN and gets routed toward Twilio, which owns that number on your behalf.
2. **Twilio receives the call and checks your configuration.** If the number is configured for voice, Twilio sends an HTTP webhook to your server announcing the incoming call.
3. **Your server responds with TwiML.** A `<Gather>` verb plays a greeting and waits for DTMF input — the IVR menu.
4. **Caller presses a key.** The DTMF tone is captured and sent back to your webhook as form data in a follow-up request.
5. **Your application logic decides the route.** Based on the digit pressed, time of day, or caller ID, your server returns new TwiML — either `<Dial>` to connect the caller to an agent over a SIP trunk, or further IVR prompts.
6. **The call connects.** SIP signaling sets up the session between Twilio and the destination (an agent's SIP endpoint, another PSTN number, or a queue); RTP carries the actual audio once it's established.
7. **STIR/SHAKEN attestation travels with the call.** If this is an outbound leg, Twilio attaches an attestation level based on how verified the calling number and account are, which downstream carriers use to decide whether to display it as "Verified" or risk flagging it as spam.
8. **Call ends.** A final status webhook fires to your server, letting you log duration, cost, and outcome.

## Special Cases

**A2P messaging from AI or conversational agents.** If an LLM-driven system is generating outbound message content dynamically, carriers still expect the registered use case to match what's actually being sent. Registering a campaign as "customer care" and then sending anything that drifts into promotional territory is exactly the kind of use-case drift that gets a campaign flagged, regardless of how the message was generated.

**International SIP trunking.** Regulatory requirements, number formats, and even which codecs are commonly supported vary by country — a trunking setup that works cleanly in the U.S. often needs real reconfiguration, not just a new number, to work correctly elsewhere.

**Porting toll-free vs. local numbers.** These follow different processes and timelines with different carrier requirements — don't assume a toll-free port and a local DID port behave the same way operationally.

## Scaling & Production Challenges

**Concurrent call and CPS limits.** New Twilio accounts without an approved Business Profile have restricted concurrent call limits and can't self-serve calls-per-second increases — a launch that assumes unlimited scale from day one will hit this wall.

**Regional SIP trunks and data residency.** Call data for a trunk is processed and stored within whichever Twilio Region that trunk is configured for — a trunk is only active in one region at a time, which matters directly for data residency requirements in regulated industries.

**Trust score erosion at scale on 10DLC.** Carriers track sending patterns per campaign, and use-case drift or high complaint rates degrade a campaign's throughput and deliverability over time, not just at registration — compliance is an ongoing operational concern, not a one-time approval.

**STIR/SHAKEN attestation at volume.** As outbound call volume grows, inconsistent attestation levels across your number pool can mean some calls display as verified and others get flagged as potential spam, purely based on account and number history rather than anything about the call itself.

## Code Examples

Formatting and validating phone numbers to E.164 before they ever reach the Twilio API:

```python
import re

def to_e164(raw_number: str, default_country_code: str = "92") -> str:
    digits = re.sub(r"\D", "", raw_number)
    if raw_number.startswith("+"):
        return f"+{digits}"
    return f"+{default_country_code}{digits.lstrip('0')}"
```

A minimal TwiML response for an inbound-call IVR menu:

```xml
<Response>
    <Gather numDigits="1" action="/voice/menu" method="POST">
        <Say>Press 1 for sales, press 2 for support.</Say>
    </Gather>
</Response>
```

A FastAPI webhook handler receiving that DTMF input:

```python
from fastapi import FastAPI, Form
from fastapi.responses import Response

app = FastAPI()

@app.post("/voice/menu")
async def handle_menu(Digits: str = Form(...)):
    destination = "+15551234567" if Digits == "1" else "+15559876543"
    twiml = f'<Response><Dial>{destination}</Dial></Response>'
    return Response(content=twiml, media_type="application/xml")
```

## Common Pitfalls

**Mistake: assuming a purchased number can send A2P SMS immediately.** A 10DLC number with no registered Brand and Campaign will have its messages blocked outright, not delayed. **Solution:** complete Campaign Registry registration before sending anything at production volume, and treat it as a prerequisite, not a formality.

**Mistake: canceling the old carrier before a port completes.** This is the single most common cause of a number going dark during a migration. **Solution:** keep the losing carrier's service active until the port is fully confirmed on Twilio's side.

**Mistake: treating short codes and 10DLC as interchangeable.** They solve overlapping problems through entirely different approval processes and cost structures. **Solution:** pick short code for high-volume marketing blasts needing maximum throughput, and 10DLC for standard application messaging from a local-looking number.

**Mistake: formatting phone numbers inconsistently across the codebase.** Free-text or locale-specific formatting causes silent routing and lookup failures. **Solution:** normalize to E.164 once, at the system boundary, and validate before storage.

**Mistake: assuming a 2023 or 2024 A2P registration is still fully compliant.** Authentication+, reseller ID, and consent rules have all changed since. **Solution:** re-audit existing campaigns against current requirements, especially the FCC's one-to-one consent rule.

## Production Best Practices

- **Register A2P campaigns before you need them, not when messages start getting blocked.** Approval isn't instant, and unregistered traffic is blocked outright, not queued.
- **Keep opt-in consent pages live and accurate.** Carriers can re-verify them at any time, and a stale or missing opt-in flow can suspend an otherwise-approved campaign.
- **Normalize phone numbers to E.164 at the boundary.** Every downstream API call and lookup depends on this being consistent.
- **Monitor STIR/SHAKEN attestation and 10DLC trust scores as ongoing metrics**, not one-time setup checks — both can degrade based on sending patterns over time.
- **Never let a port happen without a fallback plan.** Confirm the new setup is live and tested before releasing the old carrier.

## Wrapping Up

None of this terminology is complicated in isolation — the challenge is that it spans a physical network, a signaling protocol, a numbering system, and a regulatory framework that keeps moving, and most integration guides only cover the first two. The compliance side in particular has changed more in the past eighteen months than the technical side has, and a setup that was fully compliant in 2024 can be actively blocking your own traffic today without a single error in your code.

What part of the Twilio stack gave you the most trouble when you first set it up — the SIP trunking side, or the compliance side?