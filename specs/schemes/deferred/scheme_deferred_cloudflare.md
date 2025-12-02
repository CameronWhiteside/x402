# Scheme: `deferred` `cloudflare`

## Summary

The `deferred` scheme on the Cloudflare network `cloudflare:com` enables access to resources through cryptographically signed payment commitments that are settled later through the network's infrastructure.

Payment requests are authenticated using **HTTP Message Signatures (RFC 9421)**, and network acts as a trusted intermediary to provide resource access while deferring actual payment settlement.

## Protocol Flow

The protocol flow for `deferred` on the network (Cloudflare) includes an initial setup step, followed by client-driven payment with optional pre-authorization:

### First-time Setup (Client)

1. Host public keys at a `.well-known` endpoint (e.g., `https://mycrawler.com/.well-known/web-bot-auth`
2. Submit signature agent URL to the `networkUrl` from the `http-message-signatures` extension (e.g., `https://dash.cloudflare.com/?to=/:account/configurations/verified-bots`)
3. The network (Cloudflare) associates signature agent URL with a billing identity for settlement

### Payment Flow (Per Request)

1. Makes an HTTP request to a **Resource Server**.
2. **Resource Server** responds with a `402 Payment Required` status. The response includes a PAYMENT-REQUIRED header (base64-encoded JSON) containing payment requirements with the `deferred` scheme and `cloudflare:com` network with `payTo` set to `merchant`. The response also includes the `http-message-signatures` extension indicating where to find documentation on associating the HTTP message signature agent with the network.
3. **Client** constructs a payment payload containing the payment commitment (amount, asset) and signs the HTTP request using **HTTP Message Signatures (RFC 9421)**. The client includes `Signature-Agent`, `Signature-Input`, and `Signature` headers along with the PAYMENT-SIGNATURE header.
4. **Client** sends a new HTTP request with the PAYMENT-SIGNATURE header (base64-encoded JSON) and HTTP Message Signature headers.
5. **Resource Server** verifies the signature agent is recognized by the network (Cloudflare) and fetches the public key.
6. **Resource Server** verifies the HTTP Message Signature is valid using the fetched public key.
7. **Resource Server** validates the payment amount and asset match the requirements.
8. **Resource Server**, upon successful verification, grants the **Client** access to the resource and includes a PAYMENT-RESPONSE header (base64-encoded JSON) confirming the payment commitment.
9. The network (Cloudflare) acts as Merchant of Record, handling deferred settlement and billing the identity associated with the signature agent.

**Pre-Authorized Flow**: Clients with pre-authorized payment agreements can include the PAYMENT-SIGNATURE header and HTTP Message Signature headers in their initial request (step 1), bypassing the 402 response and proceeding directly to verification and access.

**Note on HTTP Message Signatures**: All requests are signed using HTTP Message Signatures (RFC 9421). The signature agent's `.well-known` URL must be known to the network (Cloudflare) and associated with a billing identity for settlement.

## PaymentRequired for deferred

The `deferred` scheme on the network (Cloudflare) uses the standard x402 `PaymentRequired` fields with required `http-message-signatures` extension:

```http
HTTP/2 402 Payment Required
Content-Type: text/html
PAYMENT-REQUIRED: eyJ4NDAyVmVyc2lvbiI6IDIsICJlcnJvciI6ICJObyBQQVlNRU5ULVNJR05BVFVSRSBoZWFkZXIgcHJvdmlkZWQiLCAicmVzb3VyY2UiOiB7InVybCI6ICJodHRwczovL2V4YW1wbGUuY29tL2FydGljbGUiLCAiZGVzY3JpcHRpb24iOiAiUHJlbWl1bSBhcnRpY2xlIGNvbnRlbnQiLCAibWltZVR5cGUiOiAidGV4dC9odG1sIn0sICJhY2NlcHRzIjogWy4uLl19

<!DOCTYPE html>
<html>
  <head><title>Payment Required</title></head>
  <body>This content requires payment.</body>
</html>
```

**Decoded PAYMENT-REQUIRED header:**

```json
{
  "x402Version": 2,
  "error": "No PAYMENT-SIGNATURE header provided",
  "resource": {
    "url": "https://example.com/article",
    "description": "Premium article content",
    "mimeType": "text/html",
    "website": "https://example.com"
  },
  "accepts": [
    {
      "scheme": "deferred",
      "network": "cloudflare:com",
      "amount": "1",
      "asset": "USD",
      "payTo": "merchant",
      "maxTimeoutSeconds": 30
    }
  ],
  "extensions": {
    "http-message-signatures": {
      "info": {
        "networkUrl": "https://dash.cloudflare.com/?to=/:account/configurations/verified-bots",
        "signatureSchemes": ["ed25519"],
        "tags": ["web-bot-auth"]
      }
    },
    "license": {
      "info": {
        "format": "uri",
        "terms": "https://example.com/ppc/license.md"
      }
    }
  }
}
```

**PaymentRequirements fields:**

- `scheme`: Must be `"deferred"`
- `network`: Must be `"cloudflare:com"` (CAIP-2 format)
- `asset`: The asset identifier (e.g., `"USD"` for fiat currency - ISO 4217 format)
- `payTo`: Must be `"merchant"` (constant indicating the network handles settlement)
- `amount`: Payment amount in smallest unit of the asset (e.g., cents for USD)
- `maxTimeoutSeconds`: Maximum time allowed for payment completion

**Extensions:**

The `deferred` scheme requires the `http-message-signatures` extension and supports an optional `license` extension (see [scheme_deferred.md](./scheme_deferred.md#extensions) for full definitions):

- `extensions.http-message-signatures` (Required): Establishes cryptographic identity
  - `info.networkUrl`: URL to the network's setup endpoint (for Cloudflare: `https://dash.cloudflare.com/?to=/:account/configurations/verified-bots`)
  - `info.signatureSchemes`: Array of supported algorithms (for Cloudflare: `["ed25519"]`)
  - `info.tags`: Array of supported tags (for Cloudflare: `["web-bot-auth"]`)
- `extensions.license` (Optional): Establishes legal commitment bound to identity
  - `info.format`: Format identifier (e.g., `"uri"`, `"rsl"`, `"markdown"`)
  - `info.terms`: License terms as a string

Note: `payTo` is set to `"merchant"` as the network handles settlement.

## PAYMENT-SIGNATURE Header Payload

The client sends a PAYMENT-SIGNATURE header containing base64-encoded JSON with the payment payload.

```http
GET /article HTTP/2
Host: example.com
Signature-Agent: mycrawler.com
Signature-Input: sig=("@authority" "signature-agent" "payment-signature"); created=1700000000; expires=1700011111; keyid="ba3e64=="; tag="web-bot-auth"
Signature: sig=:abc123...==:
PAYMENT-SIGNATURE: eyJ4NDAyVmVyc2lvbiI6MiwicGF5bG9hZCI6eyJhbW91bnQiOiI1IiwiYXNzZXQiOiJVU0QifSwiYWNjZXB0ZWQiOnsic2NoZW1lIjoiZGVmZXJyZWQiLCJuZXR3b3JrIjoiY2xvdWRmbGFyZTpjb20iLCJhbW91bnQiOiI1IiwiYXNzZXQiOiJVU0QiLCJwYXlUbyI6Im1lcmNoYW50IiwibWF4VGltZW91dFNlY29uZHMiOjMwfX0=
```

**Decoded PAYMENT-SIGNATURE header:**

```json
{
  "x402Version": 2,
  "payload": {
    "amount": "5",
    "asset": "USD"
  },
  "accepted": {
    "scheme": "deferred",
    "network": "cloudflare:com",
    "amount": "5",
    "asset": "USD",
    "payTo": "merchant",
    "maxTimeoutSeconds": 30
  }
}
```

The `payload` field contains:

- `amount`: Payment amount in smallest unit of the asset (e.g., cents for USD)
- `asset`: Asset identifier (e.g., `"USD"`)

**Note**: The `signatureAgent` and `signature` are NOT in the payment payload. They come from the HTTP Message Signature headers (`Signature-Agent` and `Signature` headers) as defined in RFC 9421.

The `accepted` field contains the full `PaymentRequirements` object that this payment fulfills (without extensions, as those are at the top level).

**Note on Extensions**: According to v2 spec, clients must echo the extensions from the `PaymentRequired` response. However, extensions are not part of the `accepted` field since `accepted` contains only the specific `PaymentRequirements` object from the `accepts` array. Extensions would be echoed at the top level of the payment payload if the client needs to respond to them.

## Verification

Steps to verify a payment for the `deferred` scheme, the network (Cloudflare) implements:

1. **Verify HTTP Message Signature is valid**: Validate the HTTP Message Signature (RFC 9421) from the `Signature` header using the public key fetched from the `Signature-Agent` header URL
2. **Verify signature agent is recognized by the network**: Confirm the signature agent URL (from `Signature-Agent` header) is known to the network (Cloudflare) and associated with a billing identity
3. **Verify amount and asset are sufficient**: Ensure the payment amount and asset meet the requirements
4. **Verify timestamp freshness**: Ensure the payment is not expired (typically within 30 seconds)
5. **Verify accepted field matches**: Verify the `accepted` field matches one of the offered `PaymentRequirements`

### Verification Pseudocode

```javascript
function verifyDeferredPayment(paymentPayload, signatureAgentHeader, httpSignature) {
  // 1. Verify signature agent is recognized by the network and get billing info
  const agentInfo = await verifySignatureAgentWithNetwork(signatureAgentHeader);

  if (!agentInfo.isRecognized) {
    return { valid: false, reason: "signature_agent_unknown" };
  }

  // 2. Fetch public key from signature agent's .well-known endpoint
  const publicKey = await fetchPublicKey(signatureAgentHeader, agentInfo.keyId);

  if (!publicKey) {
    return { valid: false, reason: "server_error" };
  }

  // 3. Verify HTTP Message Signature using the public key
  const isSignatureValid = verifyHttpMessageSignature(
    httpSignature,
    publicKey,
    buildCanonicalString(paymentPayload)
  );

  if (!isSignatureValid) {
    return { valid: false, reason: "invalid_signature" };
  }

  // 4. Verify payment signature header is correctly structured
  if (!isValidPaymentSignature(paymentPayload)) {
    return { valid: false, reason: "invalid_payment" };
  }

  // 5. Verify amount/asset are sufficient for the resource
  const isSufficientPayment = checkPaymentSufficiency(
    paymentPayload.accepted.amount,
    paymentPayload.accepted.asset,
  );

  if (!isSufficientPayment) {
    return { valid: false, reason: "invalid_payment" };
  }

  // billingIdentifier is an arbitrary network identifier (e.g., account ID)
  // used by the network to rollup and bill charges for this signature agent
  return { valid: true, reason: null, billingIdentifier: agentInfo.billingIdentifier };
}
```

## Settlement

The network (Cloudflare) acts as Merchant of Record, aggregating payment commitments, billing the identity associated with each signature agent, and distributing revenue to content owners on a periodic basis through traditional off-chain financial rails.

## PAYMENT-RESPONSE Header

Upon successful verification, the server includes a PAYMENT-RESPONSE header (base64-encoded JSON) in the 200 OK response. The response includes the license extension to remind the client of the usage terms.

```http
HTTP/2 200 OK
Content-Type: text/html
PAYMENT-RESPONSE: eyJhbW91bnQiOiAiNSIsICJhc3NldCI6ICJVU0QiLCAibmV0d29yayI6ICJjbG91ZGZsYXJlOmNvbSIsICJ0aW1lc3RhbXAiOiAxNzMwODcyOTY4LCAiZXh0ZW5zaW9ucyI6IHsibGljZW5zZSI6IHsiaW5mbyI6IHsidXJsIjogImh0dHBzOi8vZXhhbXBsZS5jb20vcHBjL2xpY2Vuc2UubWQifX19fQ==

<!DOCTYPE html>
<html>
  <head><title>Premium Article</title></head>
  <body><article>Premium content...</article></body>
</html>
```

**Decoded PAYMENT-RESPONSE header:**

```json
{
  "amount": "5",
  "asset": "USD",
  "network": "cloudflare:com",
  "timestamp": 1730872968,
  "extensions": {
    "license": {
      "info": {
        "format": "uri",
        "terms": "https://example.com/ppc/license.md"
      }
    }
  }
}
```

The `license` extension in the response serves as a reference of the usage terms that apply to the the resource being delivered by the server.

## Appendix

### Network-Specific Implementation

The network (Cloudflare) implements the `deferred` scheme with the following details:

**Network URL**: `https://dash.cloudflare.com/?to=/:account/configurations/verified-bots`

This URL provides:

1. **Setup instructions**: How to submit your signature agent's `.well-known` URL to the network
2. **Billing identity association**: How to associate your signature agent with a billing identity for settlement
3. **Public key requirements**: What public key formats and algorithms are supported
4. **Verification process**: How the network verifies HTTP Message Signatures

**Supported Tags**: `["web-bot-auth", "agent-browser-auth"]`

**Setup Process**:

1. Client hosts their public keys at a `.well-known` endpoint (e.g., `https://mycrawler.com/.well-known/web-bot-auth`)
2. Client submits this URL to the network via the network URL endpoint
3. The network associates the signature agent URL with a billing identity
4. Client can now sign requests using HTTP Message Signatures, with the `Signature-Agent` header pointing to their `.well-known` URL
5. Resource servers verify signatures by fetching public keys from the `Signature-Agent` URL and validating the signature agent is known to the network

**Note**: For the full extension definitions (`http-message-signatures` and `license`), see the [deferred scheme specification](./scheme_deferred.md#extensions).

**Example HTTP Request with Message Signatures**:

```http
GET /article HTTP/2
Host: example.com
User-Agent: Mozilla/5.0 Chrome/113.0.0 MyCrawler/1.0
Signature-Agent: mycrawler.com
Signature-Input: sig=("@authority" "signature-agent" "payment-signature"); created=1700000000; expires=1700011111; keyid="ba3e64=="; tag="web-bot-auth"
Signature: sig=:abc123...==:

Payment-Signature: eyJ4NDAyVmVyc2lvbiI6IDIsIC4uLn0=
```

The `Signature-Agent` header indicates where to find the client's public keys (e.g., `mycrawler.com`). This URL must be known to the network (Cloudflare) and associated with a billing identity. The `networkUrl` in the extension points to the network's documentation on how to associate your signature agent.

### Security Considerations

**HTTP Message Signature Verification**:

- All requests must be signed using HTTP Message Signatures (RFC 9421)
- The signature **MUST** include the following components:
  - `@authority`: The target server authority
  - `signature-agent`: The signature agent header value
  - `payment-signature`: The PAYMENT-SIGNATURE header (lowercase per RFC 9421)
- This ensures the payment commitment is cryptographically bound to the HTTP request
- Servers verify signatures by:
  1. Fetching the public key from the URL in the `Signature-Agent` header
  2. Validating the signature using the fetched public key
  3. Confirming the signature agent URL is known to the network (Cloudflare)

**Billing Identity Association**:

- The signature agent URL (from `Signature-Agent` HTTP header) must be known to the network (Cloudflare)
- The network maintains the association between signature agent URLs and billing identities with account IDs
- The signature agent is identified by the `Signature-Agent` header

**Stateless Verification**:

- Servers do not need to maintain payment state or track attempts
- All verification can be performed stateless by validating the signature and billing identity association
- No database lookups or session management required

### Error Codes

Error codes for deferred payment failures on the Cloudflare network:

- `invalid_signature`: Invalid or missing `Signature-Input` or `Signature` headers
- `signature_agent_unknown`: Signature agent not recognized by network (Cloudflare)
- `invalid_payment_signature`: Invalid `Payment-Signature` header
- `payment_failed`: Signature agent not associated with valid billing identity
- `invalid_payment`: Payment amount does not match requirements
- `server_error`: Server error

### Pre-Authorized Access

Clients with pre-authorized payment agreements can include the PAYMENT-SIGNATURE header in their initial request, bypassing the 402 response:

```http
GET /article HTTP/2
Host: example.com
Signature-Agent: mycrawler.com
Signature-Input: sig=("@authority" "signature-agent" "payment-signature"); created=1700000000; expires=1700011111; keyid="ba3e64=="; tag="web-bot-auth"
Signature: sig=:abc123...==:
PAYMENT-SIGNATURE: eyJ4NDAyVmVyc2lvbiI6MiwicGF5bG9hZCI6eyJhbW91bnQiOiIxMDAwMCIsImFzc2V0IjoiVVNEIn0sImFjY2VwdGVkIjp7Ii4uLiJ9fQ==
```

If the payment is valid, the server responds directly with `200 OK` and the requested content, skipping the 402 negotiation phase.

### Comparison with Exact Scheme

| Feature          | Exact + EVM/SVM                  | Deferred + Cloudflare                   |
| ---------------- | -------------------------------- | --------------------------------------- |
| Settlement       | Immediate blockchain transaction | Deferred off-chain settlement           |
| Transaction Fees | Gas fees required                | No transaction fees                     |
| Currency         | Cryptocurrency (USDC, etc.)      | Fiat (USD, etc.)                        |
| Infrastructure   | Blockchain wallet required       | Cloudflare account required             |
| Trust Model      | Trustless blockchain             | Trusted Merchant of Record (Cloudflare) |
