---
name: Launch a NewsBreak advertising campaign
description: Take a NewsBreak advertiser from an organization ID to a live ad — create the campaign, create the ad set with budget, bidding and targeting, upload the creative asset, and create the ad.
api: openapi/news-break-advertising-openapi.yml
operations:
  - getAdminOrgs
  - getAdAccounts
  - createCampaign
  - getEvents
  - createAdSet
  - uploadAdAssets
  - createAd
  - updateAdStatus
generated: '2026-08-01'
method: generated
source: openapi/news-break-advertising-openapi.yml
---

# Launch a NewsBreak advertising campaign

Base URL: `https://business.newsbreak.com/business-api/v1`
Auth: send `Access-Token: <token>` on **every** request. Generate the token in NewsBreak Ad Manager under **Resources → API Access Tokens → Generate Token**.

## Read this before you call anything

- **Every response is HTTP 200.** Success and failure both. Parse the body and check `code == 0`. A non-zero `code` with `errMsg` is a failure — `403` permission denied, `4031` not logged in, `4033` invalid token, `4034` throttled, `-1` unknown.
- **There is no idempotency contract.** No idempotency key, no dedup window. If `createCampaign`, `createAdSet`, `createAd` or `uploadAdAssets` times out or returns `4034`, **do not blindly retry** — call the matching `getCampaigns` / `getAdSets` / `getAds` and search by `name` to see whether the first attempt landed.
- **All money is integer cents.** A $500 daily budget is `50000`.
- **All timestamps are unix integers** on ad sets (`startTime`, `endTime`), but objects return `createTime`/`updateTime` as unix timestamps in *strings*.
- **Rate limits have no headers.** Budget against your tier (Basic 10 QPS / 600 QPM / 864,000 QPD). On `4034` from a QPM breach, wait 5 minutes; from a QPD breach, wait until 00:00 UTC.

## Steps

### 1. Find the organization and ad account

```
GET /org/admin-orgs
```
`getAdminOrgs` returns the organizations where you hold `ORG_ADMIN`. It takes no parameters — the token identifies you. If `data.list` is empty you hold no admin role anywhere.

```
GET /ad-account/getGroupsByOrgIds?orgIds=<orgId>
```
`getAdAccounts` returns ad accounts grouped by organization. Repeat `orgIds` for multiple orgs. Keep the `adAccountId` you are going to spend from.

### 2. Create the campaign

```
POST /campaign/create
{"adAccountId":"…","name":"…","objective":"WEB_CONVERSION","status":"ON"}
```
`objective` is one of `WEB_CONVERSION`, `APP_CONVERSION`, `REACH`, `WEB_TRAFFIC`, `APP_TRAFFIC`. **Objective is fixed at creation** — `updateCampaign` only changes `name`. Choose it deliberately.

Keep `data.id` as the `campaignId`.

### 3. (Conversion objectives) Find the tracking event

```
GET /event/getList/{adAccountId}
```
`getEvents` returns `PIXEL` (web) and `POSTBACK` (app) tracking events. The event's `id` is the value you pass as the ad set's `trackingId`. Note the filter convention: `os=` (present but empty) means *web events only*; omitting `os` entirely means *all operating systems*.

### 4. Create the ad set

```
POST /ad-set/create
{"campaignId":"…","name":"…","budgetType":"DAILY","budget":50000,
 "startTime":1700000000,"endTime":1701000000,"bidType":"CPC","bidRate":500,
 "deliveryRate":"STANDARD","trackingId":"…","platforms":["NEWSBREAK"],
 "targeting":{"ageGroup":{"positive":["18-30","31-44"]}}}
```
`createAdSet` is where budget, bidding, schedule, inventory and audience are decided. Conditional requirements the contract cannot express — enforce them yourself:

| If `bidType` is | You must also send |
|---|---|
| `CPM`, `CPC`, `TARGET_CPA` | `bidRate` (cents) |
| `TARGET_ROAS`, `DAY_ONE_TARGET_ROAS` | `roas` (double) |
| `CPM`, `CPC` | `deliveryRate` applies (`EVENLY` or `ASAP`, default `ASAP`) |

If the parent campaign objective is `APP_TRAFFIC`, `googlePlayId` (Android package name) or `iosAppId` (App Store URL) becomes required.

**Targeting rules.** Each field takes `{"positive":[…],"negative":[…]}`. `"all"` means unlimited and may not be combined with other values or appear in `negative`. Only location and device-location fields may carry both a positive and a negative list — mixing them on any other field is an error.

**Platform rules.** `platforms` defaults to `["APP_AND_WEB_UNLIMITED"]`, which **must appear alone**. At most one `PREMIUM_PARTNERS_*` value may appear. Duplicates are rejected. These three violations are the one place NewsBreak returns a real HTTP 400 rather than a 200 envelope.

Keep `data.id` as the `adSetId`.

### 5. Upload the creative asset

```
POST /ad/uploadAssets        (multipart/form-data)
asset=@creative.png  adAccountId=…  saveToMediaLibrary=true  mediaName="Holiday 1"
```
`uploadAdAssets` returns `data.assetUrl` on the NewsBreak CDN (`static.particlenews.com`) and a `mediaId`. **Only CDN URLs from this operation are accepted by `createAd`** — an externally hosted image URL will be rejected. `mediaName` (3–256 chars) is required only when `saveToMediaLibrary` is true.

### 6. Create the ad

```
POST /ad/create
{"adSetId":"…","name":"…","status":"ON",
 "creative":{"type":"IMAGE","headline":"…","assetUrl":"<from step 5>",
             "description":"…","callToAction":"…","brandName":"…",
             "clickThroughUrl":"https://…"}}
```
`createAd` requires `type`, `headline`, `assetUrl`, `description`, `callToAction` and `brandName` on the creative. For `VIDEO`/`PLAYABLE_VIDEO` add `coverUrl`; for `PLAYABLE_VIDEO` `playableAssetUrl` (an uploaded `.html` asset) is required.

New ads land in `onlineStatus: PENDING` and move to `ACTIVE` or `REJECTED` after review — poll `getAds` rather than assuming delivery.

### 7. Control delivery

```
PUT /campaign/updateStatus/{campaignId}   {"status":"OFF"}
PUT /ad-set/updateStatus/{adSetId}        {"status":"OFF"}
PUT /ad/updateStatus/{adId}               {"status":"OFF"}
```
`updateCampaignStatus`, `updateAdSetStatus` and `updateAdStatus` toggle `ON`/`OFF` at each level. Use these to pause — `deleteCampaign`, `deleteAdSet` and `deleteAd` are destructive and set `onlineStatus` to `DELETED`.

## Failure handling

| `code` | Meaning | Do |
|---|---|---|
| `0` | Success | Read `data` |
| `403` | Permission denied | Check the advertiser granted the token access to this org/ad account |
| `4031` | Not login | You omitted the `Access-Token` header |
| `4033` | Invalid access token | Regenerate the token in Ad Manager |
| `4034` | Rate limit exceeded | Back off 5 min (QPM) or until 00:00 UTC (QPD) — **do not retry the create immediately** |
| `-1` | Unknown error | Retry with backoff; escalate to adsupport@newsbreak.com |
