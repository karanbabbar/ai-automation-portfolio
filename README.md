# AI-Powered GTM Automation Portfolio

**Karan Babbar** — Product Growth · UX-Led GTM · AI Automation Builder

[LinkedIn](https://linkedin.com/in/karanbabbar) · [GitHub](https://github.com/karanbabbar/ai-automation-portfolio)

---

## Who I Am

I build growth systems that start with user experience.

Most GTM problems aren't traffic problems or tool problems — they're experience problems. Users block emails because the timing feels wrong. Listings get zero impressions because the signal architecture is broken. Outbound gets ignored because it's not relevant to where the user actually is in their journey.

I fix those problems by combining three things: **product thinking** (understanding the user journey deeply), **AI automation** (building systems that act on that understanding at scale), and **design** (making every touchpoint feel intentional, not automated).

At Superjoin, I owned GTM end-to-end — from the moment a free-trial user discovered the product, to the systems that moved them through activation, engagement, and conversion. Every project in this portfolio started with a user experience diagnosis before any automation was built.

---

## The GTM Stack I Built

```
TOFU — Discovery
  └── HubSpot Marketplace Optimization
        Problem: Zero impressions. Users couldn't find the product.
        Approach: Reverse-engineered the marketplace algorithm.
                  Keyword strategy built from search intent + volume data.
        Result:   0 → 3,600 weekly impressions. 1,112 active installs.
                  32.6% paid install rate. Top install hub: sales-hub-enterprise.

MOFU — Activation & Engagement
  └── Email Campaign 2.0
        Problem: 6–11% block rates. Users flagging emails as spam.
        Approach: Diagnosed with Mixpanel journey data.
                  Rebuilt drip around user behavior, not a fixed timer.
                  Proactive discovery over reactive follow-up.
        Result:   Block rate → ~0%. 25% open on plan comparison.
                  36% open on import success email.

BOFU — Outbound Pipeline
  └── LinkedIn Outbound AI SDR
        Problem: Free-trial users activating but no outbound follow-up.
        Approach: 13 cohorts mapped to activation behavior.
                  Autonomous n8n pipeline — enrich, qualify, send, deduplicate.
        Result:   10–20% LinkedIn reply rate.
                  35–45 Apollo API calls saved/week via deduplication.
```

---

## Projects

### 1. [LinkedIn Outbound AI SDR](./linkedin-outbound-ai-sdr/)
`n8n` `Apollo.io` `HeyReach` `Mixpanel` `Google Sheets`

An autonomous outbound system that identifies free-trial users by activation cohort, enriches them via Apollo, and engages them on LinkedIn — without human intervention. Built on the insight that outbound is most effective when it's timed to where the user is in their product journey, not when it's convenient for sales.

- 13 cohorts × 13 workflows × 13 HeyReach lists — each cohort gets messaging relevant to their activation stage
- 4-flag deduplication logic prevents re-enrichment and re-sending across runs
- ICP scoring (1–5) qualifies leads before HeyReach handoff
- Staggered nightly triggers (10:45pm–11:50pm) — built to run without supervision

**[→ Read the full breakdown](./linkedin-outbound-ai-sdr/README.md)**

---

### 2. [Email Campaign 2.0 — Free-Trial Drip Redesign](./email-campaign-2.0/)
`Mailmodo` `Mixpanel` `HubSpot` `Whimsical`

A full rebuild of Superjoin's free-trial email drip — starting from a UX diagnosis, not a copy refresh. Email 1.0 had 6–11% block rates because it sent emails based on inaction, not on where users actually were in their journey. Email 2.0 uses Mixpanel data to understand the real user timeline — and fires emails on user action OR a time threshold calibrated to average core/power user behavior.

- Hybrid trigger: event OR time, whichever comes first — no user falls through the gap
- Week 1: proactive discovery (not reactive follow-up). Week 2: trust + plan comparison. Week 3: 2 reminders, not 4
- Reassurance framing in Week 2–3: "nothing you've built will break" — addresses the UX anxiety behind trial-end blocks
- Block rate ~0% across all journeys. Phase 2 now optimizing open rates.

**[→ Read the full breakdown](./email-campaign-2.0/README.md)**

---

### 3. [HubSpot Marketplace Optimization — App Store SEO](./hubspot-marketplace-optimization/)
`HubSpot Marketplace` `Google Keyword Planner` `Google Sheets`

The Superjoin listing had zero impressions for 8 months — not because the product was weak, but because the listing experience wasn't built for how users search. I reverse-engineered the HubSpot Marketplace algorithm, separated the discovery problem from the ranking problem, and rebuilt the listing as a keyword-optimized experience designed for search intent.

- Keyword research across T1/T2/T3 tiers — search volume + user intent + competition mapping
- Weekly batch deployment — each change a controlled experiment with a measurable signal
- Feature descriptions rewritten as keyword clusters, not product marketing copy
- 0 → 3,600 weekly impressions. 1,112 active installs. sales-hub-enterprise as the #1 install hub.

**[→ Read the full breakdown](./hubspot-marketplace-optimization/README.md)**

---

## How I Work

**UX diagnosis before automation.** Block rates, zero impressions, unread emails — these are user experience signals, not just performance metrics. I start by understanding why users behave the way they do before building anything.

**Product thinking in GTM.** I map user journeys, identify drop-off points, and design touchpoints that feel like product features — not interruptions. The email reassurance framing, the cohort-matched outbound messaging, the search-intent keyword strategy — all of these are product decisions, not just growth tactics.

**AI automation as leverage.** I use AI and automation to act on user insights at scale — not to replace the thinking. The LinkedIn SDR runs every night because the logic behind it is sound. The email journeys trigger contextually because they're built on real behavioral data.

**Design as a growth input.** How something is framed matters as much as when it's sent. "Nothing you've built will break" is a design decision that directly drove the block rate drop. Keyword placement hierarchy is a design decision that drove the impressions breakthrough.

**Iterate with signal.** Weekly keyword batches. A/B email testing alongside live campaigns. Cohort-level workflow branching. Every change is designed to produce a measurable signal — not just ship and hope.

---

## What I'm Looking For

A full-time role at an early-stage or growth-stage startup where product, UX, and GTM are not separate teams. I want to own the funnel end-to-end — understand the user, build the automation, read the data, and ship the next iteration.

If you're building in PLG, B2B SaaS, or AI tools and need someone who thinks in user journeys but ships in workflows — let's talk.

📩 **karanbabbar2010@gmail.com**

---

## Stack

| Category | Tools |
|---|---|
| Workflow automation | n8n |
| Email & journeys | Mailmodo, HubSpot |
| Outbound | Apollo.io, HeyReach, LinkedIn |
| Analytics | Mixpanel, Google Analytics |
| CRM | HubSpot, Google Sheets |
| App store SEO | HubSpot Marketplace, Google Keyword Planner |
| Product & design | Whimsical, Notion, Figma |
| Dev | Git, GitHub, JSON, SQL basics |
