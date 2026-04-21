# Licious Meal Planner — n8n Workflow

## What This Workflow Is

This is the entire backend of the Licious AI Protein Meal Planner. It is a single n8n workflow that replaced what was originally a 3-agent architecture. It handles all business logic, state management, algorithm execution, database reads and writes, and AI calls — with no traditional API server doing any of the heavy lifting.

The workflow runs at `POST /v2/meal-planning` and is called by the React frontend at each step of the user journey.

---

## The Core Architectural Decision

Most AI-powered apps default to agents for everything. This workflow makes a deliberate opposite choice:

> **Use deterministic code for everything predictable. Use Claude only for what genuinely requires language understanding.**

In practice, Claude is called in exactly one scenario in this entire workflow — when a user types a custom protein distribution in freeform text that needs to be parsed into numbers. Every other step runs as pure JavaScript logic inside n8n's Code node.

This decision came from the failure of v1, where 3 agents handled the full flow. The agents hallucinated on structured inputs, broke on handoffs, and couldn't solve the portion optimisation problem reliably even with detailed prompts and guardrails.

---

## Workflow Nodes — What Each One Does

```
Webhook (POST /v2/meal-planning)
     │
     ▼
Get User & Products (Postgres)
     │  Single query — joins users, sessions, products, stock_tracker
     ▼
State Engine (JavaScript Code Node — ~800 lines)
     │  All business logic lives here
     ▼
Needs Claude? (IF node)
     │
     ├── YES → Call Claude NLP (HTTP Request to Anthropic API)
     │              └── Returns parsed JSON back into flow
     │
     └── NO ──────────────────────────────────┐
                                              ▼
                                     Update History (Postgres)
                                     Saves conversation + state
                                              │
                                              ▼
                                     Stage Complete? (IF node)
                                              │
                              ┌───────────────┴──────────────┐
                              YES                            NO
                              │                              │
                         Save Plan                       Respond
                         (Postgres)                  (mid-flow JSON)
                              │
                         Respond Complete
                         (final JSON)
```

---

## The State Engine — How It Works

The State Engine is a single JavaScript Code node (~800 lines) that acts as a state machine. Every request from the frontend hits this node. It reads the current state from Supabase, determines what step the user is on, processes their input, runs the relevant logic, and returns the next UI state.

### State Object

Each session carries a persistent state object stored in Supabase and passed into every execution:

```javascript
state = {
  engine_version: 1,
  step: 'budget_setup',        // current major step
  sub_step: 'init',            // current sub-step within major step
  supplement_g: 0,             // protein from supplements (deducted)
  original_protein: 180,       // total daily protein target
  protein_target: 180,         // food-only protein target
  meals_per_day: 3,
  distribution: null,          // { breakfast: 60, lunch: 60, dinner: 60 }
  distribution_label: null,    // 'Equal' | 'Heavy Breakfast' etc.
  meals: [],                   // meal objects with products, cuts, portions
  current_meal: 0,
  carried_portions: {},        // leftover protein carried to next meal
  running_price: 0,            // daily cost accumulator
  weekly_plan: null,
  cart: null,
  stage_complete: false
}
```

### Steps the State Machine Walks Through

**1. `budget_setup`**
- Asks if user takes protein supplements
- Deducts supplement grams from daily protein target
- Presents 4 distribution options: Equal, Heavy Breakfast, Heavy Lunch, Heavy Dinner
- If user types a custom split → flags `needs_claude: true` → Claude parses it into `{breakfast, lunch, dinner}` numbers

**2. `meal_curation` (runs once per meal)**
- `source_select` — user picks protein sources (eggs, chicken, fish, mutton). Max 3 sources per meal.
- `cut_select` — for non-egg sources, user picks cut type (Boneless, Curry Cut, Keema etc.)
- `product_select` — workflow queries Supabase for up to 4 SKUs per cut per source. User picks one per source.
- `portion_confirm` — workflow calculates portion sizes, shows remaining pack after use, presents utilisation options

**3. `consolidation`**
- After all meals are planned, checks if any product appears in 2+ meals
- If yes, checks if a larger pack of the same product exists that would be cheaper per gram and cover the combined volume
- Surfaces savings opportunities to the user

**4. `weekly_plan`**
- Extrapolates day 1 meal plan across 7 days
- Calculates total packs needed per product based on pack size and days covered per pack
- Builds the cart object: product, pack size, quantity, price, product URL

**5. `delivery`**
- Collects delivery frequency preference (daily, every 2 days, every 3 days, weekly)
- Collects time slot preference
- Sets `stage_complete: true` → triggers Save Plan node

---

## The Portion Optimisation Algorithm

This is the core of the planner. The algorithm runs inside the State Engine and solves the real meal planning problem: given a protein target, a set of products with fixed pack sizes, and a 7-day horizon — how do you buy and distribute protein with zero wastage and minimum cost?

### Protein Distribution

```javascript
function calcDistribution(target, label) {
  if (label === 'Equal') {
    const base = Math.floor(target / 3);
    return { breakfast: base, lunch: base, dinner: target - (base * 2) };
  }
  if (label === 'Heavy Breakfast') {
    return { breakfast: Math.round(target * 0.4), 
             lunch: Math.round(target * 0.3), 
             dinner: target - ... };
  }
  // Heavy Lunch: 30/40/30, Heavy Dinner: 30/30/40
}
```

### Portion Calculation

