# AI Gateway Implementations

This repository contains multiple implementation tracks for Azure AI Gateway scenarios.

## Implementations

### 1. Backend Pool Load Balancing — APIM + Azure AI Foundry

Priority-based routing across two Azure AI Foundry endpoints with transparent failover via APIM's built-in backend pool feature.

![Backend pool Load Balancing](https://github.com/Azure-Samples/AI-Gateway/raw/main/images/backend-pool-load-balancing.gif)

- **Guide:** [ai-gateway-backend-pool-load-balancing.md](ai-gateway-backend-pool-load-balancing.md)
- **Source lab:** [Azure-Samples/AI-Gateway — backend-pool-load-balancing](https://github.com/Azure-Samples/AI-Gateway/blob/main/labs/backend-pool-load-balancing/backend-pool-load-balancing.ipynb)

| Decision | Value |
|----------|-------|
| APIM tier | Developer |
| Backends | East US (priority 1) + West Europe (priority 2) |
| Model | `gpt-4o-mini` |
| Load balancing | Priority-based (primary + failover) |
| Auth | APIM managed identity → Cognitive Services OpenAI User |

Additional implementation-specific docs will be added here over time.
