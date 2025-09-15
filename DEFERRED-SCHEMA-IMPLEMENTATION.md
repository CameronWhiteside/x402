> write a full .md file i can use to add this new deferred payment schema to x402 github repo. it should include a summary and implementation order of all changes. break things up for modular delivery and incremental (e.g. name changes, cloudflare network, deferred-payment schema etc.) Use mostlyt eh spec, but also incporate the following feedback:
>
> 1. Keep in mind response headers can be a good option as they are more easily available to infrastructure and leavce the body free to communicate something to the user.
> 2. X-HEADERS ARE STRONGLY DISCOURAGED AND WILL CAUSE ISSUES DOWN THE ROAD, SEE RFC 6648
> 3. BASE64 ENCODED JSON IN A HEADER IS VERY PROBLEMATIC AS IT can be a factor for atta cks. Try a strctured instead; see https://httpwg.org/spec/rfc9651.html many implementations available becomeing widely used for new headers
> 4. Cloudflare cannot store state of 402 requests. It's an attack vector to spam and create a lot of records. incorprate that in nonce consideration and make sure client can manage any state at all
> 5.

# x402: Deferred Payment Scheme for the Cloudflare Network

> Adds a **`deferred-payment`** scheme and **`cloudflare`** network to x402, using HTTP structured fields for price discovery and purchase agreements, avoiding `X-` headers and JSON-in-headers, and requiring **client-managed state** (nonces/idempotency) to fit Cloudflare’s stateless 402 constraints. See Cloudflare Pay-Per-Crawl header semantics for compatibility references, and RFC 9651 for the structured-field encoding rules.

---

## Summary

This change extends the x402 **Payment Required Response** and **Payment Payload** to support a **deferred settlement** model operated by Cloudflare at the network edge. It keeps price discovery and charge confirmations in **HTTP response headers** (fast for infra; body remains free for user messaging), and it defines **new, non-`X-` structured headers** for both the 402 response and the purchase retry. It **does not** require Cloudflare to store 402 state; instead, the client must generate and track **nonces/idempotency keys**, which are verified statelessly at the edge.

This design aligns x402 with Cloudflare’s Pay-Per-Crawl (PPC) behavior (e.g., `crawler-price`, `crawler-charged`) while modernizing the wire format toward the HTTPWG’s Structured Fields standard (RFC 9651).

> Context & prior art: “Unifying x402 with Cloudflare Pay Per Crawl” PRD proposed the high-level schema, renaming `outputSchema`→`structuredSchema`, and the notion of a `deferred-payment` scheme with a `cloudflare` network. This document turns that into a spec-ready patch with concrete header definitions and migration steps.

---

## Goals & Non-Goals

**Goals**

- Introduce `scheme: "deferred-payment"` with `network: "cloudflare"` into x402.

- Use **response headers** for price discovery & charge receipts; keep 402 body human-friendly.
- Replace ad-hoc/JSON/base64 headers with **RFC 9651 Structured Fields** for new fields.

- Avoid `X-` header names (per community guidance; see RFC 6648 deprecation rationale).
- Enforce **client-managed state**: nonce/idempotency without Cloudflare storing 402 attempts.

**Non-Goals**

- Defining blockchain rails or custody; Cloudflare is **Merchant of Record** in PPC flows.

- Replacing existing PPC headers immediately; we maintain **backwards compatibility** first, then migrate.

---

## Terminology

- **PPC**: Cloudflare Pay-Per-Crawl; current headers: `crawler-price` (402) and `crawler-charged` (200).

- **Structured Fields (SF)**: RFC 9651 data model for HTTP header values (Items, Lists, Dictionaries, Binary).

---

## Wire Format Design

### 1) 402 Response (Price Discovery)

Servers MAY continue to emit legacy PPC headers for compatibility, **and MUST** emit the **new structured header** below.

**New Response Header (Structured Dictionary)**

- **`Pay-Requirements`**: SF **Dictionary** with a `deferred-payment` entry.

Example (402 Payment Required):

```
HTTP/2 402
date: Fri, 06 Jun 2025 08:42:38 GMT
content-type: text/plain; charset=utf-8

# New structured header (RFC 9651)
Pay-Requirements: deferred-payment=(network=cloudflare;amount=0.01;currency=USD;
                                  resource="https://example.com/page";
                                  mime="text/html";terms="https://example.com/terms";
                                  schema=?0)

# Optional: legacy PPC header for compatibility
crawler-price: USD 0.01
```

Notes:

- `amount` is Decimal; `currency` is a Token (`USD`). (SF types per RFC 9651.)

- `schema` is Boolean indicating presence of a body schema; detailed structure is carried in **body JSON** if needed (see below).
- **No `X-` prefix** anywhere.
- Keep body available for UX copy and optional x402 **Payment Required Response** JSON if rich metadata is needed.

**Optional 402 Body (x402 JSON)**