For meat and fish — converts protein grams needed into weight, rounded to nearest 5g:
```javascript
let grams = roundTo5((proteinNeeded / protein_per_100g) * 100);
if (grams < 50) grams = 50; // minimum portion floor
```

For eggs — calculates egg count at 6.5g protein per egg:
```javascript
const eggsNeeded = Math.ceil(proteinNeeded / 6.5);
```

### Pack Utilisation Logic

After portion is calculated, the algorithm checks how much pack is left over:

```javascript
const remaining = pack_size_grams - portion_grams;

// How many days does one pack cover at this portion size?
const totalDays = Math.floor(pack_size / portion_grams);

// Options presented to user:
// 1. Same meal for N days (pack covers multiple days — no waste)
// 2. Use remaining Xg for a different meal today (carry to next meal)
// 3. Switch to smaller pack if one exists that still covers the portion
```

When a user chooses "use remaining for a different meal", the leftover protein is calculated and stored in `carried_portions` — injected into the next meal's effective target automatically.

### Shortage and Surplus Handling

If a product covers fewer days than needed:
- Weekly grams required = daily portion × 7
- Packs needed = `Math.ceil(weekly_grams / pack_size_grams)`

If a product is used in multiple meals per day:
```javascript
// Pack usage is accumulated across meals, not double-counted
if (cartMap[product_name]) {
  existing.packs_needed += packsNeeded;
  existing.total_price = existing.packs_needed * price_per_pack;
}
```

### Consolidation Check

```javascript
// If same product appears in 2+ meals and a larger pack exists:
const biggerPerGram = bigger.price / bigger.pack_size_grams;
const currentPerGram = prod.price / prod.pack_size_grams;

if (biggerPerGram < currentPerGram && bigger.pack_size_grams >= totalGrams) {
  const savings = Math.round((prod.price * usage.meals.length) - bigger.price);
  // Surface to user as a consolidation suggestion
}
```

---

## How Claude Is Used (and Why It Is Barely Used)

Claude is called via HTTP Request node directly to `api.anthropic.com/v1/messages`. It is invoked only when `needs_claude: true` is set by the State Engine.

**The only scenario where this happens:**
A user types a custom protein distribution like "I want 50g at breakfast, 80g at lunch and 50g at dinner." The State Engine can't parse this reliably with regex, so it flags Claude to handle it.

**Claude's prompt:**
```
Parse the user's desired distribution. Respond ONLY with JSON:
{"breakfast": number, "lunch": number, "dinner": number}
Numbers must sum to [protein_target].
```

**System prompt:**
```
You are a helper that parses user input for a meal planning app. 
Respond ONLY in valid JSON, no markdown, no code fences.
```

Claude returns a clean JSON object. The State Engine picks it up on the next execution and continues the flow as normal.

Everything else — supplement handling, distribution calculation, cut selection, product filtering, portion sizing, pack utilisation, consolidation, cart building — is pure JavaScript. No AI.

---

## How the Frontend Calls This Workflow

The frontend sends a POST request to the webhook at each step:

```json
{
  "session_id": "abc123",
  "message": { "sources": ["chicken", "eggs"] }
}
```

The `message` field can be a JSON object (structured UI selection) or a plain string (freeform text). The State Engine handles both — it tries to parse JSON first, falls back to text parsing.

The workflow responds with:

```json
{
  "message": "Great! What type of chicken do you prefer?",
  "ui_type": "cut_select",
  "ui_data": { "category": "chicken", "cuts": ["Boneless", "Curry Cut", "Keema", "Leg & Thigh"] },
  "state": { ... },
  "stage_complete": false
}
```

The `ui_type` field tells the frontend which component to render. The frontend is entirely display-driven — it renders whatever the workflow tells it to render.

---

## What This Workflow Replaced

**V1: 3 agents with handoffs**
- Agent 1: Onboarding chatbot — collected user data via freeform chat
- Agent 2: Meal planner — took onboarding output, generated a meal plan
- Agent 3: Cart builder — took meal plan, built a Licious cart

**Why it failed:**
- Freeform chat produced unstructured inputs (height in cm or inches, weight in kg or lbs) that agents couldn't normalise reliably
- Context was lost in handoffs between agents — Agent 2 and Agent 3 regularly produced generic outputs instead of personalised ones
- The portion optimisation problem (bulk-first selection, wastage minimisation, cross-meal distribution) was too complex for an agent to solve consistently even with detailed instructions
- Latency was high — every step required an LLM call
- Token cost was high — all three agents ran on every interaction

**V2 result:**
- One workflow, one Code node with deterministic logic
- Claude called in at most one edge case per session
- Consistent outputs, lower latency, lower cost, no hallucinations on structured data

---

## How to Import and Run

1. Download `Meal Planner State Engine v1.json`
2. Open your n8n instance → **Import from file**
3. Add credentials:
   - **Postgres**: your Supabase connection string
   - **HTTP Header Auth**: Anthropic API key as `x-api-key`
4. Set up Supabase tables: `users`, `sessions`, `products`, `stock_tracker`
5. Populate `products` table with Licious catalogue data
6. Activate the workflow
7. Point your frontend at the webhook URL: `POST /v2/meal-planning`

---

*Part of the [Licious AI Meal Planner](https://github.com/karanbabbar/licious-ai-meal-planner) — built by [Karan Babbar](https://github.com/karanbabbar)*