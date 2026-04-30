# MCP Integration

> Using AAP as an MCP tool for Claude, ChatGPT, and any MCP-compatible agent.

## Overview

The AAP MCP server exposes three tools that allow an MCP-compatible AI agent to discover offers, check the requirements for a selected offer, and generate a checkout link for the user.

The MCP checkout tool only returns a URL. It does not process payment, charge the user, or transfer money. The user decides whether to open the link and complete checkout.

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
- Session ID, used when generating a checkout link
- Number of results
- For each offer: provider, name, price, data, contract, network, 5G, credit check, EU roaming, and offer ID

**Example interaction:**

> **User:** Find me a cheap SIM deal with at least 10GB and no contract
>
> **Agent calls** `search_offers({ vertical: "sim", min_data_gb: 10, contract_months: 0 })`
>
> **Returns:** 4 offers from SMARTY, giffgaff, VOXI, Lebara with a session ID

### `get_checkout_requirements`

Return the information, confirmations, eligibility checks, and constraints needed before generating checkout for a specific offer.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `offer_id` | string | Yes | Offer ID from `search_offers` |

**Output:**

Returns:
- Required fields or confirmations
- Eligibility criteria to check with the user
- Offer-specific constraints
- Suggested next step before calling `get_checkout_link`

**Example interaction:**

> **Agent:** SMARTY 30GB looks like the best match because it gives you more data at the same price and keeps the rolling monthly terms you asked for.
>
> **Agent calls** `get_checkout_requirements({ offer_id: "..." })`
>
> **Returns:** any required confirmations and checkout constraints for that offer

### `get_checkout_link`

Generate a hosted checkout link for the selected offer. This records the attribution context internally and returns a URL the user can open to complete checkout.

This tool does **not** process payment, charge the user, or transfer money. It only generates the checkout link.

**Input:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `session_id` | string | Yes | Session ID from `search_offers` |
| `offer_id` | string | Yes | Offer ID the user selected |
| `context` | string | No | Why this offer was chosen |

**Output:**

Returns:
- Checkout URL, if generated
- Status
- Attribution reference details, when available

**Example interaction:**

> **User:** Yes, I'd like to go with that SMARTY deal.
>
> **Agent calls** `get_checkout_link({ session_id: "...", offer_id: "...", context: "Best match: 30GB for £10 rolling monthly" })`

## Full Conversation Flow

```text
User: I need a new SIM deal. Something cheap, at least 10GB, no contract.

Agent: [calls search_offers(vertical="sim", min_data_gb=10, contract_months=0)]
       I found 4 options for you:

       1. SMARTY 30GB — £10/month (Three network, no credit check)
       2. giffgaff 10GB — £10/month (O2 network, 5G included)
       3. giffgaff 25GB — £12/month (O2 network, 5G included)
       4. Lebara 5GB — £5/month (Vodafone network)

       SMARTY 30GB looks like the best match because it gives you more data
       at the same price and keeps the rolling monthly terms you asked for.

User: Sounds good, let's go with SMARTY.

Agent: [calls get_checkout_requirements(offer_id)]
       Before I generate the checkout link, please confirm you're happy with
       the rolling monthly SIM terms and that you'll complete checkout yourself.

User: Yes, that's fine.

Agent: [calls get_checkout_link(session_id, offer_id, context="Best match: 30GB for £10 rolling monthly")]
       Here's your checkout link: https://aap.link/r/...

       You can open the link to review the merchant checkout and complete the purchase.
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `AAP_API_URL` | `http://localhost:3456` | AAP API base URL |
| `AAP_API_KEY` | `aap_demo_key` | API key for authentication |

---

*AAP Specification v1.0.0*
