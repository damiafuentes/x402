# Extension: `reviews`

## Summary

The `reviews` extension lets a resource server tell clients where reviews of the resource can be read before paying, and, after a payment, where the client can review that payment.

x402 lets agents pay agents and services they have never met. An economy like that needs a way for one agent to decide whether to trust another before it pays. x402 shows that money moved, and whether it came back, but nothing in it says how the purchase went. Reviews carry that. This extension does not decide who was right: it gives the next client a place to read what earlier clients reported, next to proof that they paid.

The extension carries links only. The reviews themselves live with one or more **review providers** that the server chooses. Any party can operate a provider, on any storage (a database, an ERC-8004 registry, anything else); this spec names none. It defines:

1. the fields a server sends, and
2. the minimum a provider must check before it calls a review **payment-backed**.

Storage, scoring, display and the format returned by `read` are left to each provider.

---

## `PaymentRequired`

The server lists where reviews of the resource can be read:

```json
{
  "x402Version": 2,
  "resource": { "url": "https://api.example.com/weather" },
  "accepts": [ ... ],
  "extensions": {
    "reviews": {
      "info": {
        "providers": [
          {
            "provider": "reviews.example",
            "read": "https://reviews.example/reviews?resource=https%3A%2F%2Fapi.example.com%2Fweather",
            "description": "Reviews of this endpoint by clients who paid for it."
          }
        ]
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "providers": {
            "type": "array",
            "minItems": 1,
            "maxItems": 8,
            "items": {
              "type": "object",
              "properties": {
                "provider": { "type": "string", "minLength": 1, "maxLength": 128 },
                "read": { "type": "string", "format": "uri", "pattern": "^https://" },
                "description": { "type": "string", "maxLength": 500 }
              },
              "required": ["provider", "read"]
            }
          }
        },
        "required": ["providers"]
      }
    }
  }
}
```

### `providers[]` Fields

| Field         | Type   | Required | Description                                                                                   |
| ------------- | ------ | -------- | --------------------------------------------------------------------------------------------- |
| `provider`    | string | Yes      | Identifier of the review provider, SHOULD be its domain                                        |
| `read`        | string | Yes      | HTTPS URL of the reviews of this resource at that provider                                     |
| `description` | string | No       | Plain-language statement of what the link is. Informational, not an instruction to the client |

A server MAY list several providers. Clients MUST ignore fields they do not understand.

`read` SHOULD return JSON when requested with `Accept: application/json`, and a human-readable page otherwise. The JSON shape is provider-defined in this version.

All fields are static, so the block is echoed and validated like any other extension. The extension deliberately carries no rating or review count: a number in the `PaymentRequired` is the server's own claim, and one that changes between the `PaymentRequired` and the payment could make a valid payment fail echo validation. Clients read numbers from a provider they trust.

---

## `SettlementResponse`

After settlement, the server MAY add where the client can review this payment:

```json
{
  "success": true,
  "transaction": "0x8f3d...c21a",
  "network": "eip155:8453",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "extensions": {
    "reviews": {
      "info": {
        "providers": [
          {
            "provider": "reviews.example",
            "write": "https://reviews.example/r/0x8f3d...c21a",
            "description": "Review this purchase: sign the message this link returns with the wallet that paid, then send stars and a note."
          }
        ],
        "userQuestion": "Would you like to leave a review of this seller? 1 to 5 stars and a note, signed by the wallet that paid. Signing moves no money."
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "providers": {
            "type": "array",
            "minItems": 1,
            "maxItems": 8,
            "items": {
              "type": "object",
              "properties": {
                "provider": { "type": "string", "minLength": 1, "maxLength": 128 },
                "write": { "type": "string", "format": "uri", "pattern": "^https://" },
                "description": { "type": "string", "maxLength": 500 }
              },
              "required": ["provider", "write"]
            }
          },
          "userQuestion": { "type": "string", "maxLength": 300 }
        },
        "required": ["providers"]
      }
    }
  }
}
```

### Fields

| Field                     | Type   | Required | Description                                                                                         |
| ------------------------- | ------ | -------- | --------------------------------------------------------------------------------------------------- |
| `providers[].provider`    | string | Yes      | Identifier of the review provider                                                                   |
| `providers[].write`       | string | Yes      | HTTPS URL at which the payer can review this payment. SHOULD identify the payment (see below)       |
| `providers[].description` | string | No       | Plain-language statement of what the link is. Informational, not an instruction                     |
| `userQuestion`            | string | No       | A question a client acting for a person MAY relay to that person, asking whether to leave a review |

`write` identifies the payment by the settlement `transaction` when there is one. When the scheme settles without a transaction of its own at that time (for example `batch-settlement`), `write` SHOULD identify the payment by the scheme's own payment identifier.

