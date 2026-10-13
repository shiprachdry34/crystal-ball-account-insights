---
name: crystal-ball-account-insights
description: >-
  Retrieves Total Account Opportunity (TAO) data and generates Sales Pitches for a Crystal Ball account. Use this skill whenever the user asks about TAO, total account opportunity, wallet share, sales opportunities, account potential, or wants a sales pitch for a solution area — with either an account name or a BP_ID. Trigger phrases include: "get TAO for", "show TAO for", "TAO details for", "pull TAO data", "total account opportunity for", "what is the TAO for", "show opportunities for", "account TAO", "TAO for account", "generate a sales pitch for", "write a pitch for", "sales pitch for [solution] at [account]".
metadata:
  author: Shipra
  version: 1.7.0
  tags: crystal-ball tao sales account-insights opportunities sales-pitch
  required-mcp-servers: "https://mxpresso.cfapps.eu10-004.hana.ondemand.com/mcp"
---

# Crystal Ball Account Insights — TAO Lookup & Sales Pitch

Retrieve Total Account Opportunity (TAO) data and generate AI Sales Pitches for a Crystal Ball account.

---

## Required Connectors

This skill requires two MCP connectors and references two supporting skill docs:

- **MXP** — for account search (`mcp_5fd8a577_search-accounts`)
- **CMI MCP** — for TAO data, Sales Pitch, and CompeteOne (`mcp_313d61d5_get_account_tao`, `mcp_313d61d5_get_account_sales_pitch`, `mcp_313d61d5_search_competeone`)

Supporting skill loaded during Step 4:
- `sap-reference-customer-finder` — for LeanIX customer story search

If either connector is unavailable, stop and tell the user which connector needs to be enabled before proceeding.

---

## Parallel Execution Overview

To minimise end-to-end time, two enrichment calls are fired as **background parallel tasks** as soon as their inputs are available — they do not wait for the sales pitch to complete:

| Task | Fires when | Inputs needed |
|---|---|---|
| **Step 4a-i** — account reference check | End of Step 1c (bp_id confirmed) | `bp_id` + solution name (from user’s request) |
| **Step 4b** — competitive asset search | End of Step 3a (TAO row resolved) | Solution name + `competitors` field from TAO row |

Both tasks run in parallel with the main flow. By the time the ~2-minute pitch generation completes, their results are already available. **Both results are embedded directly into Card 2 of the pitch** under `Customer References` and `Competitive Battlecards` — no separate presentation step is needed.

---

## Step 1 — Resolve the Account ID (BP_ID)

### 1a — Check if BP_ID is already in the prompt

Scan the user's message for a numeric account/BP ID (e.g. "161825", "account 161825", "BP ID 161825").

- **Numeric ID found** → use it as `account_id` and **skip immediately to Step 2**. Do not call `search-accounts`.
- **Only an account name given** → continue to Step 1b.

### 1b — Search for the account by name

Call `mcp_5fd8a577_search-accounts` with the account name exactly as the user typed it:

```
mcp_5fd8a577_search-accounts(
  searchTerm = "<account name from prompt>",
  prioritizeAccountType = true,
  limit = 20
)
```

`prioritizeAccountType = true` ensures results are ordered: **Global Ultimate (GU) first → Planning Entity → Business Partner**.

### 1c — Handle search results

**Single result returned:**
Confirm the match and proceed to Step 2 without asking. Example:
> "Found **Acme Corp** (ID: 161825 | Global Ultimate | Germany | Consumer Products). Fetching TAO data…"

**Multiple results returned:**
Present a numbered list sorted by account type (GU → Planning Entity → Business Partner). Include ID, type, country, and industry for each. Example:

> I found multiple accounts matching "[name]". Present the result in a table format. Please select the one you want:
>
> 1. **Acme Corp** — ID: 161825 | Global Ultimate | Germany | Chemicals
> 2. **Acme Corp GmbH** — ID: 161900 | Planning Entity | Germany | Chemicals
> 3. **Acme Corp Ltd** — ID: 162100 | Business Partner | UK | Chemicals
>
> Which one should I use?

**Wait for the user to confirm** before proceeding. Do not assume or default to the first result.

**No results returned:**
Inform the user and ask them to check or rephrase the account name. Do not proceed to Step 2.

---