```json
{
  "paymentRequirements": [
    {
      "scheme": "deferred-payment",
      "network": "cloudflare",
      "maxAmountRequired": 10000,
      "resource": "https://example.com/page",
      "description": "Article: Example",
      "mimeType": "text/html",
      "structuredSchema": {
        "termsOfServiceUrl": "https://example.com/terms",
        "usagePolicy": "Single-view; no model training"
      }
    }
  ]
}
```

- `outputSchema` is **renamed** to `structuredSchema`.

### 2) Paid Access Retry (Agreement Proof)

Clients reattempt with bot identity (Cloudflare **Web Bot Auth** signature suite) **and** a **new structured header** expressing agreement.

**Required Request Headers**

- Web Bot Auth: `signature-agent`, `signature-inputs`, `signature` (as today).

- **`Pay-Agreement`** (SF **Dictionary**) carrying client agreement & anti-replay material.

Example (200 OK):

```
GET /page HTTP/2
signature-agent: ...           ; from Web Bot Auth
signature-inputs: ...          ; from Web Bot Auth
signature: ...                 ; from Web Bot Auth
Pay-Agreement: deferred-payment=(network=cloudflare;price=0.01;currency=USD;
                                 ts=1730872958;nonce="b2t-7Gt5...";client=token:cf_bot_123;
                                 agree=:MIIC...SIG...:;charge-id=:Af3...IDEM...:)
```

- `agree` is SF **Byte Sequence** (colon-base64, not JSON) containing a signature over: method, URL, `price`, `currency`, `ts`, `nonce`, and (if present) terms. (Avoids JSON-in-header, uses RFC 9651 binary type.)

- `charge-id` is a deterministic SF Byte Sequence (e.g., hash of `{agent-id, resource, ts, nonce, price}`) allowing **idempotent** server replies **without server state**.
- `nonce` uniqueness is **client-enforced**; server validates freshness (`ts` window) and signature binding but does not store seen nonces.

**Server Response (Success)**

```
HTTP/2 200
crawler-charged: USD 0.01
Pay-Result: charge-id=:Af3...IDEM...:;amount=0.01;currency=USD
```

- `Pay-Result` is a new SF **Dictionary** summarizing billing outcome for machine parsing; legacy `crawler-charged` remains for compatibility.

**Server Response (Mismatch)**

```
HTTP/2 402
Pay-Requirements: deferred-payment=(network=cloudflare;amount=0.02;currency=USD; ...)
crawler-price: USD 0.02
```

---

## Anti-Replay, Idempotency, & Statelessness

Cloudflare **cannot** store state for 402 requests; storing every attempt would be an abuse vector. Therefore: **the client must manage state** (nonces, budgets, idempotency).

- **Client** responsibilities:

  - Generate a unique `nonce` per attempt; track consumed nonces locally.
  - Enforce budgets and spending limits internally (per PPC guidance).

  - Treat repeated `Pay-Result.charge-id` as idempotent (no double-spend).

- **Server** (stateless) checks:

  - Valid **Web Bot Auth** identity and request signature.

  - `agree` signature binds `(method, url, price, currency, ts, nonce[, terms])`.
  - `ts` is within a short freshness window; `nonce` format is valid (but not stored).
  - Recomputes `charge-id` deterministically from signed fields to produce a stable receipt without storage.

---

## Backwards Compatibility

- **Coexistence** period: keep emitting `crawler-price`/`crawler-charged` while introducing `Pay-Requirements` / `Pay-Result`.

- Clients MAY continue using PPC “max price then 200/402” probing; x402 adds a structured path to formalize the purchase agreement and receipt semantics.

---

## Field Definitions (IANA/Registry Text)

All new fields use **RFC 9651 Structured Fields**.

- **`Pay-Requirements`** (Response, Dictionary)

  - Key: `deferred-payment` → Inner Dictionary with:
    - `network`: Token. MUST be `cloudflare`.
    - `amount`: Decimal. Price required.
    - `currency`: Token (e.g., `USD`).
    - `resource`: String (absolute or request URL).
    - `mime`: String (e.g., `"text/html"`).
    - `terms`: String (URL).
    - `schema`: Boolean (whether the 402 body includes x402 metadata).

- **`Pay-Agreement`** (Request, Dictionary)

  - Key: `deferred-payment` → Inner Dictionary with:
    - `network`: Token. MUST be `cloudflare`.
    - `price`: Decimal; `currency`: Token.
    - `ts`: Integer (Unix time).
    - `nonce`: String (client-unique).
    - `client`: Token (agent id; optional).
    - `agree`: Byte Sequence (signature).
    - `charge-id`: Byte Sequence (deterministic idempotency key).

- **`Pay-Result`** (Response, Dictionary)
  - `charge-id`: Byte Sequence.
  - `amount`: Decimal; `currency`: Token.

> **Why Structured Fields vs JSON-in-headers?** To avoid parsing ambiguity, normalize types (Decimals, Tokens, Binary), and match modern HTTP guidance.

---

## JSON Body Schema Changes (x402)

