# News Break

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

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