> ⏩ **PARALLEL FIRE — Step 4a-i (account reference check)**
> As soon as `bp_id` is confirmed (single result) or the user selects an account (multiple results), **immediately** call `search_refhub` for the account reference check (Step 4a-i) **in parallel** with proceeding to Step 2. Do NOT wait for the TAO data or the sales pitch.
>
> Inputs available at this point:
> - `bp_id` — zero-padded 10-digit account ID from the account search result
> - Solution name — from the user’s original request (e.g. "LeanIX")

---

## Step 2 — Fetch TAO Data

Once `account_id` is confirmed, call:

```
mcp_313d61d5_get_account_tao(
  account_id = <confirmed account_id>,
  max_level = 3
)
```

Use `max_level = 4` if the user asks for full detail or wants sub-area drill-down.

### 2a — Apply priority filter (if specified)

If the user mentioned a priority in their prompt (e.g. "High or Very High", "only Very High"), filter the returned rows to matching values before rendering.

Supported priority values: `Very High`, `High`, `Medium`, `Low`, `Very Low`.

If no priority filter is mentioned, show all rows.

### 2b — Handle large responses

If the TAO result is offloaded to a file (large payload), use `read_file` to load the data before filtering and rendering.

### 2c — Render the results

Display the TAO data as a table using `render_ui` with `hint: "table"`. Include these columns:

| Column | Source field |
|---|---|
| Solution Area | `solution_area` |
| Code | `solution_area_code` |
| Parent | `parent` (show "—" for top-level L2 areas) |
| Priority | `priority` |
| Strategy | `strategy` |
| Est. Wallet (k) | `estimated_wallet_k` (rounded to integer) |
| Gap to Wallet (k) | `gap_to_wallet_k` (rounded to integer) |
| Wallet Share | `wallet_share` (formatted as %) |

After the table, add a brief summary (3–5 bullets) highlighting:
- The solution area with the largest estimated wallet
- The largest untapped gap (highest `gap_to_wallet_k`)
- Very High priority areas with low wallet share — these are **Attack** opportunities
- Very High priority areas with high wallet share — these are strong **Expand/Defend** positions

---

## Step 3 — Generate a Sales Pitch (when requested)

Use this step when the user asks for a sales pitch, a pitch narrative, or talking points for a specific solution area at an account.

### 3a — Resolve the solution_area_code

The sales pitch API requires a `sub_solution_area_code` (the `materialHierarchyId`). This comes from the `solution_area_code` column in the TAO table.

- **If Step 2 was already called** → look up the user's mentioned solution in the TAO table and read its `solution_area_code` directly. Do not call `get_account_tao` again.
- **If Step 2 was NOT called** → call `get_account_tao` now (Step 2) to get the table, then match the user's solution to its `solution_area_code`.

**Matching the user's solution to a code:**
Search the TAO table for a row whose `solution_area` name contains the user's mentioned solution (case-insensitive). Examples:

| User says | Match in table | `solution_area_code` |
|---|---|---|
| "Integration Suite" | SAP Integration Suite | `SAINS` |

**Use only L3 codes** (sub-solution area level, e.g. `SACXSS`)

---

> ⏩ **PARALLEL FIRE — Step 4b (competitive asset search)**
> As soon as the target TAO row is identified and its `competitors` field is read, **immediately** launch all Step 4b `search_competeone` calls **in parallel** with Step 3b (pitch generation). Do NOT wait for the pitch to complete.
>
> Inputs available at this point:
> - Solution name — from the user’s original request
> - Competitor list — from the `competitors` field of the resolved TAO row (comma-separated; use the first 3–4 named competitors for the searches)

---

### 3b — Call get_account_sales_pitch

```
mcp_313d61d5_get_account_sales_pitch(
  account_id = <confirmed account_id>,
  sub_solution_area_code = <resolved solution_area_code>
)
```

Generation runs server-side and takes up to ~2 minutes. The tool polls automatically — do not call it again while waiting.

While waiting for the pitch, the parallel tasks (Step 4a-i and Step 4b) continue running in the background.

### 3c — Present the pitch as two cards

When the pitch tool returns **and** both parallel tasks (Step 4a-i and Step 4b) are complete, render the pitch as **two `render_ui` cards** in sequence.

#### Card 1 — KPI card

Call `render_ui` with `hint: "kpi"` to surface the four key opportunity metrics. Extract these values from the pitch response text:

| KPI field | What to extract from the pitch response |
|---|---|
| Estimated Wallet (EUR) | The estimated wallet size for the sub-solution area (e.g. "3.5M€" → "3,500 K") |
| SAP Wallet Share | The current wallet share percentage (e.g. "13.0%") |
| Gap to Capture (EUR) | The remaining gap to wallet (e.g. "3.0M€" → "3,000 K") |
| Cloud Exit ACV (EUR) | SAP's current cloud exit ACV (e.g. "0M€" or "273K") |

Express monetary values in EUR thousands (K). Title the card:
> `SAP [Solution Area Name] Opportunity KPIs — [Account Name]`

Add a `dataDescription` note: `Account [ID] · Monetary values in EUR thousands`.

Example `render_ui` call:
```
render_ui(
  intent = "SAP LeanIX Opportunity KPIs — adidas AG",
  hint = "kpi",
  data = [{
    "Estimated Wallet (EUR)": "887 K",
    "SAP Wallet Share": "31%",
    "Gap to Capture (EUR)": "614 K",
    "Cloud Exit ACV (EUR)": "273 K"
  }],
  dataDescription = "Account 161825 · Monetary values in EUR thousands"
)
```

#### Card 2 — Text card

Call `render_ui` with `hint: "text"` to render the full pitch narrative. Card 2 contains **all content in one card** — the pitch body from the API plus the enrichment results from the two parallel tasks. Structure under these headings in order:

```
**Account Overview**
[Full account overview paragraph — do not truncate]

---

**The Opportunity**
[Key metrics and recommended strategy — written as flowing sentences]

---

**Buying Signals — Intent This Week**

**1. [Signal category name]**
[Full paragraph for signal 1 including recommended up-sell]

**2. [Signal category name]**
[Full paragraph for signal 2 including recommended up-sell]

---

**Cross-sell Opportunities**
[Full cross-sell paragraph]

---

**Next Steps**
1. [Step 1]
2. [Step 2]
3. [Step 3]
4. [Step 4]

---

**Customer References**
[Embed the full result from Step 4a-i here using the exact text rules below]

---

**Competitive Battlecards**
[Embed the full result from Step 4b here as a markdown bullet list]
```

Do NOT truncate, summarise, or paraphrase any section. Preserve the full sentences exactly as returned by the pitch API.

> ⛔ **Strip tool promotion links from Next Steps:** When extracting the Next Steps section from the pitch API response, **remove** any sentences, bullet points, or hyperlinks that mention or link to **CompeteGPT**, **CustomerGPT**, or **ReferenceGPT**. These tools are available directly and do not need to be surfaced as links in the card. Keep all other Next Steps content intact.

Title Card 2:
> `SAP [Solution Area Name] Sales Pitch — [Account Name]`

##### Customer References — content rules

Use the result from Step 4a-i to populate this section. Apply the exact text based on what was returned:

**Account IS a reference customer for the solution:**
```
[Account name] is a reference customer for [solution].
- [[Reference title 1]]([profile link 1])
- [[Reference title 2]]([profile link 2])
```

**Account is NOT a reference customer for the solution, but has profiles for other SAP solutions:**
```
[Account name] is not currently listed as a reference customer for [solution].

They do have active reference profiles for other SAP solutions:
- [[Reference title 1]]([profile link 1])
- [[Reference title 2]]([profile link 2])
```

**No reference profiles found at all:**
```
[Account name] is not currently listed as a reference customer for [solution] or any other SAP solution.
```

##### Competitive Battlecards — content rules

Use the results from Step 4b to populate this section as a markdown bullet list. Each found asset becomes one line:

```
- [[Asset Title]]([tracking_link]) ([Competitor] | [published_date])
```

If no assets were found for any competitor after all retries:
```
No competitive battlecards found. Use CompeteGPT for on-demand competitive positioning.
```

Do NOT use a table format in the card. Do NOT download or summarise asset content. Do NOT add a link or mention of CompeteGPT, CustomerGPT, or ReferenceGPT.

#### Error handling

If `errorMessage` is set and the pitch response is empty, tell the user the pitch could not be generated and show the error. Suggest trying an L3 sub-area code if an L2 code was used.

---

## Step 4 — Post-Pitch Follow-up

The Step 4a-i and Step 4b results are already embedded in Card 2. The only remaining action is to ask the user the follow-up question below.

