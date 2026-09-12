---
name: getty-images-generate-image
description: Generate or modify a commercially safe image with Generative AI by Getty Images, then download it — with the credit, concurrency and permanence constraints an agent must respect.
api: Getty Images API v3
base_url: https://api.gettyimages.com/v3/
spec: openapi/_original/getty-images-swagger-v3-openapi.json
operations:
  - POST /v3/ai/enhance-prompt
  - POST /v3/ai/image-generations
  - POST /v3/ai/image-generations/refine
  - POST /v3/ai/image-generations/extend
  - POST /v3/ai/image-generations/object-removal
  - POST /v3/ai/image-generations/background-removal
  - POST /v3/ai/image-generations/background-replacement
  - POST /v3/ai/image-generations/background-generations
  - POST /v3/ai/image-generations/influence-color-by-image
  - POST /v3/ai/image-generations/influence-composition-by-image
  - POST /v3/ai/file-registrations
  - DELETE /v3/ai/file-registrations/{fileRegistrationId}
  - GET /v3/ai/image-generations/{generationRequestId}
  - GET /v3/ai/image-generations/{generationRequestId}/images/{index}/download-sizes
  - PUT /v3/ai/image-generations/{generationRequestId}/images/{index}/download
  - POST /v3/ai/image-generations/{generationRequestId}/images/{index}/variations
  - GET /v3/ai/generation-history
  - POST /v3/ai/redownloads
operation_id_note: >-
  The Getty Images OpenAPI 3.0.4 contract declares no operationId on any operation; these are
  addressed by METHOD + PATH from the live spec.
generated: '2026-09-12'
method: generated
source: https://developer.gettyimages.com/ai-generation/
---

# Generate and license an image with Generative AI by Getty Images

## Entitlement and cost — read first

- These endpoints are **restricted to accounts with an AI Generation licence product**.
  Without it you get 401/403, not a helpful message.
- Getty states plainly that using them **"may result in the deduction of a credit depending on
  the terms of your license."** There is **no credit-reversal operation** in the contract and
  **no idempotency key**. A retried generation POST is a second charge. Confirm with the
  caller before firing one.
- Modifying an existing creative-library image additionally requires a traditional licence
  (e.g. Premium Access) that covers downloading the original.
- The model is **Generative AI by Getty Images**, built on Bria Fibo Lite and trained on
  Getty's own licensed library for commercial safety. Model card:
  https://developer.gettyimages.com/ai-generation/model-card/

## 1. Optionally sharpen the prompt

```
POST /v3/ai/enhance-prompt
```

## 2. Register any input file you are supplying

Refine (inpainting), object removal and the influence-by-image operations take a client file —
a black-and-white **mask** (white marks the region to change) or a reference image.

```
POST /v3/ai/file-registrations
```

Keep the returned `fileRegistrationId`. When you are done,
`DELETE /v3/ai/file-registrations/{fileRegistrationId}` — it is the one Gen AI object with a
real delete.

## 3. Choose the operation

| Goal | Operation |
|---|---|
| Text to image | `POST /v3/ai/image-generations` |
| Fill a masked region ("inpaint") | `POST /v3/ai/image-generations/refine` |
| Expand beyond the borders ("outpaint") | `POST /v3/ai/image-generations/extend` |
| Remove an object behind a mask | `POST /v3/ai/image-generations/object-removal` |
| Remove / replace / generate a background | `.../background-removal`, `.../background-replacement`, `.../background-generations` |
| Match a reference image's palette and tone | `.../influence-color-by-image` |
| Match a reference image's pose and composition | `.../influence-composition-by-image` |
| More of a result you already have | `POST /v3/ai/image-generations/{generationRequestId}/images/{index}/variations` |

`POST /v3/ai/image-generations` accepts optional camera controls: `lens_type`
(`wide_angle`, `telephoto`) and `depth_of_field` (`shallow`, `deep`), plus `aspect_ratio`.
Read the live spec for the full parameter set rather than assuming — Getty ships new
parameters regularly and announces them only in the release notes.

Each generation returns **four images**, addressed by `(generationRequestId, index)`. There is
no id for an individual generated image.

## 4. Poll for the result, and mind the 410

```
GET /v3/ai/image-generations/{generationRequestId}
```

A generation request is **transient**. Once it expires:

```
HTTP/2 410
{"ErrorCode":"GenerationRequestGone","ErrorMessage":"The generation request with the given identifier is gone"}
```

**410 is permanent — never retry it.** Getty states the same request will always return 410.
The durable record is `GET /v3/ai/generation-history` (and
`GET /v3/ai/generation-history/{generationRequestId}`); persist the id there before you need it.

## 5. Download the one you want

```
GET  /v3/ai/image-generations/{generationRequestId}/images/{index}/download-sizes
PUT  /v3/ai/image-generations/{generationRequestId}/images/{index}/download
```

Check sizes first, then PUT to download. For something generated earlier, use
`POST /v3/ai/redownloads`.

## 6. 429 here means two different things

On `/v3/ai/image-generations/*`, a 429 is **either** the account's queries-per-second limit
**or** its limit on **concurrent pending generations** — and the response does not say which.
Getty's own diagnostic: if you know you are under your QPS limit, it is concurrency.

Their stated guidance is to **wait 1 second and call again**. There is no `Retry-After` header
to read. Cap your in-flight generations rather than relying on backoff alone; Getty recommends
a fault-handling library (Polly for .NET, Tenacity for Python).

## Compliance note

If you are placing this output on the EU market, Getty publishes the Article 53(1)(d) public
summary of training content at
https://developer.gettyimages.com/ai-generation/summary-of-training-content/, naming the EU
authorised representative and the upstream model dependency.

See also: `errors/getty-images-problem-types.yml`,
`rate-limits/getty-images-rate-limits.yml`, `conformance/getty-images-conformance.yml`.
