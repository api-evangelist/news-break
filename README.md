# News Break

NewsBreak is the leading local news and information platform in the United States, operated by
Particle Media, Inc. Its AI-powered platform serves more than 40 million Americans a month with
local news, alerts, crime maps, weather, events and community content.

- https://www.newsbreak.com/
- https://www.newsbreak.com/who-we-are

## APIs

| API | Host | Docs |
|---|---|---|
| NewsBreak Advertising API (API for Business) | `https://business.newsbreak.com/business-api/v1` | https://advertising-api.newsbreak.com/hc/en-us |
| NewsBreak MSP Monetization Reporting API | `https://msp-platform.newsbreak.com` | https://doc.msp.newsbreak.com/business-api-doc/docs/overview/ |

## Artifacts

- `openapi/` — two OpenAPI 3.1.0 documents (28 operations). **NewsBreak publishes no
  machine-readable contract**; both were authored by API Evangelist from the provider's published
  reference, with every operation carrying an `externalDocs` link back to the source article.
- `llms/` — NewsBreak's own `llms.txt`, saved verbatim.
- `authentication/`, `conventions/`, `errors/`, `rate-limits/`, `lifecycle/`, `changelog/`,
  `conformance/`, `data-model/` — the documented contract semantics.
- `skills/`, `arazzo/`, `mcp/`, `agentic-access/`, `overlays/` — agent-facing derivations.
- `security/`, `well-known/` — probe results, including negative results.

## Notable

- **No OpenAPI, no MCP server, no agent card, no SDKs, no GitHub org, no Postman collection, and no
  `/.well-known/` document on any host.** `msp-platform.newsbreak.com/swagger.json` answers HTTP 200
  with a **zero-byte body**.
- **Errors arrive inside HTTP 200.** Both APIs return `{code, errMsg, data}`; only `code == 0` is
  success. There is no RFC 9457.
- **No idempotency contract** on any create operation.
- Rate limits are published as real numbers (Basic 10 QPS / 600 QPM / 864,000 QPD, up to Premium
  30 / 1,800 / 2,592,000) but signalled only by body code `4034` — there are no rate-limit headers.
- `business.newsbreak.com`, the host that carries the API credential, is the only NewsBreak host on
  **TLS 1.2 with no HSTS**.
- The MSP Reporting API passes its secret in the **query string**.
