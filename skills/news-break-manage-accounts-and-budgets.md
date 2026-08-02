---
name: Manage NewsBreak ad accounts, users and spending caps
description: Provision NewsBreak advertising organizations — create ad accounts, grant and revoke user access by role, and read or set account-level spending caps.
api: openapi/news-break-advertising-openapi.yml
operations:
  - getAdminOrgs
  - createAdAccount
  - getAdAccounts
  - addAdAccountUser
  - deleteAdAccountUser
  - getAccountSpendingCap
  - updateAccountSpendingCap
generated: '2026-08-01'
method: generated
source: openapi/news-break-advertising-openapi.yml
---

# Manage NewsBreak ad accounts, users and spending caps

Base URL: `https://business.newsbreak.com/business-api/v1` · Auth: `Access-Token` header.

This is the agency/platform provisioning surface: standing up ad accounts under an organization, controlling who can touch them, and capping what they can spend.

## Roles

| Role | Scope | Grants |
|---|---|---|
| `ORG_ADMIN` | Organization | Required to update spending caps. `getAdminOrgs` lists the orgs where you hold it. |
| `ACC_ADMIN` | Ad account | Full ad account control |
| `ACC_OPERATOR` | Ad account | Operate campaigns/ad sets/ads |
| `ACC_VIEWER` | Ad account | Read only |

## Steps

### 1. Confirm your admin scope

```
GET /org/admin-orgs
```
`getAdminOrgs` takes no parameters — the token identifies the caller. `data.list` is the set of organizations where you are `ORG_ADMIN`. An empty array is a valid answer, and it means you cannot create ad accounts or change spending caps anywhere.

### 2. Create an ad account

```
POST /ad-account/create
{"adAccountName":"…","companyName":"…","industry":"Vehicles//Vehicle Type","orgId":"…"}
```
`createAdAccount` requires all four fields.

- `adAccountName` 1–256 chars; `companyName` 1–1024 chars.
- `industry` must be `<category>//<subcategory>` (note the **double slash**) and must match a value from NewsBreak's published industry list — e.g. `"Ad Safety Risk//Other"`. An arbitrary string will be rejected.
- **No idempotency.** A retried create makes a second ad account. If a create times out, call `getAdAccounts` and match on name before retrying.

### 3. List ad accounts

```
GET /ad-account/getGroupsByOrgIds?orgIds=123&orgIds=234
```
`getAdAccounts` returns accounts grouped by organization (`data.list[].adAccounts[]`). Repeat `orgIds` per organization. You only get accounts you have access to — an account missing from the response means no access, not that it does not exist.

### 4. Grant access

```
POST /ad-account/addUser
{"orgId":"…","adAccountId":"…","email":"user@example.com","role":"ACC_VIEWER"}
```
`addAdAccountUser` is keyed on **email**. If the user does not exist yet, NewsBreak creates an account and sends an invitation email — so calling this operation sends real mail to a real person. It also grants an organization-level membership role if the user has none.

`adAccountId` must belong to `orgId` or the call fails.

### 5. Revoke access

```
POST /ad-account/deleteUser
{"orgId":"…","adAccountId":"…","userId":"…"}
```
`deleteAdAccountUser` is keyed on **userId**, not email — asymmetric with add. It removes only the ad-account-level role; **any organization-level role is left in place**, so this alone does not fully offboard someone.

> There is no operation to list the users on an ad account. Membership is write-only from the API's point of view — keep your own record of who you granted, because you cannot read it back, and you need the `userId` (which add does not return) to revoke.

### 6. Read spending caps

```
POST /balance/getAccountBudgetInfo
{"accountIds":["123","234","456"]}
```
`getAccountSpendingCap` returns per-account `accountRemaining`, `accountSpendingCap` and `accountTotalSpend` as doubles.

- Send between 1 and 500 IDs. Zero IDs is an error; more than 500 returns only the first 500.
- Only accounts that **have an account-level cap set** and that you can access are returned meaningfully. Others come back with `canViewBudget: false` and a `failMessage` — check that flag per row before reading the numbers.

### 7. Set spending caps

```
POST /balance/updateAccountsBudget
{"adAccountsBudgetUpdate":[{"adAccountId":"123","budget":500000},
                           {"adAccountId":"234","budget":1000000}]}
```
`updateAccountSpendingCap` is **ORG_ADMIN only**.

- `budget` is in **cents**, bounded 0–10,000,000,000 (\$0–\$100M).
- The new cap **must be greater than the account's current total spend** — read `accountTotalSpend` from step 6 first.
- If no cap is currently set, this enables and sets it.
- **Partial success is the norm.** Each account is processed individually; the response is `data.list[]` of `{adAccountId, message}`. A `code: 0` at the envelope level does **not** mean every account succeeded — iterate the list and read every `message`.

## Failure handling

All of these return HTTP 200; check `code`. `403` permission denied (usually: you are not `ORG_ADMIN`, or the ad account is not in that org), `4031` missing header, `4033` bad token, `4034` throttled, `-1` unknown.

Treat `createAdAccount`, `addAdAccountUser` (sends email) and `updateAccountSpendingCap` (moves money limits) as operations requiring human confirmation in an agent loop — none of them are reversible through the API, and none are idempotent.
