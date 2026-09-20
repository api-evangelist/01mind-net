---
name: 01mind-net-render-document
description: Rehearse, then render a real .docx or .xlsx through 01Mind — dry-run the payload for free, then execute against the free monthly allowance or a purchased credit, and hand the base64 file to a person.
api: openapi/01mind-net-openapi.json
operations:
  - "GET /document-templates (no operationId in the provider's spec)"
  - "POST /document-templates (no operationId)"
  - "POST /sandbox/execute/{listingId} (no operationId)"
  - "POST /execute/{listingId} (no operationId)"
method: generated
generated: '2026-09-19'
grounding: >-
  The four routes above exist verbatim in openapi/01mind-net-openapi.json; none carries an operationId, so
  they are named by method and path. Limits, statuses and challenge strings are quoted from that contract and
  from conventions/, rate-limits/ and sandbox/ in this repo. Nothing here was invented.
---

# Render a document with 01Mind

The listingId is `render-document`. Base URL `https://01mind.net`. Everything is `application/json`.

## 0. Know what it costs before you start

- Five documents a month are free on an API key (`X-API-Key`), resetting on the 1st (UTC); after that each document is $0.25 USDC via x402. Without a key and without a purchase, `POST /execute/render-document` returns **402** with a body that names both paths.
- Request bodies up to 4 MB are accepted; larger returns **413** and nothing is consumed.
- Output is `.docx` or `.xlsx` only — no PDF. The file comes back as base64 in the same response; 01Mind keeps no copy and there is no download URL later.

## 1. Pick a shape, or a template

`GET /document-templates` (free, no key) lists the first-party templates — `invoice`, `report`, `meeting-minutes`, `statement-of-work` — each with the placeholders it needs. With `X-API-Key` you also see your own.

If you will send the same layout repeatedly, save it once with `POST /document-templates` (requires `X-API-Key`; free; does not use the allowance). A **409** means the templateId is taken or the per-key template limit is reached.

Otherwise send `format` plus **exactly one** of `blocks` (docx), `markdown` (docx), `sheets` (xlsx) or `templateId` + `values`. See `RenderDocumentInput` in the spec for the block types (`title`, `heading` level 1-2, `text`, `para`, `mono`, `table`).

## 2. Dry-run it — always

`POST /sandbox/execute/render-document` with the identical body. Free, no key, no payment. It runs the same validation as the paid call and returns the sanitised sheet names, block counts and every warning, without producing a file or consuming anything. A **400** here would have been a 400 on the paid route too; fix it now.

## 3. Execute

`POST /execute/render-document` with the same body, plus one of:

- `X-API-Key` header — draws on the free monthly allowance (or purchased credit tied to the key); or
- wallet proof in the body — `walletAddress`, `signedAt` (ISO 8601, within 10 minutes of the server clock) and `signature`, an EIP-191 personal_sign of exactly `01Mind: collect render-document as <walletAddress> at <signedAt>`. Each signature works once.

Read the result: `file` (base64), `filename`, `contentType`, `bytes`, `sha256` (keep it — it is the dispute anchor), `warnings[]` (every change the server made to your input), `paidWith` and the remaining-allowance counters.

## 4. Handle the failures the contract declares

- **400** — invalid input; nothing consumed. Re-check against the dry run.
- **401** — wallet proof failed (wrong message, expired `signedAt`, or a reused signature); the response names the exact message to sign. Sign a fresh challenge.
- **402** — no free allowance and no purchased credit for this wallet. Pay through x402 (retry with the payment header) or supply a key with allowance left.
- **403** — listing blocked pending legal review, or no settled purchase by a proven wallet.
- **413** — body or document over a stated limit; the message names it.
- **429** with `Retry-After` — gateway quota (per the provider's failure-mode catalog). Wait the stated time.

## Rules that are not in the status codes

- There is **no idempotency key**. A retry after an ambiguous timeout needs a fresh signature and, if the first call succeeded, consumes a second document. Prefer the dry run to reduce the chance of a paid failure, and log the `sha256` of anything you receive.
- A failed request does not use up a purchase (Terms of Sale 5.4). If you paid and received nothing, the remedy is an email to 01mind@01mind.net quoting the transaction (Terms 6.2) — there is no refund operation.
- Paying accepts the Terms of Sale in force at that moment (version 2026-09-15 as of this writing).
