---
name: sap-reference-customer-finder
description: >-
  Finds SAP reference customer records from the RefHub platform and related reference customer assets from the digital library. Use this skill whenever the user asks to find, search, show, or look up reference customers or asks about reference customer evidence, customer stories, testimonials, or SAP references combined with any product name, solution area, region, country, industry, or customer name — for example: "find references for SuccessFactors in EMEA", "show RISE references in Americas", "look up customer references in automotive industry", "which customers are references for S/4HANA in APAC", "get reference profiles for Retail in Germany", "Is Adidas a reference customer", "Share customer success stories for Adidas on how they implemented BDC, challenges faced etc.". Activates when the user mentions "reference", "reference customer", "RefHub", "reference profile", "Reference customer testimonials", "Reference Customer success stories".
metadata:
  version: 3.6.2
  tags: sap references refhub customer-references crm
  required-mcp-servers: "https://cmi-referencegpt-agent-dev.cfapps.eu10-004.hana.ondemand.com/mcp https://mxpresso.cfapps.eu10-004.hana.ondemand.com/mcp"
---

# SAP Reference Customer Finder

> **MANDATORY PRE-CONDITION — NO EXCEPTIONS**
> You MUST have read this SKILL.md file in the current session before passing `skill_mode: true` to `search_refhub` or `download_asset_dgl`.
> If you have not read this file, or cannot confirm it is loaded, you MUST use `search_ReferenceGPT` instead.
> Do NOT pass `skill_mode: true` based on assumptions — only set it after explicitly reading and following this workflow.

