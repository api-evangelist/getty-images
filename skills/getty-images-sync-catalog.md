---
name: getty-images-sync-catalog
description: Keep a local copy of the Getty Images catalogue in sync using the Asset Changes pull loop — the contractually required path for removing deleted assets from a downstream system.
api: Getty Images API v3
base_url: https://api.gettyimages.com/v3/
spec: openapi/_original/getty-images-swagger-v3-openapi.json
operations:
  - GET /v3/asset-changes/channels
  - PUT /v3/asset-changes/change-sets
  - DELETE /v3/asset-changes/change-sets/{change-set-id}
  - GET /v3/images/{id}
  - GET /v3/videos/{id}
operation_id_note: >-
  The Getty Images OpenAPI 3.0.4 contract declares no operationId on any operation; these are
  addressed by METHOD + PATH from the live spec.
generated: '2026-09-12'
method: generated
source: https://developer.gettyimages.com/asset-change/
---

# Sync a downstream catalogue with Asset Changes

## Why this matters

Getty publishes **no webhooks**. Asset Changes is a client-pull change feed, and it is how a
platform storing Getty content stays current. Getty is explicit that for most clients
**removing deleted assets is a contractual obligation** — this is not an optimisation.

## 1. Read your channels once, then cache them

```
GET /v3/asset-changes/channels
Api-Key: <API_KEY>
Authorization: Bearer <ACCESS_TOKEN>
```

An account has **up to four** channels, one per (asset type x family):

- Editorial Images
- Creative Images
- Editorial Film
- Creative Film

Which channels you get is governed by the bundles in the account's agreement. Getty's guidance
is to call this **once at application startup and cache the result** — channels do not change
after they are established.

> Since **March 2025** channels are no longer split by lifecycle. Older integrations that
> expected separate new / update / delete channels per asset type must be reworked to one
> channel per asset type and family.

## 2. The loop

Per channel, repeat:

1. **Request a batch.**
   ```
   PUT /v3/asset-changes/change-sets
   Api-Key: <API_KEY>
   Authorization: Bearer <ACCESS_TOKEN>
   ```
   with the channel id and a batch size. The response carries the changes and a
   **change set identifier**.

2. **Process every change** on your side (see step 3).

3. **Confirm the batch.**
   ```
   DELETE /v3/asset-changes/change-sets/{change-set-id}
   ```

4. If the batch came back **empty**, you are caught up — sleep briefly before looping.

**The ack is the flow control.** New changes are not returned from
`PUT /v3/asset-changes/change-sets` until the current change set is confirmed. If you do not
DELETE, the next PUT returns **the same batch again**. That is the system's safety property:
a crash mid-batch costs you a re-delivery, never a skipped change. It is also the closest
thing this API has to idempotency — there is no `Idempotency-Key` header anywhere in the
Getty contract, so do not look for one.

## 3. Apply each change by `asset_lifecycle`

| `asset_lifecycle` | Meaning | What to do |
|---|---|---|
| `New` | Newly available under the agreement | **Upsert** |
| `Update` | Metadata changed (usually keywords and captions) | **Upsert** |
| `Delete` | Removed from the Getty catalogue or no longer usable | **Remove from your system** |

Two hard rules from Getty's own documentation:

- **Treat `New` and `Update` identically, as an upsert.** Messages are ordered chronologically
  within a channel, but Getty does not track per-client asset state, so an `Update` can arrive
  before the `New` for the same asset.
- **Silently ignore a `Delete` for an asset you have never seen.** Deletes are emitted for
  assets a client may never have had access to. This is expected, not an error.

Design for **multiple messages about one asset in a short span**, and make the processor
order-independent.

## 4. Operational notes

- Run one loop per channel; they are independent.
- Every request needs `Api-Key`, and the change-set operations need a bearer token.
- On `429`, back off — there is no `Retry-After` header to read, only a
  `X-Error-Detail: Account Over Queries Per Second Limit` on the QPS path. Batch size is the
  lever: fewer, larger batches cost fewer queries per second.
- On `500`, retry the PUT. Because the batch is not released until you DELETE it, a retry is
  safe by construction.
- Watch https://developer.gettyimages.com/status/ — the incident log records past occasions
  (2024-03-20, 2024-07-25) where editorial notifications stopped flowing to Asset Changes
  channels entirely. An empty batch is normal; **days** of empty batches on a live channel is
  not, and the status page is the only place that will tell you.

See also: `conventions/getty-images-conventions.yml` (event_surface),
`data-model/getty-images-data-model.yml` (AssetChangeSet).
