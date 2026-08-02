---
name: Report on NewsBreak ad performance
description: Pull NewsBreak advertising performance — run a synchronous report, save a reusable custom report, and read publisher-side monetization revenue from the MSP Reporting API.
api: openapi/news-break-advertising-openapi.yml
operations:
  - runSynchronousReport
  - createCustomReport
  - getCustomReportById
  - getMonetizationReport
generated: '2026-08-01'
method: generated
source:
  - openapi/news-break-advertising-openapi.yml
  - openapi/news-break-monetization-reporting-openapi.yml
---

# Report on NewsBreak ad performance

NewsBreak has **two separate reporting APIs on two different hosts** with different credentials. Pick the right one:

| You are | API | Host | Auth |
|---|---|---|---|
| An advertiser / agency buying ads | Advertising API | `https://business.newsbreak.com/business-api/v1` | `Access-Token` header |
| A publisher / supply partner selling inventory | MSP Monetization Reporting API | `https://msp-platform.newsbreak.com` | `token` query parameter |

## Advertiser side — spend and conversions

### Run a report now

```
POST /reports/getIntegratedReport
{"name":"Last 7 days by campaign","timezone":"America/Los_Angeles",
 "dateRange":"LAST_7_DAYS","dimensions":["DATE","CAMPAIGN"],
 "metrics":["COST","IMPRESSION","CLICK","CONVERSION"],
 "filter":"AD_ACCOUNT","filterIds":["123456"],"dataSource":"HOURLY"}
```
`runSynchronousReport` returns the rows inline in `data.rows`.

**Required fields:** `name`, `timezone`, `dateRange`, `dimensions`, `metrics`.

**Date range rules — these will bite you:**
- `dateRange` is one of `FIXED`, `YESTERDAY`, `LAST_7_DAYS`, `LAST_14_DAYS`, `LAST_30_DAYS`, `MONTH_TO_DATE`, `QUARTER_TO_DATE`, `TODAY`.
- `startDate`/`endDate` (`YYYY-MM-DD`) are required **only** when `dateRange` is `FIXED`.
- The `HOUR` dimension only works with `YESTERDAY`, `TODAY`, or a `FIXED` range of **at most 1 day**.
- The `DATE` dimension works with the relative ranges or a `FIXED` range of **at most 30 days**.

**Dimensions are UPPERCASE only:** `DATE`, `HOUR`, `ORG`, `AD_ACCOUNT`, `CAMPAIGN`, `AD_SET`, `AD`, `PLACEMENT`.

**`dataSource` matters for money.** `HOURLY` (the default) is *the official basis for income settlement on NewsBreak*. `REALTIME` is for monitoring only — never reconcile invoices against it.

**`timezone` defaults to PDT, not UTC.** Always set it explicitly or your day boundaries will drift.

### Save a reusable report

```
POST /reports/createReport      # same body shape as above, plus emails[] and editors[]
GET  /reports/getReportById?reportId=…
```
`createCustomReport` persists the definition; `getCustomReportById` runs it and returns rows. Use this when the same report is pulled on a schedule.

### Reading the rows

`ReportRow` is fully denormalized — every row carries an id **and** a display name for each hierarchy level (`orgId`/`organization`, `adAccountId`/`adAccount`, `campaignId`/`campaign`, `adSetId`/`adSet`, `adId`/`ad`).

Two traps:
1. **Monetary metrics come in pairs.** `cost` (integer cents) and `costDecimal` (double cents); same for `conversionValue`, `cpm`, `cpc`, `cpa`. Use the `*Decimal` variant for anything you divide.
2. **`-1` means "not applicable", not zero.** `cpm`, `cpc` and `cpa` return `-1` / `-1.0` when undefined. Filter these out before averaging or you will report negative CPCs.

## Publisher side — monetization revenue

```
GET https://msp-platform.newsbreak.com/reporting
    ?org_id=111&app_id=2&token=…
    &start_date=2026-07-01&end_date=2026-07-31
    &timezone=UTC&breakdown=date&breakdown=seat
    &metrics=revenue&metrics=ecpm&metrics=payout_revenue
```
`getMonetizationReport` returns a **bare JSON array** of rows on success — not the `{code,data}` envelope the Advertising API uses. On failure you get the envelope with a non-zero `code`. Handle both shapes.

- Credentials go in the **query string** (`org_id`, `app_id`, `token`). Treat the whole URL as a secret: do not log it, do not put it in a referrer-bearing context.
- `timezone` is a closed enum here — `PT`, `ET`, `UTC`, `Beijing Time` — defaulting to `UTC`. This is a *different* vocabulary from the advertiser-side IANA strings.
- `breakdown` is repeated per dimension: `date`, `os`, `seat`, `placement_id`, `seat_ad_unit`, `device_type`. **No breakdown is selected by default.**
- `metrics` defaults to `revenue`, `imp`, `ecpm`. Add `payout_revenue` (publisher net), `fill_count`, `request_count`, `click`, `ctr`.
- `filter` requires its corresponding `breakdown` dimension to be selected.
- `revenue` is gross USD; `payout_revenue` is publisher net USD. Do not conflate them.

## Failure handling

Both APIs return HTTP 200 for most errors — always check `code` before trusting a response. `403` permission denied, `4031` not logged in, `4033` invalid token, `4034` throttled, `-1` unknown. Reports are read-only, so a `4034` retry after backoff is safe here (unlike the create operations in the campaign skill).

No rate-limit headers exist. Advertising API tiers: Basic 10 QPS / 600 QPM / 864,000 QPD, Advanced 20/1,200/1,728,000, Premium 30/1,800/2,592,000. Endpoint limits are independent, so a throttled report endpoint does not block campaign reads.
