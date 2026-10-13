# Crystal Ball Account Insights

Retrieve Total Account Opportunity (TAO) data and generate AI Sales Pitches for SAP accounts — developed in strategic partnership with CMI, leveraging SAP Crystal Ball and MXP MCP connectors.

## What it does

Given an account name or BP ID, this skill:

1. Resolves the account to a Crystal Ball BP ID via MXP account search
2. Fetches the Total Account Opportunity (TAO) table — solution areas with priority, wallet size, gap, competitors, and sales plays
3. On request, generates an AI sales pitch for a specific solution area (polls Crystal Ball until complete)
4. Presents the pitch as a KPI card + full narrative card
5. Automatically follows up with LeanIX customer stories (via ReferenceGPT) and competitive battlecards for key competitors (via CompeteOne)

## Example prompts

- *"Get TAO for Adidas"*
- *"Show total account opportunity for account 161825"*
- *"Generate a sales pitch for LeanIX at BMW"*
- *"What are the high priority solution areas for Siemens?"*
- *"Write a pitch for the HCM solution area for account 12345"*

## TAO columns returned

| Column | Description |
|---|---|
| Solution Area | SAP solution / sub-solution name |
| Code | `materialHierarchyId` (used as `sub_solution_area_code` for sales pitch) |
| Priority | Very High / High / Medium / Low |
| Strategy | Attack / Expand / Defend |
| Est. Wallet (k) | Estimated total wallet in EUR thousands |
| Gap to Wallet (k) | Untapped opportunity in EUR thousands |
| Wallet Share | SAP's current share of wallet |

## Required MCP servers

| Server | Purpose |
|---|---|
| `cmi-referencegpt-agent-dev.cfapps.eu10-004.hana.ondemand.com/mcp` | TAO data and Sales Pitch (`get_account_tao`, `get_account_sales_pitch`) |
| `mxpresso.cfapps.eu10-004.hana.ondemand.com/mcp` | Account search and resolution (`mcp_5fd8a577_search-accounts`) |

## Skill file

[SKILL.md](SKILL.md)

## Author

Shipra Chaudhary  
`shipra.chaudhary@sap.com`

## License

Apache License 2.0 — see [LICENSE](./LICENSE)
