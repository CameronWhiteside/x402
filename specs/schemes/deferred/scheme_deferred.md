# Scheme: `deferred`

## Summary

`deferred` is a payment scheme designed for agentic payments that do not require immediate settlement. Its primary purpose is to enable instant access to resources through cryptographically secure payment commitments using HTTP Message Signatures. These payments are settled later through trusted network infrastructure.

`deferred` payments eliminate transaction delays and gas fees while maintaining strong security through signature verification and account association with network providers, making them ideal for high-frequency, low-value transactions.

## Example Use Cases

### High-Volume, Asynchronous Micro-payments (Consolidated Settlement)

This scheme facilitates systems requiring continuous access for AI agents or automated crawlers. The individual, low-cost requests are granted instantly via cryptographic payment commitment. The server may later aggregate commitments and execute the financial settlement (which can use fiat, stablecoins, or traditional rails) is batched and consolidated to occur periodically (daily or weekly), eliminating per-request overhead and transaction fees.

### Subscription and Licensing Agreements (Pre-negotiated Access)

Agents can access resources immediately under pre-negotiated licensing or subscription terms. The scheme provides programmatic access verification (via signature and identity), while the financial settlement and any usage-specific legal terms (like LLM inference/training access) are managed separately through the trusted network's infrastructure association to the signature agent.

### Zero-Friction, Pay-Per-Use Access for Verified Identities

By binding the cryptographic payment commitment to a verified identity, the scheme enables instant delivery of premium content or API services. Similar to the exact scheme, this scheme supports simple access models like pay-per-article or per-API-call without forcing the user into a traditional API Key, subscription, or manual billing cycle.

## Appendix

### Extensions

The `deferred` scheme requires one extension and supports an optional extension:

#### HTTP Message Signatures Extension (Required)

The `http-message-signatures` extension establishes the **identity** of the paying agent through cryptographic signatures (RFC 9421). This extension is required for all deferred scheme implementations.

**Full Extension Definition:**

```json
{
  "http-message-signatures": {
    "info": {
      "networkUrl": "https://network.example.com/signature-agents",
      "signatureSchemes": ["ed25519", "ecdsa-p256-sha256", "rsa-pss-sha512"],
      "tags": ["web-bot-auth", "agent-browser-auth"]
    },
    "schema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "networkUrl": {
          "type": "string",
          "format": "uri",
          "description": "URL to the network's setup endpoint and documentation"
        },
        "signatureSchemes": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Supported cryptographic signature algorithms"
        },
        "tags": {
          "type": "array",
          "items": {
            "type": "string"
          },
          "description": "Supported signature tags for validation"
        }
      },
      "required": ["networkUrl"]
    }
  }
}
```

**Fields:**

- **`networkUrl`** (required): URL to the network's documentation and setup endpoint where signature agents can associate their identity with a billing identity
- **`signatureSchemes`** (required): Array of supported cryptographic algorithms (e.g., `["ed25519", "ecdsa-p256-sha256", "rsa-pss-sha512"]`)
- **`tags`** (required): Array of supported signature tags that identify the purpose (e.g., `["web-bot-auth"]`)

**Purpose**: Establishes the cryptographic identity of the paying agent and provides information on how to associate that identity with the network for billing.

#### License Extension (Optional)

The `license` extension establishes the **legal commitment** bound to the identity. This extension is optional and can be used when both parties need to explicitly communicate the terms of the payment commitment.

**Full Extension Definition:**

```json
{
  "license": {
    "info": {
      "format": "uri",
      "terms": "https://example.com/terms.md"
    },
    "schema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "format": {
          "type": "string",
          "enum": ["uri", "rsl", "markdown", "plaintext", "json"],
          "description": "Format identifier describing how to interpret the terms field"
        },
        "terms": {
          "type": "string",
          "description": "License terms as a string (URL, RSL, markdown, etc.)"
        }
      },
      "required": ["format", "terms"]
    }
  }
}
```

**Fields:**

- **`format`** (required): Format identifier describing how to interpret the `terms` field

  - `"uri"`: Terms field contains a URL or data URI to the license document
  - `"rsl"`: Terms field contains RSL (Responsible Source License) formatted text
  - `"markdown"`: Terms field contains Markdown formatted license text
  - `"plaintext"`: Terms field contains plain text license
  - `"json"`: Terms field contains JSON-stringified structured license data

- **`terms`** (required): License terms as a string
  - If `format` is `"uri"`: An HTTPS URL or data URI pointing to the license
  - If `format` is `"rsl"`, `"markdown"`, `"plaintext"`: The actual license text
  - If `format` is `"json"`: JSON string containing structured license data

**Purpose**: Establishes the legal agreement and terms that bind the payment commitment to the identity. This ensures both the paying agent and the resource provider understand the usage rights, obligations, and settlement terms.

**Examples**:

```json
// URI format
{ "format": "uri", "terms": "https://example.com/license.md" }

// RSL format
{ "format": "rsl", "terms": "RSL-1.0: commercial-use, modification | attribution, share-alike | liability, warranty" }

// Markdown format
{ "format": "markdown", "terms": "# License\n\nThis content is licensed under..." }
```

**Extension Requirements:**

- **Identity** (`http-message-signatures`) - **Required**: Outlines acceptable HTTP Message Signatures formats and the accepted tags that the server requires to identify the paying agent. This is essential for the network to associate payments with a billing identity.
- **Commitment** (`license`) - **Optional**: Defines the legal terms and obligations associated with that payment commitment. This can be used when explicit usage terms need to be communicated.

**How They Work Together:**

The `http-message-signatures` extension establishes the cryptographic identity of the paying agent, allowing the network to verify who is making the payment and associate it with a billing identity for settlement. When the optional `license` extension is included, it binds specific usage terms to that identity, creating a complete framework where the network knows both who is paying and what terms govern the usage of the accessed resource.
