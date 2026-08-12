---
name: jeeng-pull-an-advertiser-campaign-report
description: Pull campaign and campaign-line performance from the Jeeng / OpenWeb Email Monetization
  advertiser API using OData V4 grain reports, or the flexible advertiser performance report that
  returns a CSV via a 303 redirect.
api: Jeeng Email Monetization — Advertisers API
generated: '2026-08-12'
method: generated
source: openapi/jeeng-advertisers-openapi.yml, openapi/jeeng-authentication-openapi.yml
operations:
  - getting-an-access-token
  - reporting-campaigns
  - reporting-campaign-lines
  - reporting-advertiser-performance
---

# Pull an advertiser campaign report

Three ways to read advertiser delivery: a campaign-level grain report, a campaign-line-level grain
report, and a flexible CSV report you shape yourself.

## 1. Get an access token — `getting-an-access-token`

Form-encoded `POST` to
`https://login.microsoftonline.com/revenuestripe.onmicrosoft.com/oauth2/v2.0/token` with
`client_id`, `client_secret`, `grant_type=client_credentials` and
`scope=api://revenuestripe.onmicrosoft.com/partners/.default`. Send the token as
`Authorization: Bearer <access_token>`; it lasts about an hour.

Credentials come from your Jeeng account manager — API access is not self-serve.

## 2. Grain reports — `reporting-campaigns` / `reporting-campaign-lines`

```
GET https://powerinbox.azure-api.net/reporting/campaigns/{interval}?$filter=Date eq 2023-12-13Z&api-version=1.0
GET https://powerinbox.azure-api.net/reporting/lines/{interval}?$filter=Date eq 2023-12-13Z&api-version=1.0
```

- `{interval}` is `daily` or `hourly`.
- `$filter` is a **required** OData V4 expression. There is no "give me everything" call — always
  scope by date.
- `api-version=1.0` is required.

Query and response are OData V4. The campaigns report may return a `next` link; follow it until it
is absent. No page-size parameter is published.

Use `reporting-campaigns` for spend pacing across a campaign, and `reporting-campaign-lines` when
you need to see which line inside the campaign is carrying delivery.

## 3. Flexible report — `reporting-advertiser-performance`

```
GET https://powerinbox.azure-api.net/reporting/advertiserperformance
    ?requestedFields=DateTime,CampaignId,Impressions,Clicks,Spend
    &startInclusive=2026-03-01T00:00:00Z
    &endExclusive=2026-04-01T00:00:00Z
    &filter[CampaignId]=123
    &groupBy=CampaignId
    &api-version=1.0
Accept: text/csv
```

- `Accept: text/csv` is a **required header**.
- The date range is half-open and capped at **32 days**. Longer ranges are a 400 — loop month by
  month for longer histories.
- Every `groupBy` field must also be in `requestedFields`.
- Note the filter idiom differs from the grain reports: bracketed `filter[Field]=value` pairs here,
  an OData `$filter` expression there. They are not interchangeable.

The endpoint answers **303 See Other**; the `Location` header points at the CSV download. Most
clients follow it automatically.

## 4. Interpreting the results

- All timestamps are UTC.
- Email advertising is not web advertising: proxy opens (Apple, Gmail) inflate raw open counts, and
  Jeeng exposes MPP-adjusted figures for exactly this reason. Segment by device/proxy before drawing
  conclusions about performance.
- Hourly is the finest supported grain.
- Invalid Traffic (IVT) filtering is already applied.

## 5. Errors

| Status | Meaning | Do |
| --- | --- | --- |
| 400 | Invalid parameters | Check field names, `groupBy` ⊆ `requestedFields`, range ≤ 32 days, `$filter` syntax |
| 401 `Unauthorized` | No/expired token | Refresh |
| 401 `Not authorized to this endpoint.` | Wrong scope | Re-request with the partners `.default` scope — this endpoint is where a wrong-scope token usually first fails |
| 500 | Server error | Retry, then contact support@jeeng.com with the campaign id, line id and date range |

Errors are `{"statusCode": …, "message": …}` — not problem+json, and with no error code vocabulary.

## Notes

- No rate limits or `RateLimit-*` headers are published; cache results and avoid tight polling.
- To act on what you find, see `jeeng-manage-a-campaign-line`.
