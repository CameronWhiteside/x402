# Extension: `http-message-signatures`

## Summary

The `http-message-signatures` extension establishes the **identity** of the paying agent through cryptographic signatures (RFC 9421). This extension is used by network implementations that authenticate payment commitments using HTTP Message Signatures.

## Purpose

Establishes the cryptographic identity of the paying agent and provides information on how to associate that identity with the network for billing.

## Extension Definition

```json
{
  "http-message-signatures": {
    "schema": {
      "$schema": "https://json-schema.org/draft/2020-12/schema",
      "type": "object",
      "properties": {
        "registrationUrl": {
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
      "required": ["registrationUrl"]
    },
    "info": {
      "registrationUrl": "https://network.example.com/signature-agents",
      "signatureSchemes": ["ed25519", "ecdsa-p256-sha256", "rsa-pss-sha512"],
      "tags": ["web-bot-auth", "agent-browser-auth"]
    }
  }
}
```

## Fields

- **`registrationUrl`** (required): URL to the network's documentation and setup endpoint where signature agents can associate their identity with a billing identity
- **`signatureSchemes`** (required): Array of supported cryptographic algorithms (e.g., `["ed25519", "ecdsa-p256-sha256", "rsa-pss-sha512"]`)
- **`tags`** (required): Array of supported signature tags that identify the purpose (e.g., `["web-bot-auth"]`)

## Usage

Networks that use HTTP Message Signatures for authentication include this extension in the `PaymentRequired` response to inform clients:

1. Where to register their signature agent with the network (`registrationUrl`)
2. Which cryptographic algorithms are supported (`signatureSchemes`)
3. Which signature tags are accepted for validation (`tags`)

The client must:

1. Host their public keys at a `.well-known` endpoint
2. Register their signature agent URL with the network via the `registrationUrl`
3. Sign HTTP requests using HTTP Message Signatures (RFC 9421) with the appropriate tag

## Example Networks

- **Cloudflare** (`cloudflare:com`): Uses this extension with `ed25519` signatures and `web-bot-auth` tag

## Example

```json
{
  "extensions": {
    "http-message-signatures": {
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "registrationUrl": { "type": "string", "format": "uri" },
          "signatureSchemes": { "type": "array", "items": { "type": "string" } },
          "tags": { "type": "array", "items": { "type": "string" } }
        },
        "required": ["registrationUrl"]
      },
      "info": {
        "registrationUrl": "https://dash.cloudflare.com/?to=/:account/configurations/verified-bots",
        "signatureSchemes": ["ed25519"],
        "tags": ["web-bot-auth"]
      }
    }
  }
}
```