- Add `scheme: "deferred-payment"` and `network: "cloudflare"` to `paymentRequirements[]`.

- **Rename** `outputSchema` → **`structuredSchema`** (spec + SDKs + fixtures).

- Allow optional `termsOfServiceUrl` / `usagePolicy` under `structuredSchema`.

---

## End-to-End Flow (Happy Path)

1. **Client → GET /resource**
2. **Server → 402** with `Pay-Requirements` (and optional body), may also include `crawler-price` for legacy.

3. **Client** builds `Pay-Agreement` with fresh `nonce`, `ts`, and `agree` signature (also sends Web Bot Auth headers).

4. **Server** verifies, computes `charge-id` deterministically, settles deferred charge, returns **200 + `Pay-Result`** (and `crawler-charged`).

---

## Security Considerations

- **No `X-` headers**; they cause long-term interoperability issues (see feedback & general IETF guidance).
- **No Base64-encoded JSON in headers**; use **SF Byte Sequences** for signatures/IDs (colon-base64, typed).

- **Client state**: replay prevention is achieved by `(nonce, ts)` inside `agree` + deterministic `charge-id`.
- **Agent identity**: rely on Web Bot Auth signatures to bind bot identity at request time.

---

## Related Work

- Cloudflare PPC: crawler pricing/charging headers and crawler onboarding & signatures.

- HTTP Structured Fields (successor to RFC 8941).

- Trusted agentic commerce discussions (market direction for agent/payment standards).

---

## Implementation Plan (Modular, Incremental)

### Phase 0 — Naming & Types (small PRs)

1. **Spec**: Rename `outputSchema` → `structuredSchema` everywhere (spec, examples).

2. **SDKs**: Update types/validators/generators; add deprecation warnings for `outputSchema`.
3. **Test Vectors**: Add fixtures with `structuredSchema`.

### Phase 1 — Headers & Parsers (compatible)

1. **Define new SF headers** in the spec: `Pay-Requirements`, `Pay-Agreement`, `Pay-Result`.
2. **Reference RFC 9651** and include ABNF-like prose for allowed parameters & types.

3. **Reference PPC** for legacy coexistence; document mapping between `crawler-price`/`crawler-charged` and new fields.

4. **Implement parsers/serializers** (language SDKs) for the new headers.

### Phase 2 — `deferred-payment` (cloudflare) scheme

1. **Spec**: Add `scheme: "deferred-payment"`, `network: "cloudflare"`; define server expectations and client responsibilities (nonces, ts, agree).

2. **Edge behavior**: Document stateless verification and deterministic `charge-id`.
3. **SDK**: Helper to build `agree` payload (canonical string → signature), `charge-id` computation, nonce utilities.

### Phase 3 — Wire Examples & Conformance

1. **Golden flows**: 402→retry→200 with and without optional body JSON; with legacy PPC headers present.

2. **Negative tests**: price mismatch, stale `ts`, malformed `nonce`, invalid signature, different currency.
3. **Interop suite**: CLI fixtures and Postman collections.

### Phase 4 — Migration & Deprecation

1. Encourage integrators to **read new SF headers** first; **fall back** to legacy PPC if absent.
2. After adoption, Cloudflare MAY mark legacy headers as “deprecated for new features” (spec note only; no breaking change).

---

## Example Canonicalization for `agree`

Pseudo-input to sign (UTF-8, LF):

```
method: GET
url: https://example.com/page
price: 0.01
currency: USD
ts: 1730872958
nonce: b2t-7Gt5...
terms: https://example.com/terms
```

- Signature algorithm/profile is referenced from Web Bot Auth bot identity.

- The result is put into `agree` as an SF **Byte Sequence** (colon-base64).

---

## Spec Diffs (for PR reviewers)

- **Add**: `scheme = "deferred-payment"`; **Add**: `network = "cloudflare"`.

- **Rename**: `outputSchema` → `structuredSchema`.

- **Define**: `Pay-Requirements` (Response, SF Dict), `Pay-Agreement` (Request, SF Dict), `Pay-Result` (Response, SF Dict).

- **State**: Clients MUST manage nonce/idempotency; servers MUST remain stateless for 402.
- **Compat**: Map PPC headers → x402 fields; examples included.

---

## Appendix A — PPC Mapping Reference

- `crawler-price: USD 0.01` ⇢ `Pay-Requirements: deferred-payment; amount=0.01; currency=USD` (and body JSON if needed).

- `crawler-charged: USD 0.01` ⇢ `Pay-Result: amount=0.01; currency=USD`.

---

## Appendix B — Motivation & Ecosystem

Industry momentum is building around **agentic commerce** and cryptographically authenticated machine clients. Aligning x402 with **network-level monetization** at the edge (PPC) provides distribution and safety while keeping the protocol **open and vendor-neutral**. (See agentic commerce proposals for broader context.)

---

_Reviewed materials: Cloudflare Pay-Per-Crawl developer guide (headers, flows), HTTP Structured Fields (data model), and prior x402↔Cloudflare unification PRD._
