---
name: getty-images-search-and-license
description: Find a Getty Images or iStock image or video that the caller's licence agreement actually covers, then license and download it — including the pre-flight checks that stop an agent from spending a licence twice.
api: Getty Images API v3
base_url: https://api.gettyimages.com/v3/
spec: openapi/_original/getty-images-swagger-v3-openapi.json
operations:
  - GET /v3/search/images/creative
  - GET /v3/search/images/editorial
  - GET /v3/search/videos/creative
  - GET /v3/search/videos/editorial
  - GET /v3/images/{id}
  - GET /v3/videos/{id}
  - GET /v3/downloads
  - POST /v3/downloads/images/{id}
  - POST /v3/downloads/videos/{id}
operation_id_note: >-
  The Getty Images OpenAPI 3.0.4 contract declares NO operationId on any of its 76
  operations, so every operation in this skill is addressed by METHOD + PATH, taken verbatim
  from the live spec at https://api.gettyimages.com/swagger/v3/swagger.json.
generated: '2026-09-12'
method: generated
source: https://developer.gettyimages.com/docs/gettingstarted/
---

# Search and license a Getty Images asset

## Before you start

- You need an **Api-Key** and, for anything that touches a download, an **access token**.
  Both are issued by a Getty Images account representative against an existing licence
  agreement. There is no self-service key and no sandbox — **your first call is production**.
- `POST /v3/downloads/...` is a **licence event**. It draws against the customer's agreement,
  it has **no idempotency key**, and there is **no reversal operation**. Treat it the way you
  would treat a payment.

## 1. Get an access token

```
POST https://authentication.gettyimages.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=<API_KEY>&client_secret=<API_SECRET>&grant_type=client_credentials
```

The response carries `access_token`, `token_type: Bearer` and `expires_in` (1800 seconds).
**Cache it and reuse it until it expires.** Token requests count against the account's rate
limit, so re-minting early makes throttling more likely, not less.

## 2. Search, asking for the download links up front

```
GET /v3/search/images/creative?phrase=<terms>&fields=largest_downloads&page=1&page_size=30
Api-Key: <API_KEY>
Authorization: Bearer <ACCESS_TOKEN>
```

- `fields=largest_downloads` is **not** returned by default. Neither is `download_sizes` nor
  `downloads`. Request them explicitly, and send the bearer token — Getty rejects those field
  values without one.
- An asset whose `largest_downloads` array is **empty** is outside the caller's agreement.
  Do not attempt to license it; the download will return 401.
- `enhanced_search` (natural-language matching) is **on by default**. If the caller needs
  literal term matching, send `enhanced_search=false`.
- Pagination is `page` / `page_size`. `page_size` accepts only 1, 2, 3, 4, 5, 6, 10, 12, 15,
  20, 25, 30, 50, 60, 75, 100. Compute the last page from `result_count`; asking for one past
  it returns `400 InvalidPage`, whose `ErrorMessage` names the real maximum.
- Editorial content lives on `/v3/search/images/editorial`; creative on `/creative`.
  `/v3/search/images` searches both.

## 3. Pre-flight: has this already been licensed?

There is no dry-run mode, so this read-only check is the only safeguard available:

```
GET /v3/images/{id}/downloadhistory
Api-Key: <API_KEY>
Authorization: Bearer <ACCESS_TOKEN>
```

or, for the account as a whole, `GET /v3/downloads`. If the asset already appears there, the
caller may be able to re-use the existing licence rather than draw a new one. Ask before
licensing again.

## 4. License and download

```
POST /v3/downloads/images/{id}?product_type=<product>&auto_download=false
Api-Key: <API_KEY>
Authorization: Bearer <ACCESS_TOKEN>
```

- `product_type` comes from the `largest_downloads[].downloads[].product_type` value in the
  search response (`easyaccess`, `premiumaccess`, `editorialsubscription`, …). Do not guess it.
- **Prefer the hypermedia URI.** The search response gives you
  `largest_downloads[].downloads[].uri` — POST to that rather than assembling the path.
- `auto_download=false` returns `{"uri": "https://delivery.gettyimages.com/..."}` instead of a
  302 straight into the bytes. Use it: it lets you record the delivery URI before fetching.
- The delivery URI is **opaque**. Do not parse it. Read `content-disposition` for the
  filename, `content-length` for the size and `content-type` for the media type.
- Follow redirects. Video download URLs return **307**, and the API uses 302 and 303
  elsewhere.

## 5. Handle the errors the way Getty documents them

| Status | Body | What to do |
|---|---|---|
| 400 | `{"ErrorCode":"InvalidPage", …}` | Clamp the page from `result_count` and retry once. |
| 401 | `{"message":"Unauthorized"}` | Missing/invalid `Api-Key`, an expired token, **or an asset outside the agreement**. Do not retry blindly — re-mint the token once, then stop. |
| 403 | `{"message":"Forbidden"}` | Permission problem. **Never retry** with the same credentials. |
| 404 | `{"ErrorCode":"ImageNotFound", …}` | Wrong id, or an image id used on a video route. |
| 429 | `{"message":"Too Many Requests"}` | Back off. There is **no `Retry-After`** and no `RateLimit-*` headers — only `X-Error-Detail: Account Over Queries Per Second Limit`. Use exponential backoff. |
| 500 | `{"message": …}` | Transient. Retry once after a short delay, then report to apisupport@gettyimages.com. |

Branch on `ErrorCode` when it is present. **Never** match on `ErrorMessage` wording — Getty
states explicitly that it is informational and may change.

## Rules that apply to every step

- `Api-Key` on **every** request, no exceptions.
- No idempotency key exists. If a `POST /v3/downloads/...` times out, **do not blind-retry** —
  call `GET /v3/downloads` first to find out whether it landed.
- Search responses are cached server-side (creative image search: 24 hours, `max-age=86400`).
  You cannot force freshness.
- Localise with the `Accept-Language` header. Do not send `GI-Country-Code` unless the account
  is explicitly permitted to — not all customers are.

See also: `conventions/getty-images-conventions.yml`,
`errors/getty-images-problem-types.yml`, `rate-limits/getty-images-rate-limits.yml`.
