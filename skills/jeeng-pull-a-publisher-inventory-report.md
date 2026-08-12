---
name: jeeng-pull-a-publisher-inventory-report
description: Pull container, placement or flexible performance reporting for a publisher from the
  Jeeng / OpenWeb Email Monetization API, including the Apple MPP adjusted-opens caveat and the CSV
  redirect on the flexible report.
api: Jeeng Email Monetization — Publishers API
generated: '2026-08-12'
method: generated
source: openapi/jeeng-publishers-openapi.yml, openapi/jeeng-authentication-openapi.yml
operations:
  - getting-an-access-token
  - reporting-containers
  - reporting-placements
  - reporting-publisher-performance
---

# Pull a publisher inventory report

Reports how a publisher's email inventory performed: served ads, opens (raw and Apple-MPP
adjusted), clicks, revenue, CTR and RPM, at container level, placement level, or as a
caller-shaped CSV.

## Before you start

API access is not self-serve. A Jeeng account manager provisions a `client_id` and `client_secret`
for your account; there is no signup form. If you do not have credentials, this flow cannot run.

## 1. Get an access token — `getting-an-access-token`

`POST https://login.microsoftonline.com/revenuestripe.onmicrosoft.com/oauth2/v2.0/token`
with `Content-Type: application/x-www-form-urlencoded` and the fields:

- `client_id` — provisioned by Jeeng support
- `client_secret` — provisioned by Jeeng support
- `grant_type` — always `client_credentials`
- `scope` — always `api://revenuestripe.onmicrosoft.com/partners/.default`

Use the returned `access_token` as `Authorization: Bearer <access_token>`. It expires in about an
hour (`expires_in: 3599`), so cache it and refresh rather than requesting one per call.

**The single most common failure is a wrong scope.** Any other scope value produces `401 Not
authorized to this endpoint.` — and a wrong-scope token can still succeed on some endpoints, so the
error often appears only on `reporting-publisher-performance`. If you see it, re-request the token
before debugging anything else.

## 2. Choose the right report

| Need | Operation | Grain |
| --- | --- | --- |
| Performance per container | `reporting-containers` | daily |
| Performance per placement | `reporting-placements` | daily |
| Caller-shaped fields/filters/grouping as CSV | `reporting-publisher-performance` | your choice |

## 3a. Container or placement report

`GET /reporting/containers` or `GET /reporting/placements` on `https://powerinbox.azure-api.net/`.

Both require two query parameters:

- `api-version=1.0` — required on **every** operation; omitting it is a request error, not a
  default to latest.
- `$filter` — a required OData V4 expression, e.g. `Date eq 2023-12-13Z`.

Responses are OData V4 JSON. A response **may** include a `next` link; follow it until it is absent.
No page-size parameter is published, so do not try to control the page size.

## 3b. Flexible performance report — `reporting-publisher-performance`

`GET /reporting/publisherperformance` with:

- `Accept: text/csv` — a required **header**
- `requestedFields` — comma-separated, e.g. `DateTime,PlacementId,Opens,Clicks,Revenue`
- `startInclusive` / `endExclusive` — RFC 3339 timestamps; the range is half-open and must be
  **32 days or fewer**
- `filter[Field]=value` — optional key/value pairs, e.g. `filter[PlacementId]=456`
- `groupBy` — optional comma-separated fields; every field named here **must** also appear in
  `requestedFields`
- `api-version=1.0`

This endpoint answers **303 See Other** with a `Location` header. Follow the redirect to download
the CSV. Most HTTP clients do this automatically — if yours does not, follow it explicitly.

## 4. Read the numbers correctly

- **Adjusted vs raw opens.** "Adjusted" columns apply a correction factor for Apple Mail Privacy
  Protection preloads. CTR and RPM are calculated on adjusted opens. Never mix adjusted and raw
  opens across reports.
- **All timestamps are UTC** and cannot be adjusted.
- **Hourly is the floor.** Finer groupings may appear in the portal but are not supported.
- **Revenue is indicative.** Jeeng states the payment invoice, not the report, carries the final
  revenue number. Do not use report revenue for reconciliation or accounting.
- Invalid Traffic (IVT) filtering has already been applied to these figures.

## 5. Handle errors

| Status | Meaning | Do |
| --- | --- | --- |
| 400 | Invalid parameters | Check field names, that `groupBy` ⊆ `requestedFields`, and that the range ≤ 32 days |
| 401 `Unauthorized` | Missing/expired token | Refresh the token |
| 401 `Not authorized to this endpoint.` | Wrong scope | Re-request with the partners `.default` scope |
| 500 | Server error | Retry, then contact support@jeeng.com with the placement id and date range |

Errors come back as `{"statusCode": …, "message": …}` — the Azure API Management envelope, **not**
RFC 9457 problem+json. There is no error code vocabulary, so branch on HTTP status.

## Notes

- No rate limits or `RateLimit-*` headers are published; be conservative and cache report results.
- There is **no** operation to list containers or placements. Ids come from the portal
  (Container Details / Placement Details) or from a prior report's `ID` column.