### 4a — Reference Customer Check (RefHub)

Load the `sap-reference-customer-finder` SKILL.md and follow its workflow for tool calls and filter resolution.

#### Step 4a-i — Account reference check (fired in parallel at Step 1c)

This call was already dispatched at Step 1c. The `search_refhub` call uses:

```
search_refhub(
  skill_mode = true,
  prompt = "Is [account name] a reference customer for [solution]?",
  filters = {
    "bp_id": "<10-digit zero-padded account ID>",
    "l3_product": "<solution name from user’s request, e.g. 'SAP LeanIX'>",
    "product_level": "L3"
  },
  limit = 10
)
```

Collect the result and embed it into Card 2 under `Customer References` (see Step 3c rules above). Do NOT present it separately as plain text.

#### Step 4a-ii — Follow-up question (ask after Card 2 is rendered)

After presenting the two pitch cards, ask the user the following question exactly:

> "Would you like me to show SAP [solution] reference customers in the [industry] industry or find customer success stories? Or deep-dive on a CMI competitive asset for a competitor or solution?"

**Wait for the user’s reply before taking any action.** Handle their response as follows:

- **Reference customers / success stories** → run the industry-wide `search_refhub` call (Step 4a-ii-A below)
- **Deep-dive on a competitive asset** → ask which competitor or solution they want to explore, then download the most relevant asset from Step 4b results using `download_asset_dgl` (if `dgl_id` is set) or `download_asset_competeone` (if `dgl_id` is null), and summarise the content
- **Both** → handle sequentially, reference search first

##### Step 4a-ii-A — Industry-wide reference search

If the user wants reference customers or customer success stories, call `search_refhub` with `skill_mode: true`:

```
search_refhub(
  skill_mode = true,
  prompt = "Show SAP [solution] reference customers in [industry]",
  filters = {
    "l3_product": "<solution name from TAO row>",
    "product_level": "L3",
    "industry": "<account industry from Step 1>",
    "customer_story": "Yes"
  },
  limit = 10
)
```

Present the results as a **plain markdown table** (do NOT use `render_ui`):

| Customer | Region | Country | Products | Story |
|---|---|---|---|---|
| [name] | [region] | [country] | [products] | [View]([reference link]) |

If no stories are found for that industry, tell the user and ask: "No stories found for [industry]. Would you like me to widen the search and show SAP [solution] references across all industries?" If they confirm, retry without the `industry` filter and note the relaxation.

---

### 4b — Competitive Assets (CompeteOne, fired in parallel at Step 3a)

This search was already dispatched at Step 3a using the competitor list from the TAO row’s `competitors` field. Use the first 3–4 named competitors from that field for the searches:

```
search_competeone(competitor="<competitor 1>", solution="<solution name>", asset_type="Battlecard")
search_competeone(competitor="<competitor 2>", solution="<solution name>", asset_type="Battlecard")
search_competeone(competitor="<competitor 3>", solution="<solution name>", asset_type="Battlecard")
```

If a competitor returns 0 results with `asset_type="Battlecard"`, retry once without the `asset_type` filter for that competitor. If all per-competitor searches return 0, retry once with `keyword="<solution name>"` and `asset_type="Battlecard"` as a broad fallback.

Collect the results and embed them into Card 2 under `Competitive Battlecards` (see Step 3c rules above). Do NOT present results separately or use a table.

---

## Edge Cases

| Situation | What to do |
|---|---|
| Priority filter returns 0 rows | Tell the user, offer to show all priorities |
| TAO tool returns empty or error | Tell the user and suggest verifying the account ID in Crystal Ball |
| User provides both a name and an ID | Use the numeric ID directly — skip account search |
| Sales pitch error with L2 code | Ask the user to pick a sub-area (L3 code) from the TAO table |
| solution_area_code not found for user's solution | Show the full TAO table and ask the user to pick the correct row |
| Sales pitch times out | Inform the user, suggest retrying — generation can occasionally take longer |
| KPI values not clearly stated in pitch response | Use 0 or "N/A" for that field and note it in the card description |
| TAO row `competitors` field is empty | Fall back to searching `search_competeone` with only the solution name keyword and no competitor filter |
| Step 4a-i or 4b still running when pitch completes | Wait for both to finish before rendering Card 2 |