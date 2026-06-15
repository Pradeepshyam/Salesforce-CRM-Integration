# MuleSoft API-Led Connectivity Demo

A 3-tier API-led integration connecting **Salesforce CRM**, **live currency exchange**, and **simulated ERP data** into a unified Account Intelligence API — consumed by a **Salesforce Agentforce agent** for real-time pre-call briefings.

Built to demonstrate MuleSoft API-led connectivity patterns with Salesforce integration.



---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        CONSUMERS                                │
│  Salesforce Agentforce Agent  ·  Portfolio Chat (Claude + API)  │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │ Experience  │  GET /api/account/{name}
                    │    API      │  RAML-first design
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  Process    │  GET /api/account-360/{accountName}
                    │    API      │  Scatter-Gather + DataWeave merge
                    └──┬───┬───┬──┘
                       │   │   │
          ┌────────────┘   │   └────────────┐
          │                │                │
   ┌──────▼──────┐ ┌──────▼──────┐ ┌───────▼─────┐
   │ Salesforce   │ │   ERP Mock  │ │  Currency   │
   │ System API   │ │ System API  │ │ System API  │
   │              │ │             │ │             │
   │ OAuth 2.0    │ │ Static JSON │ │ Frankfurter │
   │ SOQL queries │ │ (SAP sim)   │ │ EUR → USD   │
   └──────────────┘ └─────────────┘ └─────────────┘
```

### Tier 1 — System APIs

| API | Source | Auth | What It Returns |
|-----|--------|------|-----------------|
| **Salesforce System API** | Salesforce Developer Org | OAuth 2.0 Client Credentials | Account (name, industry, support tier, CSM, renewal date) + Opportunity (stage, amount in EUR, close date) |
| **Currency System API** | [Frankfurter API](https://api.frankfurter.app) | None | Live EUR → USD exchange rate |
| **ERP Mock System API** | Static JSON on CloudHub | None | Order history with line items, fulfillment status, and invoice status. Simulates SAP S/4HANA. |

### Tier 2 — Process API

**Account Aggregator** — Calls all 3 System APIs in parallel using MuleSoft's **Scatter-Gather** router, then merges responses with a **DataWeave** transformation into a unified payload:

```dataweave
%dw 2.0
output application/json

var sfData = payload."0".payload
var erpData = payload."1".payload
var currencyData = payload."2".payload

var rate = currencyData.rates.USD default 1.0
var orders = erpData.orders default []
var totalRevenue = sum(orders.totalValueUSD default [0])

---
{
    account: {
        name: sfData.account.name,
        industry: sfData.account.industry,
        supportTier: sfData.account.supportTier,
        csm: sfData.account.csm,
        renewalDate: sfData.account.renewalDate
    },
    opportunity: sfData.opportunity,
    orders: orders,
    currencyRate: { from: "EUR", to: "USD", rate: rate },
    summary: {
        totalOrders: sizeOf(orders),
        totalRevenueUSD: totalRevenue
    }
}
```

### Tier 3 — Experience API

**Account Intelligence API** — Contract-first: RAML spec designed in Anypoint Design Center, published to Exchange, then implemented with APIkit Router scaffolding. Single endpoint: `GET /api/account/{name}`.

---

## Salesforce Integration

The Experience API is consumed from Salesforce via:

1. **Named Credential** with External Credential (Custom auth protocol) pointing to CloudHub
2. **External Service** created by uploading an **OpenAPI 3.0 JSON spec** (manually translated from the RAML — Salesforce doesn't support RAML natively). Auto-generates Apex classes.
3. **Agentforce Agent** ("Sales Intelligence Agent") with an action of type _API → External Services → Get Account_

When a sales rep asks the agent _"Give me a pre-call briefing for Acme Corp"_, the agent calls the External Service action → Named Credential → Experience API → Process API → all 3 System APIs → merged response → natural language briefing.

---

## Demo Companies

| Company | Industry | Tier | Opportunity Stage | Amount (EUR) | Orders | Scenario |
|---------|----------|------|-------------------|-------------|--------|----------|
| Acme Corp | Technology | Enterprise | Negotiation | €130K | 2 (paid) | Happy path renewal |
| Globex Industries | Manufacturing | Professional | Prospecting | €85K | 0 | New deal |
| Initech Systems | Financial Services | Enterprise | Closed Won | €210K | 2 (1 outstanding) | Won deal, payment issue |
| Umbrella Ltd | Healthcare | Standard | Qualification | €60K | 1 (paid) | Early stage |

---

## Repo Structure

```
├── system-apis/
│   ├── currency-system-api/       # Frankfurter API integration
│   ├── erp-mock-system-api/       # Static ERP/SAP simulation
│   └── salesforce-system-api/     # Salesforce OAuth + SOQL
├── process-api/                   # Scatter-Gather aggregation
├── experience-api/                # APIkit + RAML scaffolding
└── specs/
    ├── account-intelligence-api.raml           # Original RAML spec
    └── account-intelligence-api-openapi.json   # OpenAPI 3.0 (for Salesforce External Service)
```

Each API directory contains the Mule XML configuration (`src/main/mule/`) and Maven project file (`pom.xml`). All 5 APIs are deployed independently to **CloudHub 2.0** on 0.1 vCore replicas.

---

## Key Technologies

- **MuleSoft Anypoint Platform** — Anypoint Studio, CloudHub 2.0, API Manager, Design Center, Exchange
- **DataWeave 2.0** — Transformation and data merging
- **RAML 1.0** — API specification language
- **Salesforce** — Developer Org, Agentforce, External Services, Named Credentials
- **OAuth 2.0** — Client Credentials flow (Salesforce Connected App)

---

