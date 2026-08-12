---
name: jeeng-manage-a-campaign-line
description: Update an advertiser campaign line's daily spend goal and CPC/CPM/CPA pricing, and
  transition campaign lines and creatives between running, paused and archived on the Jeeng /
  OpenWeb Email Monetization API.
api: Jeeng Email Monetization — Advertisers API
generated: '2026-08-12'
method: generated
source: openapi/jeeng-advertisers-openapi.yml, openapi/jeeng-authentication-openapi.yml
operations:
  - getting-an-access-token
  - management-line-update
  - management-line-status
  - management-creative-status
---

# Manage a campaign line

The entire write surface of the Jeeng advertiser API is three operations: change a campaign line's
budget and pricing, change a campaign line's status, and change a creative's status. Everything
else — creating campaigns, targeting, inventory selection, discovery settings — is portal-only.

## Before you start

Credentials are provisioned by an account manager; there is no self-serve signup. You also need the
campaign line id and creative id, which come from the portal (Campaigns List → Campaign Details →
Campaign Lines). No API operation lists them.

## 1. Get an access token — `getting-an-access-token`

`POST https://login.microsoftonline.com/revenuestripe.onmicrosoft.com/oauth2/v2.0/token`,
form-encoded, with `client_id`, `client_secret`, `grant_type=client_credentials` and
`scope=api://revenuestripe.onmicrosoft.com/partners/.default`.

Send the result as `Authorization: Bearer <access_token>`; it lives about an hour. Requesting any
other scope yields `401 Not authorized to this endpoint.`

Every call below also requires `api-version=1.0` as a query parameter.

## 2. Update budget and pricing — `management-line-update`

`PUT https://powerinbox.azure-api.net/management/line/{id}?api-version=1.0`

```json
{
  "budget": 500,
  "pricing": { "cpc": 0.45 }
}
```

- `budget` is the **daily spend goal**, an integer — not a lifetime budget.
- `pricing` carries the bid. **Set exactly one** of `cpc` (price per click), `cpm` (price per 1000
  impressions) or `cpa` (amount for achieving CPA goals). Sending more than one is not supported.
- Switching a line's pricing model mid-flight changes how it competes for inventory; read
  "How Campaign Delivery Works" before doing it in bulk.

## 3. Pause, resume or archive a line — `management-line-status`

`POST https://powerinbox.azure-api.net/management/line/{id}/transition/{status}?api-version=1.0`

`{status}` is one of `running`, `paused`, `archived`. The target state is in the URL, so the call is
naturally repeatable — but note Jeeng publishes **no idempotency key and no idempotency guarantee**,
so treat retries as at-least-once and re-read state from a report rather than assuming a result.

`archived` is the terminal state in the published enum; there is no documented un-archive path, so
do not archive as a way of pausing.

## 4. Pause, resume or archive a creative — `management-creative-status`

`POST https://powerinbox.azure-api.net/management/creative/{id}/transition/{status}?api-version=1.0`

Same three states, same semantics, applied to a single creative rather than the whole line. Use this
to retire one underperforming creative without disturbing the line's delivery.

## 5. Verify the change

There is no GET for a campaign line. To confirm a change took effect, pull
`reporting-campaign-lines` (`GET /reporting/lines/{interval}` with `interval` = `daily` or `hourly`
and a required OData `$filter` such as `Date eq 2026-08-12Z`) and look at delivery after the change.

Email advertising does not react instantly: impressions accrue as recipients open mail, so allow at
least a full send cycle before judging a pricing or status change.

## Errors

Both management operations publish only `200` and `400`, and the published 400 schema is an empty
object — the failure reason is not machine-readable. Errors arrive as
`{"statusCode": …, "message": …}` (Azure API Management), not RFC 9457 problem+json.

| Status | Do |
| --- | --- |
| 400 | Check the id, that `api-version=1.0` is present, that `status` is one of the three enum values, and that only one `pricing` field is set |
| 401 `Unauthorized` | Refresh the token |
| 401 `Not authorized to this endpoint.` | Re-request the token with the partners `.default` scope |

## Notes

- No webhooks or callbacks exist; there is no way to be notified that a line changed state.
- No rate limits are published. Sequence bulk updates conservatively.