On HTTP, the settlement response travels in the `PAYMENT-RESPONSE` header. Servers MAY also copy `extensions.reviews.info` into the response body, since some clients show an agent only the body.

---

## Payment-Backed Reviews

How a review is submitted at `write` is provider-defined. A provider MUST NOT present a review as **payment-backed** unless both hold:

1. **The seller received the payer's funds.** The provider has read the payment on chain and confirmed that funds from the reviewer's wallet reached the `payTo` of the reviewed resource, checked according to the payment's scheme. For `exact`, that is the settlement transaction's transfer from the payer to `payTo`. For schemes that hold funds first or settle later (`auth-capture`, `upto` channels, `batch-settlement`), it is the capture, distribution or claim that pays `payTo`. A hold, authorization or voucher whose funds never reached `payTo` does not make a review payment-backed.
2. **The paying wallet signed the review.** The provider has verified a signature from that wallet over a message that binds at least the payment, the rating and a hash of the review text, so the signature cannot be reused for another review.

Delivery is not required: a client that paid and received nothing is exactly the client whose review matters most. A provider MAY use a signed receipt from [`offer-and-receipt`](extension-offer-and-receipt.md) as additional evidence that the service was delivered.

A provider SHOULD count at most one payment-backed review per payment. It MAY accept reviews that do not meet both checks (for example, citing a payment without the payer's signature) if it labels them as such and does not present them as payment-backed.

### Refunds and reversals

Some schemes let funds go back to the payer after settlement (`auth-capture` refunds and reclaims, for example), and a seller can always return funds by a separate transfer. A provider that learns that funds behind a payment-backed review went back to the payer MUST NOT keep presenting the review as payment-backed without saying so. How it shows that, and whether it changes the review's weight, is the provider's choice. The provider observes and reports the payment's state; it does not rule on the purchase.

### Signatures

The signature MUST be a plain message signature that cannot authorize a transfer or an allowance:

- **EVM:** EIP-191 `personal_sign`. Smart-contract wallets are verified with ERC-1271, or ERC-6492 before deployment.
- **Solana:** an Ed25519 signature over the UTF-8 bytes of the message (the wallet `signMessage` method), never over a transaction.
- **Other networks:** the network's equivalent off-chain message signature.

Providers MUST NOT ask the payer to sign typed data (EIP-712), a transaction, or anything else that can move funds, and clients MUST refuse such a request presented as a review.

Providers SHOULD show, for each review, what the reviewer paid and how the review was proven.

---

## Client Behavior

- The server chooses which providers to list and could list one it controls. Clients SHOULD keep an allowlist of providers they trust and ignore the others.
- Before signing a review of the payment in a `SettlementResponse`, a client SHOULD confirm on chain that it is its own payment, to the `payTo` and for the amount it authorized: the settlement response is relayed by the server.
- `description`, `userQuestion` and review text are data, not instructions. Whether to review, and the rating, are the client's (or its user's) decision.

---

## Security Considerations

- **Fake providers:** a server can list a provider it runs, with invented reviews. Mitigated by client allowlists of providers.
- **Wrong payment:** a server can put another party's transaction in the settlement response. Mitigated by provider check 1 and the client's on-chain check.
- **Free reviews through holds:** under schemes that hold funds first, a payer can authorize and then void or reclaim at little cost. Mitigated by check 1: only funds that reached `payTo` count.
- **Signature misuse:** limited by requiring plain message signatures that bind the payment, rating and text hash, and by refusing typed-data or transaction signatures.
- **Prompt injection:** review text is written by third parties. Clients passing it to a language model SHOULD mark it as untrusted.
- **Privacy:** fetching `read` tells the provider which resource the client is considering. Reviews and reviewer wallet addresses are public.

---

## Relationship to Other Extensions

- **`reputation` (#1024)** records feedback on chain against ERC-8004 agent identities, with seller-signed proof of interaction. `reviews` is narrower and complementary: it only standardizes where reviews are read and written, and what "payment-backed" means. An ERC-8004 feedback aggregator can be listed as a `reviews` provider, so the two compose rather than compete: identity and on-chain feedback can stay in `reputation`, discovery links in `reviews`.
- **`offer-and-receipt`:** a signed receipt can serve a provider as evidence of delivery (see above). `reviews` defines no signing by the server.

---

## References

- [Core x402 Specification](../x402-specification-v2.md)
- [`auth-capture` scheme](../schemes/auth-capture/scheme_auth_capture.md)
- [`batch-settlement` scheme](../schemes/batch-settlement/scheme_batch_settlement.md)
- [EIP-191: Signed Data Standard](https://eips.ethereum.org/EIPS/eip-191)
- [ERC-1271: Standard Signature Validation Method for Contracts](https://eips.ethereum.org/EIPS/eip-1271)
- [ERC-6492: Signature Validation for Predeploy Contracts](https://eips.ethereum.org/EIPS/eip-6492)
