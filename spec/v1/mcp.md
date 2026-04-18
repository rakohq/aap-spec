# MCP Integration

> Using AAP as an MCP tool for Claude, ChatGPT, and any MCP-compatible agent.

## Overview

The AAP MCP server exposes three tools that allow any MCP-compatible AI agent to discover offers, make recommendations, and initiate checkout — without writing any code.

## Installation

```json
{
  "mcpServers": {
    "aap": {
      "command": "bunx",
      "args": ["@rakohq/mcp"],
      "env": {
        "AAP_API_KEY": "your-api-key",
        "AAP_API_URL": "https://api.rako.sh"
      }
    }
  }
}
```

## Tools

### `search_offers`

Search for product offers available through AAP.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `vertical` | string | No | `sim`, `broadband`, `energy`, `flights`, `hotels`, `insurance` |
| `provider` | string | No | Filter by provider name |
| `max_price` | number | No | Maximum monthly price in GBP |
| `min_data_gb` | number | No | Minimum data allowance in GB |
| `contract_months` | number | No | Contract length (0 = rolling) |

**Output:**

Returns formatted text with:
- Session ID (needed for recommend and checkout)
- Number of results
- For each offer: provider, name, price, data, contract, network, 5G, credit check, EU roaming, offer ID

**Example interaction:**

> **User:** Find me a cheap SIM deal with at least 10GB and no contract
>
> **Agent calls** `search_offers({ vertical: "sim", min_data_gb: 10, contract_months: 0 })`
>
> **Returns:** 4 offers from SMARTY, giffgaff, VOXI, Lebara with session ID

### `recommend`

Record a product recommendation and log the attribution event.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | Yes | Session ID from search_offers |
| `offer_id` | string | Yes | ID of the offer to recommend |
| `context` | string | No | Why this offer was recommended |

**Output:**

Returns:
- Recommendation ID
- Offer summary (provider, name, price)
- Fallback URL for drop-off recovery
- Confirmation that attribution was recorded

**Example interaction:**

> **Agent:** Based on your needs, I recommend the SMARTY 30GB Rolling Monthly at £10/month. No contract, no credit check, unlimited calls and texts.
>
> **Agent calls** `recommend({ session_id: "...", offer_id: "...", context: "User wanted cheap SIM, 10GB+, no contract" })`

### `checkout`

Initiate checkout for a recommended offer.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | Yes | Session ID from search_offers |
| `recommendation_id` | string | Yes | Recommendation ID from recommend |

**Output:**

Returns:
- Transaction ID
- Status
- Checkout URL (if applicable)

**Example interaction:**

> **User:** Yes, I'd like to go with that SMARTY deal.
>
> **Agent calls** `checkout({ session_id: "...", recommendation_id: "..." })`

## Full Conversation Flow

```
User: I need a new SIM deal. Something cheap, at least 10GB, no contract.

Agent: [calls search_offers(vertical="sim", min_data_gb=10, contract_months=0)]
       I found 4 options for you:

       1. SMARTY 30GB — £10/month (Three network, no credit check)
       2. giffgaff 10GB — £10/month (O2 network, 5G included)  
       3. giffgaff 25GB — £12/month (O2 network, 5G included)
       4. Lebara 5GB — £5/month (Vodafone network)

       I'd recommend the SMARTY 30GB at £10/month — you get 3x the data
       of the giffgaff 10GB plan at the same price, and no credit check.

       [calls recommend(session_id, offer_id, context="Best value: 30GB for £10")]

User: Sounds good, let's go with SMARTY.

Agent: [calls checkout(session_id, recommendation_id)]
       I've started the checkout process. Here's a link if you'd like to 
       complete the purchase: https://aap.link/r/...

       If you prefer to check it out yourself later, that link will keep 
       your deal reserved.
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AAP_API_URL` | `http://localhost:3456` | AAP API base URL |
| `AAP_API_KEY` | `aap_demo_key` | API key for authentication |

---

*AAP Specification v1.0.0*
