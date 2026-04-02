# Agent Attribution Protocol (AAP) — Specification

> The open standard for AI agent affiliate attribution.

AAP defines how AI agents discover offers, record recommendations, and earn commission when those recommendations lead to purchases — without cookies, tracking pixels, or browser sessions.

## Documents

| Document | Description |
|----------|-------------|
| [Specification](./spec/v1/README.md) | The full AAP v1 protocol specification |
| [AAP Code](./spec/v1/aap-code.md) | The signed attribution token — the core primitive |
| [Attribution Model](./spec/v1/attribution.md) | How attribution works — binary + fallback chain |
| [Commission Model](./spec/v1/commission.md) | Commission types, floors, and settlement |
| [API Reference](./spec/v1/api.md) | REST API endpoints |
| [MCP Integration](./spec/v1/mcp.md) | Using AAP as an MCP tool |
| [Security](./spec/v1/security.md) | Cryptographic signing, fraud prevention, trust model |

## Quick Start

```python
# Python
from aap import AAP
aap = AAP(api_key="your-key")
offers = aap.search(vertical="sim", max_price=10)
rec = aap.recommend(offers[0], context="user asked for cheap SIM")
```

```typescript
// JavaScript/TypeScript
import { AAP } from '@affiliai/sdk';
const aap = new AAP({ apiKey: 'your-key' });
const { offers, sessionId } = await aap.search({ vertical: 'sim', maxPrice: 10 });
const rec = await aap.recommend({ sessionId, offerId: offers[0].id, context: 'user asked for cheap SIM' });
```

```json
// MCP (Claude Desktop, Cursor, etc.)
// Add to your MCP config — no code required
{
  "mcpServers": {
    "aap": {
      "command": "bunx",
      "args": ["@affiliai/mcp"],
      "env": { "AAP_API_KEY": "your-key" }
    }
  }
}
```

## The Problem

Affiliate marketing is a $17B+ industry built on cookies, tracked links, and last-click attribution. AI agents don't click links, don't carry cookies, and don't browse websites. The entire attribution chain breaks when software does the shopping.

## The Solution

AAP replaces the entire chain with one principle: **the transaction flows through the protocol.** Attribution is automatic because AAP was the pipe.

## Licence

This specification is published under the [Apache 2.0 licence](./LICENCE).

"Agent Attribution Protocol" is a trademark of Affili AI Ltd (UK00004367937).

---

*Affili AI Ltd — [affili.ai](https://affili.ai)*