> **Feedback:** Always include this feedback link at the end of every response: 📣[ Give us feedback — tell us what you like or how we can do better](https://sapit-home-prod-004.launchpad.cfapps.eu10.hana.ondemand.com/site#feedbackservice-Display&/topic/e77c482c-8e89-47ab-a652-46206b5e33d8/createFeedback)

## Tools

| Tool | When to use |
|---|---|
| `mcp_3ff46438_query-entries` | Step 2a — resolve solution term to MXP hierarchy |
| `mcp_5fd8a577_search-accounts` | Step 2b — resolve a customer/account name to MXP account ID(s); the ID(s) are then passed to `search_refhub` as `bp_id`  |
| `list_unique_values_refhub` | Step 2b — fetch live distinct DB values for a low-cardinality dimension (country · industry · planning_group · competed_against · replaced_solutions · partners_involved). Call at most once per dimension per session; only when that dimension is in the prompt |
| `search_refhub` | Step 3 — always the final call; returns deduplicated results |
| `download_asset_dgl` | Step 6 — fetch a customer story asset's content by `asset_id`. PDF → full text inline as `extracted_text`; non-PDF → `unsupported_format` + share the `dgl_view_url` browser link. `dgl_view_url` always returned |

## Required MCP Connectors

| MCP Server | Tools provided |
|---|---|
| [CMI MCP](https://cmi-referencegpt-agent-dev.cfapps.eu10-004.hana.ondemand.com/mcp) | `search_ReferenceGPT` · `search_refhub` · `list_unique_values_refhub` · `download_asset_dgl` |
| [MXP](https://mxpresso.cfapps.eu10-004.hana.ondemand.com/mcp) | `mcp_3ff46438_query-entries` · `mcp_5fd8a577_search-accounts` |

---

## Workflow

### Step 1 — Parse filters from the prompt

Extract the dimensions below. Set `null` for anything not mentioned. **Do not infer.**

| Dimension | Examples |
|---|---|
| `solution` | SAP product, solution name, or abbreviation (BTP, S/4, SF, SuccessFactors) |
| `customer_name` | Company or customer name — resolved to account ID(s) via MXP account search (Step 2b), then passed as `bp_id` |
| `region` | EMEA, APAC, Americas |
| `country` | Full country name or abbreviation |
| `industry` | Industry vertical (Retail, Utilities, Healthcare) |
| `bp_id` | 10-digit SAP CRM account ID(s) — a single ID or a list; zero-fill left to 10 |
| `reference_id` | Reference profile ID — always uppercase, starts with `RP` |
| `company_size` | Map to `"Large Enterprise"` or `"Medium Enterprise"` |
| `competed_against` | Competitor evaluated against |
| `replaced_solutions` | Solution or vendor replaced/displaced |
| `partners_involved` | Partner name (e.g. Accenture, Deloitte) |
| `active_references` | `true` when the user asks for "active references" / "active reference profiles" — server excludes inactive profiles (`rp_status = ST13`). Auto-detected from the word "active" in the prompt; also pass explicitly if needed. Do **not** confuse with `regional_buying_classification` (that filter no longer exists). |
| `is_rise` | `true` when the user asks for RISE references / RISE with SAP customers. Auto-detected from "rise" in the prompt; also pass explicitly. |
| `is_cloud` | `true` to filter to Industry Cloud products only. Pass explicitly when the user asks for Industry Cloud references. |
| `ai_usecase` | `Yes` when the user asks for references that use AI / SAP Business AI / GenAI / an AI use case; else null |
| `customer_story` | `Yes` or `No` |

---

### Step 2 — Resolve non-null dimensions (run in parallel)

#### 2a — Resolve solution via MXP

For "ERP" (generic), skip MXP and pass:
```json
{"solution_sets": [{"solution_area": "Cloud ERP Private", "product_level": "L1"}, {"l2_product": ["Cloud ERP Finance", "Cloud ERP SCM"], "product_level": "L2"}], "solution_operator": "OR"}
```

For all other terms, call `mcp_3ff46438_query-entries`:
- `worksphereId`: `6e3c9171-5cd4-4a71-b77e-4a43ffb0cdc1`
- `entityId`: `172e4357-6cda-4417-9444-15617c29c3ca`
- `top`: `500`
- `selectedFields`: `["lpr_name_merged", "solution_area_l4", "solution_area_l4.parent.name", "solution_area_l4.parent.parent.name", "solution_area_l4.parent.parent.parent.name"]`
- `searchCriteriaJson`: `{"and": [{"column": "eppm_status", "equal": "Active"}, {"or": [{"column": "lpr_name_merged", "ilike": "%<TERM>%"}, {"column": "solution_area_l4.name", "ilike": "%<TERM>%"}, {"column": "solution_area_l4.parent.name", "ilike": "%<TERM>%"}, {"column": "solution_area_l4.parent.parent.name", "ilike": "%<TERM>%"}, {"column": "solution_area_l4.parent.parent.parent.name", "ilike": "%<TERM>%"}]}]}`

If 0 results: retry with expanded name. If still 0, continue without solution filter.

**Match level mapping:**

| MXP field | Level | filter key |
|---|---|---|
| `lpr_name_merged` | LPR | `product_name` |
| `solution_area_l4.name` | L4 | `l4_product` |
| `solution_area_l4.parent.name` | L3 | `l3_product` |
| `solution_area_l4.parent.parent.name` | L2 | `l2_product` |
| `solution_area_l4.parent.parent.parent.name` | L1 | `solution_area` |

Always extract ALL levels. Set `product_level` to the matched level. Multiple distinct names at the same level → set that key to a list.

Multiple solutions: run MXP for each independently. Use `solution_sets` when solutions are at different hierarchy levels or trees. Default operator is `AND`; use `OR` only when user says "or" / "either".

#### 2b — Customer name · Region / Country / Industry

**Region** — hard-coded alias table, no lookup:

| User input | DB value |
|---|---|
| EMEA, Europe, MEA | `EMEA` |
| MEE, Eastern Europe, DACH, CEE | `Middle and Eastern Europe` |
| APAC, APJ, Asia, Japan, ANZ | `Asia Pacific` |
| Americas, NA, LATAM, US, Canada | `Americas` |

**Activity** — hard-coded enum, no lookup:

| User phrasing | DB value |
|---|---|
| customer story, success story | `Customer Story` |
| analyst / press / influencer, AR/PR/IR | `AR/PR/IR Activities` |
| logo use | `Logo Usage` |
| reference call, peer reference | `Reference Discussion` |
| speaking engagement, event speaker | `Speaking Engagement` |

**Active references (`active_references`)** — no lookup. Set `active_references=true` when the user asks for "active references", "active reference profiles", or similar. The server excludes profiles with status ST13 (inactive). Auto-detected from the word "active" in the prompt — you do not need to set it explicitly unless the prompt is ambiguous.

**AI use case (`ai_usecase`)** — no lookup. If the user asks for references that **use AI**, mention **SAP Business AI / GenAI**, or want an **AI use case**, set `ai_usecase="Yes"`. The server keeps only references with a real AI use case (non-empty, excluding the `NO AI` placeholder). Otherwise leave null.

**Market unit** — use `planning_group` (not `region`). Resolve via `list_unique_values_refhub("planning_group")`, reason onto exact DB value(s), pass in `planning_group`.

**Country / Industry / Competed against / Replaced solutions / Partners involved** — Lazy Facet Fetch: call `list_unique_values_refhub("<dimension>")` once per session, reason user's term onto exact DB value, pass it.

**Customer name → MXP account search** (this is the ONLY way to resolve a customer name):

When the user names a specific company/customer:
1. Call `mcp_5fd8a577_search-accounts` with the user's typed name.
2. From the initial results, identify the Global Ultimate (GU) account. Then run additional **parallel** `search-accounts` calls using:
   - The **GU's own account name** (e.g. if GU is "OGE Energy Corp", search "OGE Energy Corp") — this reliably surfaces all BPs linked to that GU.
   - Any other likely **full-name variants** if the original term was an abbreviation or short form (e.g. "OG&E" → also search "Oklahoma Gas Electric").
3. From ALL results across all searches, collect every account where `statusOne360 = "Active"` — regardless of type (Global Ultimate, Planning Entity, or Business Partner).
4. Pass ALL collected Active account IDs as `bp_id` in Step 3. Never pass `customer_name` to `search_refhub` — the `bp_id` list is what filters RefHub.
- `mcp_5fd8a577_search-accounts` returns **no Active accounts** → tell the user no matching account was found for "[input]" and ask them to confirm or refine the customer name. Do **not** search RefHub.

---

### Step 3 — Call search_refhub

```
search_refhub(
  prompt="<original user prompt>",
  filters={
    "product_name": "<str|list|null>",
    "l4_product": "<str|list|null>",
    "l3_product": "<str|list|null>",
    "l2_product": "<str|list|null>",
    "solution_area": "<str|list|null>",
    "product_level": "<LPR|L4|L3|L2|L1>",
    "solution_sets": "<list|null>",
    "solution_operator": "<AND|OR|null>",
    "country": "<str or null>",
    "region": "<str or null>",
    "planning_group": "<str|list|null>",
    "industry": "<str or null>",
    "bp_id": "<str|list|null>",
    "reference_id": "<str or null>",
    "company_size": "<Large Enterprise|Medium Enterprise|null>",
    "competed_against": "<str or null>",
    "replaced_solutions": "<str or null>",
    "partners_involved": "<str or null>",
    "activity": "<str or null>",
    "active_references": "<true|null>",
    "is_rise": "<true|null>",
    "is_cloud": "<true|null>",
    "ai_usecase": "<Yes|null>",
    "customer_story": "<Yes|No|null>"
  },
  limit=50,
  include_story=<true when the user asked about a single named customer, false otherwise>,
  skill_mode=true
)
```

**Fallbacks:**
- If `solution_sets` with `AND` returns 0 rows: retry with `OR`. Tell user: *"No customers found with both solutions — showing customers with either."*
- If still 0 rows with multiple non-product filters: drop `country`, then `industry`, then `region` one at a time until results appear. Inform user of each relaxation.
- If 0 after all relaxations: proceed to Step 5 no-results message.

**Response format:** Always `{"rows": [...], "summary": "..."}`. If `summary` exists, output it verbatim before the table.

---

### Step 4 — Handle confirm responses

#### 4a — Customer resolution (no confirm step)

Customer names are resolved automatically in Step 2b via `mcp_5fd8a577_search-accounts` (all Active accounts — GU, PE, and BPs — collected and passed as `bp_id`). There is **no** `confirm_customer` prompt — never ask the user to disambiguate the customer name. `search_refhub` is always called with the resolved `bp_id`(s), never `customer_name`.

#### 4b — Confirm list-filter values

For `competed_against` / `replaced_solutions` / `partners_involved`: if resolved DB value differs notably from user input, confirm before searching:
> "I found '[resolved DB value]' as the closest match for '[user input]'. Should I use that?"

---

### Step 5 — Present results

- Single named customer (resolved to `bp_id` via MXP account search) → **card view** (Step 5a)
- Any other filter → **table view** (Step 5b), regardless of result count

#### 5a — Single customer → card view

First output: "Yes, [Customer Name] is an SAP reference customer."
 "Would you like me to deep-dive into [Customer Name]'s implementation — including their reference details and customer story?" 
If `result.rows` is empty: "I couldn't find [Customer Name] in the SAP reference database. For questions or to request a reference, reach out to the Customer References team: [Submit a request](https://launcher.value-experience-hub.for.sap/experiences/atlas/groups/request/pages/request-employee-submission-form?p_origin_id=OR6)" — then stop.

Call `render_ui` with `hint="detail"` and a one-element array:

| Card field | Source | Notes |
|---|---|---|
| Customer Name | `Customer Name` | As returned |
| Industry | `Industry` | As returned |
| Region | `Region` | As returned |
| Planning Group | `planning_group` | As returned |
| Solution Area | `Solution Area` | As returned |
| Products | `Products` | As returned |
| Activity | `Activity` | Omit if null |
| Reference Profiles & Descriptions | `reference_details[]` | One combined section. For **each** reference profile, output its **hyperlinked title** — `[title](rp_url)` (fall back to `[reference_id](rp_url)`, then plain `reference_id`) — immediately followed on the next line by a 2–3 line summary of that profile's `description`. Repeat per profile. Omit a profile whose `description` is empty; if none have a title/url, derive the links from `References` (split on ` · `) |
| Reference Manager | `Reference Team Contact` | As returned |
| Customer Story | `customer_story_links` | Only if `story_assets` non-empty with titles |
| Updated Date | `Updated Date` | As returned |

After the card stop.

#### 5b — Multiple customers → table view

If `result.rows` is empty: "No reference customers found matching your criteria. For questions or to request a reference, reach out to the Customer References team: [Submit a request](https://launcher.value-experience-hub.for.sap/experiences/atlas/groups/request/pages/request-employee-submission-form?p_origin_id=OR6)" — then stop.

Output a plain Markdown table directly in the chat. Use these columns: Customer Name | Industry | Planning Group | Products | Reference Titles | Activity | Reference Manager | Customer Story| Updated Date .

- `Reference Titles`: use the `References` field verbatim — it is already ` · `-separated markdown links (`[title](rp_url)`); keep each reference title hyperlinked to its RP URL. Fall back to the plain `Reference Titles` field only if `References` is empty.
- `Activity`: comma-separated as returned; blank if null 
- `Reference Team Contact`: render as **`[Contact](mailto:<email>)`** hyperlink — **NOT** the raw email address as link text
- `Customer Story`: "Yes" or "No" — no links
- Empty fields: `—`

After the table, output exactly: "Would you like to deep-dive on any of these customers to learn more about their SAP journey and customer story?" — then stop.

---

### Step 6 — Deep-dive: customer implementation summary

**Trigger:** User says "tell me more", "how did they implement it", "what's their story", "summarize their SAP journey", or asks for detail about a specific customer.

**Prerequisite:** You already have the `search_refhub` result for that customer. If not, resolve the customer name to `bp_id`(s) via `mcp_5fd8a577_search-accounts` (Step 2b) and call `search_refhub` with that `bp_id` and `include_story=true` first — never pass `customer_name` to `search_refhub`.

#### 6a — Gather internal sources (run in parallel)

**Source 1 — Reference descriptions (highest priority)**
From `reference_details[]`, collect every entry's `title`, `rp_url`, and `description`.

**Source 2 — Customer Story asset (if available)**
If `story_assets` is non-empty:
1. Pick the most relevant asset — prefer `assetType` = Customer Story or case study; use `metadataUpdatedDate` as tiebreaker (most recent wins)
2. Call `download_asset_dgl(asset_id="<selected_id>", skill_mode=true)`
3. PDF: read `extracted_text` inline. Non-PDF: `unsupported_format` is set — use `dgl_view_url` + description only
4. If `download_error` / `extract_error` / `extracted_text` is empty: fall back to `dgl_view_url` + description
5. Always use `asset_title` from `download_asset_dgl` as link text — never `assetType` or category labels

#### 6b — Gather external source

Search the web for `"<Customer Name>" SAP <solution>`. Use results to supplement — never to contradict — internal sources.

#### 6c — Compose the segmented summary

Structure the response in clearly labelled sections **in this priority order**:

---

**🔒 Internal SAP Reference Data**
*(sourced from RefHub - notes from reference manager)*

- Business challenge and why the customer chose SAP
- Solutions implemented (use `Products_by_SA` for scope)
- Key outcomes, metrics, or quotes from `reference_details[].description`
- Reference links: list each reference as `[title](rp_url)` from `reference_details` — use `reference_id` as fallback text if `title` is null

---

**🛡️ Internal Published Customer Story**
*(sourced from SAP Digital library - SAP verified)*

- Business challenge and why the customer chose SAP
- Solutions implemented
- Key outcomes, metrics, or quotes
- Link: `[Read full story](<dgl_view_url>)`
- Additional assets (if any): list each as `[<asset_title>](<dgl_view_url>)` — use the `asset_title` field returned by `download_asset_dgl`, never `assetType` or category labels

*(Omit this section if no Customer Story URL exists)*

---

**🌐 External / Web Sources**
*(sourced from public web — not verified by SAP)*

- Any relevant supplementary context from web search
- Cite each source as `[Source title](URL)`