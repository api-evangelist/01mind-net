---
name: 01mind-net-request-tool
description: Ask Charon, 01Mind's Tool Generation Engine, to build a Safe-tier micro-tool from a declared recipe — read the format guide, submit the recipe with real test parameters, poll the request, and report a production fault if the built tool misbehaves.
api: openapi/01mind-net-openapi.json
operations:
  - getToolRequestFormatGuide
  - getCatalogueMenu
  - submitToolGenerationRequest
  - getToolGenerationRequest
  - reportToolProductionFault
  - listCatalogueAdditions
  - getToolGenerationFaultLog
method: generated
generated: '2026-09-19'
grounding: Every operationId above exists verbatim in openapi/01mind-net-openapi.json; enums and rules are quoted from ToolGenerationRequestInput and the public format guide.
---

# Have Charon build a tool

Asking is free. A Safe-tier request is auto-built the moment a valid recipe is supplied; the finished tool becomes a purchasable catalogue listing priced like a comparable existing tool (around $0.003 USDC per call for a single micro-tool). Nothing is generated as code: a tool is a declared recipe — endpoints it may call, or arithmetic it may perform.

## 1. Read the contract before failing against it

- `GET /tool-requests/format-guide` (`getToolRequestFormatGuide`) — public; the exact recipe format, worked examples and the most common failure modes.
- `GET /charon/menu` (`getCatalogueMenu`) — public; what Charon already builds (Track A categories) and curated bundle ideas (Track B), with current prices.

## 2. Get a key, and visit Charon with it

`POST /tool-requests` requires `X-API-Key`. Keys are free and instant via `POST /keys` (documented by the provider in prose; not declared as a path in the OpenAPI). The provider's orientation states a request from a key that has not first visited `GET /charon` with that key attached is refused (401 without a key, 403 `MustVisitCharonFirst`). Note `/charon` is under a robots.txt Disallow as an agent-facing route reached by direct URL.

## 3. Write the recipe yourself — Charon will not

`ToolGenerationRequestInput` requires `requestingAgent` and `toolDescription`; `toolDescription` is used only for classification (Safe / Needs Approval / Unsafe) and naming, never turned into a recipe. `recipe` is one of:

- `{ "type": "math-expression", "expression": "input.celsius * 9 / 5 + 32" }` — every variable **must** be written `input.<name>`; a bare name is rejected outright.
- `{ "type": "api-call", "url": "https://...", "allowedParams": [...] }` — a real, live https endpoint, GET only, at most 20 allowed query params.
- `{ "type": "bundle", "steps": [{ "key": ..., "recipe": ... }, ...] }` — 2 to 8 leaf steps, no nesting, no step can read another's result.

Supply `testParams` with a real sample value for every `input.<name>` / allowed param (namespaced per step key for a bundle): Charon runs one real, live validation call against it before anything is listed.

## 4. Submit and read the decision

`POST /tool-requests` (`submitToolGenerationRequest`) returns **201** with `requestId`, `decision` — `AutoBuilt`, `PendingStewardApproval`, `RejectedUnsafe`, `ApprovedByStewart`, `RejectedByStewart` — and `registryTier` (`Safe`, `Needs Approval`, `Unsafe`). Safe work only: calculation, or reading from a source declared up front. Anything that writes, spends money, or reaches beyond what was declared is refused, not negotiated (failure mode FM-005).

Poll `GET /tool-requests/{requestId}` (`getToolGenerationRequest`) for `validationRegistryStatus` (`Pending`, `Passed`, `Failed`), `buildFeeCharged` and `requestCountForThisTool`. **404** means no such request.

## 5. After it is live

- `GET /tool-requests/catalogue-additions` (`listCatalogueAdditions`) — tools promoted into the catalogue after five built instances.
- If a built tool misbehaves in production, `POST /tool-requests/{requestId}/production-fault` (`reportToolProductionFault`) triggers Charon's self-repair path and marks the listing `Degraded`, which blocks purchase until re-verified (FM-003). Check `GET /listings/{listingId}/status` (undeclared route, live) for `Active`/`Degraded`.
- `GET /tool-requests/fault-log` (`getToolGenerationFaultLog`) — public log of `FaultRecord`s (severity `CRITICAL`/`STANDARD`, status `Open`/`Escalated`/`Resolved`).

There is no cancel or withdraw operation for a request, and no idempotency key — do not resubmit an in-flight request.
