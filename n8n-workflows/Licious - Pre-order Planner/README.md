# Licious AI Protein Meal Planner

An AI-powered meal planning assistant built for [Licious](https://www.licious.in/) — India's leading meat and seafood brand. The app helps users plan their weekly protein intake based on personal health goals and generates an optimised Licious grocery list — built for zero wastage, cost efficiency, and minimal variety complexity.

**Live App:** [meal-prep-guide-3.preview.emergentagent.com](https://meal-prep-guide-3.preview.emergentagent.com)

---

## What It Does

1. **Onboards the user** — collects age, weight, height, goal (weight loss / gain / muscle), meals per day preference, and protein preference (meat, chicken, eggs)
2. **Calculates daily calorie and protein targets** — personalised to each user's body metrics and goal
3. **Plans a weekly meal schedule** — distributes protein across meals with zero wastage using a custom algorithm
4. **Generates a Licious grocery list** — selects the right product SKUs, pack sizes, and quantities optimised for cost and portion efficiency

---

## Tech Stack

| Layer | Tool |
|---|---|
| Frontend | React + Tailwind CSS (built on Emergent) |
| Backend | Python (FastAPI) |
| Workflow & AI | n8n + Claude API (Anthropic) |
| Database | Supabase (PostgreSQL) |
| Product Data | Scraped from Licious via Apify, stored in Supabase |

---

## The Hard Problem: Meal Planning Is a Portion Optimisation Problem

Most meal planners stop at "here's your daily protein target." The real challenge is what comes after — how do you actually buy and distribute protein across a week with real pack sizes, real cuts, and real constraints?

This is the core algorithm the planner solves:

### Step 1: Protein Distribution Across Meals

The planner first determines how protein should be distributed across the day based on the user's meal preference (3 or 4 meals). It doesn't split protein equally — it weights meals from heaviest to lightest, so a user eating 3 meals gets a different distribution than one eating 4.

### Step 2: Source and Cut Selection

Once distribution is set, the user selects which protein source goes in which meal — e.g. eggs for breakfast, chicken for lunch, meat for dinner. The planner restricts options to **4 SKUs per cut per protein source** to keep choices manageable and reduce analysis paralysis.

### Step 3: Portion Utilisation — the Nuanced Part

This is where the algorithm does its real work. The goal is to minimise waste and cost by buying in bulk and utilising portions efficiently:

**Same-day distribution:** If a protein source appears in 2 or more meals in a day (e.g. eggs at breakfast and chicken at lunch), the planner presents the user with a choice of cut for each meal slot — keeping variety intentional, not accidental.

**Cross-meal cut nudging:** If a user prefers chicken breast at lunch and chicken keema at dinner but doesn't want the same cut twice in a day, the planner nudges utilisation of the remaining portion the next day — rather than buying two separate smaller packs.

**Bulk-first logic:** The algorithm always calculates the weekly volume required for each protein source first, then selects the largest pack size that covers it. Smaller packs are only used to top up the remainder.

**Shortage handling:** If there is a shortage of 50g or more after pack selection, the planner automatically increases the preferred protein source to cover the gap — no user intervention needed.

**Surplus handling:** If there is a surplus of 50g or more, the planner adjusts by increasing that meal's portion slightly and reducing other sources proportionally — keeping the daily and weekly protein macro on target.

### Step 4: Weekly Extrapolation

The first day's meal plan is extrapolated across the full week. The user can then make changes day by day, or chat with the planner to iterate — the agent applies the same rules above when making any adjustments.

### Step 5: Grocery List Generation

Since Licious has not published a public cart API, the planner outputs a structured grocery list — protein source, cut type, pack size, quantity, and estimated cost — that the user takes to the Licious website to order.

The grocery list is optimised by the same rules: prioritise large portions, limit cut variety in sync with user preferences, and minimise cost while keeping macros accurate.

---

## Architecture

```
User (Web App)
     │
     ▼
Frontend (React / Emergent)
     │  Structured form inputs
     ▼
n8n Workflow — Meal Planner State Engine
     │
     ├── Calorie & macro calculator (deterministic logic)
     ├── Meal distribution algorithm (coded rules)
     │     ├── Portion utilisation engine
     │     ├── Shortage / surplus adjustment
     │     └── Bulk-first pack selection
     ├── Claude API — handles edge cases & ambiguous inputs
     └── Supabase — fetches Licious product catalogue
          │
          ▼
     Weekly plan + grocery list → Frontend
```

---

## The Build Journey: From Multi-Agent to State Engine

### V1 — Multi-Agent Architecture (did not work well)

The first version used **3 handoff agents**:
- **Agent 1** — Onboarding (chat-based Q&A)
- **Agent 2** — Meal planner
- **Agent 3** — Weekly cart builder

**Why it failed:**

**Unstructured user inputs broke the pipeline.** Users typed answers in a freeform chat — height in cm or inches, weight in kg or pounds, abbreviations like "H" or "W". The agents couldn't normalise this reliably, causing inconsistent outputs and occasional hallucinations.

**Agent handoffs lost context.** When Agent 1 passed user data to Agent 2, nuanced details were dropped or misinterpreted. Each agent would often revert to generic outputs rather than personalised ones.

**The portion optimisation problem was too complex for a general agent.** Even with detailed instructions, guardrails, and examples — the agent couldn't consistently handle bulk-first selection, shortage/surplus logic, and cross-meal distribution simultaneously. It needed a deterministic algorithm, not a prompt.

**Poor UX.** Freeform chat requires users to type everything. Response latency was high. Progressive disclosure was impossible — users had no sense of how many steps remained.

---

### V2 — State Engine Architecture (current)

**Key insight: most of the onboarding is deterministic. There is no need for an agent to control it.**

Age, weight, height, gender, goal — these are fixed-input fields. A structured multi-step form handles them better, faster, and more reliably than a chatbot. Claude is only invoked when the input is genuinely ambiguous or edge-case.

**The portion optimisation problem was solved with a coded algorithm** — hard rules for distribution weighting, pack selection, shortage/surplus adjustment, and cross-meal nudging. This made the output reliable, fast, and cost-efficient (significantly fewer tokens consumed).

**Results:**
- Consistent, personalised outputs for every user
- Significantly faster response time
- Zero hallucinations on structured inputs
- Better UX — progressive disclosure, visual weekly plan output

---

## What I Learned

> The best AI systems know when *not* to use AI.

Using an agent for deterministic tasks wastes tokens, adds latency, and introduces unpredictability. The right pattern is: **code what you can predict, use AI only for what you genuinely can't.**

The portion optimisation problem also taught me that agents are not good optimisers out of the box. If you need consistent rule-following across multiple interdependent constraints — write the rules in code, not in a prompt.

---

## Repository Structure

```
n8n-workflows/
└── Licious - Pre-order Planner/
    └── Meal Planner State Engine v1.json   ← current production workflow
```

**Frontend + backend code:** [github.com/karanbabbar/licious-ai-meal-planner](https://github.com/karanbabbar/licious-ai-meal-planner)

---

## How to Import the n8n Workflow

1. Download `Meal Planner State Engine v1.json`
2. Open your n8n instance
3. Click **"Import from file"** and select the JSON
4. Add your credentials: Anthropic API key, Supabase URL + key
5. Activate the workflow

---

*Built by [Karan Babbar](https://github.com/karanbabbar) — Growth PM & AI builder*