# SAP Reference Customer Finder

Find SAP customer references and success stories via the RefHub platform and digital library — developed in strategic partnership with CMI and the Global References Team, leveraging SAP MCP connectors.

## What it does

Given a natural language prompt, this skill:

1. Resolves SAP product/solution names via the MXP hierarchy
2. Resolves customer names to account IDs via MXP account search
3. Queries RefHub (`search_refhub`) with structured filters via CMI MCP — returning up to 50 deduplicated reference customers
4. Presents results as an interactive table (multi-customer) or a detail card (single customer)
5. On request, performs a deep-dive: combines internal reference profile descriptions, published customer story assets (from SAP Digital Library), and web sources into a structured summary

## Example prompts

- *"Find references for SuccessFactors in EMEA"*
- *"Show RISE reference customers in Americas"*
- *"Which customers are references for S/4HANA in the automotive industry?"*
- *"Is Adidas a reference customer?"*
- *"Show me ERP references in Germany with a published customer story"*
- *"Tell me more about Adidas's SAP implementation"*

## Filters supported

| Filter | Examples |
|---|---|
| Solution / product | BTP, S/4HANA, SuccessFactors, ERP, LeanIX |
| Customer name | Adidas, BMW, … |
| Region | EMEA, Americas, Asia Pacific, Middle and Eastern Europe |
| Country | Germany, US, India, … |
| Industry | Retail, Healthcare, Utilities, … |
| Company size | Large Enterprise, Medium Enterprise |
| RISE | Detected automatically from "RISE" in prompt |
| Active references | Detected automatically from "active" in prompt |
| AI use case | References using SAP Business AI / GenAI |
| Competitor displaced | Oracle, Microsoft, Salesforce, … |
| Partner involved | Accenture, Deloitte, Capgemini, … |
| Customer story | Filter to customers with a published asset |

## Required MCP servers

| Server | Purpose |
|---|---|
| `cmi-referencegpt-agent-dev.cfapps.eu10-004.hana.ondemand.com/mcp` | RefHub reference data (`search_refhub`, `list_unique_values_refhub`, `download_asset_dgl`) |
| `mxpresso.cfapps.eu10-004.hana.ondemand.com/mcp` | SAP product hierarchy & account resolution (`mcp_3ff46438_query-entries`, `mcp_5fd8a577_search-accounts`) |

## Skill file

[SKILL.md](SKILL.md)

## Author

Shipra Chaudhary  
`shipra.chaudhary@sap.com`

## License

Apache License 2.0 — see [LICENSE](./LICENSE)
