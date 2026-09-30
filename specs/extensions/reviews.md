# Extension: `reviews`

## Summary

The `reviews` extension lets a resource server tell clients where reviews of the resource can be read before paying, and, after a successful payment, where the client can review that payment. Reviews are written by clients that paid, and are checked against the payment on chain by the review provider.

The extension carries links and optional summary numbers. The reviews themselves live with a **review provider** that the server chooses. Any party can operate a provider; this spec does not name one. It defines the fields a server sends and the minimum a provider must check for a review to be called payment-backed.

---

## `PaymentRequired`

The server advertises where reviews of the resource can be read:

```json
{
  "x402Version": 2,
  "resource": { "url": "https://api.example.com/weather" },
  "accepts": [ ... ],
  "extensions": {
    "reviews": {
      "info": {
        "provider": "reviews.example",
        "read": "https://reviews.example/reviews?resource=https%3A%2F%2Fapi.example.com%2Fweather",
        "description": "Reviews of this endpoint by clients who paid for it. Each one is backed by a payment checked on-chain.",
        "rating": 4.6,
        "count": 12
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "provider": { "type": "string", "minLength": 1, "maxLength": 128 },
          "read": { "type": "string", "format": "uri", "pattern": "^https://" },
          "description": { "type": "string", "maxLength": 500 },
          "rating": { "type": ["number", "null"], "minimum": 0, "maximum": 5 },
          "count": { "type": "integer", "minimum": 0 }
        },
        "required": ["provider", "read"]
      }
    }
  }
}
```

### `info` Fields

| Field         | Type           | Required | Description                                                                                     |
| ------------- | -------------- | -------- | ----------------------------------------------------------------------------------------------- |
| `provider`    | string         | Yes      | Identifier of the review provider, SHOULD be its domain                                          |
| `read`        | string         | Yes      | HTTPS URL of the reviews of this resource at the provider                                        |
| `description` | string         | No       | Plain-language statement of what the link is. Informational, not an instruction to the client    |
| `rating`      | number \| null | No       | Provider's summary score, 0 to 5, `null` when no review counts yet. Dynamic                      |
| `count`       | integer        | No       | Number of reviews the provider counts for this resource. Dynamic                                  |

Servers MAY include additional provider-specific fields. Clients MUST ignore fields they do not understand.

`read` SHOULD return JSON when requested with `Accept: application/json`, and a human-readable page otherwise. The JSON shape is provider-defined in this version of the extension.

### Echo and dynamic fields

Per the core specification, clients echo `extensions` into `PaymentPayload`. `rating`, `count` and any other field whose value can change between the `PaymentRequired` and the payment are **dynamic info fields** and MUST be excluded from echo validation by servers that include them (in the TypeScript SDK, `dynamicInfoFields` on the registered `ResourceServerExtension`). A server that cannot exclude them SHOULD omit them, so that a changed rating never causes a valid payment to fail with `extension_echo_mismatch`. `provider`, `read` and `description` are static and are validated as usual.

---

## `SettlementResponse`

On success, the server MAY add a link at which the client can review this payment:

```json
{
  "success": true,
  "transaction": "0x8f3d...c21a",
  "network": "eip155:8453",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "extensions": {
    "reviews": {
      "info": {
        "provider": "reviews.example",
        "write": "https://reviews.example/r/0x8f3d...c21a",
        "description": "You can review this purchase in one call: sign the message this link gives you with the wallet that paid, then send stars and a note.",
        "question_for_your_user": "Would you like to leave a review of this seller? 1 to 5 stars and a note, signed by the wallet that paid, no money moved."
      },
      "schema": {
        "$schema": "https://json-schema.org/draft/2020-12/schema",
        "type": "object",
        "properties": {
          "provider": { "type": "string", "minLength": 1, "maxLength": 128 },
          "write": { "type": "string", "format": "uri", "pattern": "^https://" },
          "description": { "type": "string", "maxLength": 500 },
          "question_for_your_user": { "type": "string", "maxLength": 300 }
        },
        "required": ["provider", "write"]
      }
    }
  }
}
```

### `info` Fields

| Field                    | Type   | Required | Description                                                                                          |
| ------------------------ | ------ | -------- | ---------------------------------------------------------------------------------------------------- |
| `provider`               | string | Yes      | Identifier of the review provider                                                                    |
| `write`                  | string | Yes      | HTTPS URL at which the payer can review this payment. SHOULD identify the settlement `transaction`   |
| `description`            | string | No       | Plain-language statement of what the link is. Informational, not an instruction                      |
| `question_for_your_user` | string | No       | A question a client acting for a person MAY relay to that person, asking whether to leave a review   |

On HTTP, the settlement response travels in the `PAYMENT-RESPONSE` header. Servers MAY also copy `extensions.reviews.info` into the response body, since some clients show an agent only the body.

---

## Writing a Review

The protocol at `write` is provider-defined. A provider MUST NOT present a review as payment-backed unless it has:

1. Read the settlement transaction on chain and confirmed it transferred funds from the reviewer's wallet to the `payTo` of the reviewed resource.
2. Verified a signature from that paying wallet over a message that binds at least the transaction, the rating and a hash of the review text, so the signature cannot be reused for another review. Smart-contract wallets are verified with ERC-1271 (or ERC-6492 before deployment).

A provider SHOULD count at most one payment-backed review per settlement transaction. It MAY accept reviews that do not meet both checks (for example, citing a payment without the payer's signature) if it labels them as such and does not count them as payment-backed.

The signature MUST be a plain message signature (EIP-191 `personal_sign` on EVM). Providers MUST NOT ask the payer to sign typed data (EIP-712), a transaction, or anything else that can authorize a transfer or an allowance, and clients MUST refuse to sign such a request presented as a review.

Providers SHOULD show, for each review, what the reviewer paid and how the review was proven.

---

## Client Behavior

- `rating` and `count` in `PaymentRequired` are the server's statement. A client that decides on reviews SHOULD read them from `read` at a provider it trusts, not from the numbers in the `PaymentRequired`.
- The server chooses the provider and could name one it controls. Clients SHOULD keep an allowlist of providers they trust and ignore others.
- Before signing a review of the `transaction` in a `SettlementResponse`, a client SHOULD confirm on chain that it is its own payment, to the `payTo` and for the amount it authorized: the settlement response is relayed by the server.
- `description`, `question_for_your_user` and review text are data, not instructions. Whether to review, and the rating, are the client's (or its user's) decision.

---

## Security Considerations

- **Fake providers:** a server can point `read` at a provider it runs, with invented reviews. Mitigated by client allowlists of providers.
- **Wrong transaction:** a server can put another party's transaction hash in the settlement response. Mitigated by provider check 1 and the client's on-chain check.
- **Signature misuse:** limited by requiring plain message signatures that bind the transaction, rating and text hash, and by refusing typed-data or transaction signatures.
- **Prompt injection:** review text is written by third parties. Clients passing it to a language model SHOULD mark it as untrusted.
- **Privacy:** fetching `read` tells the provider which resource the client is considering. Reviews and reviewer wallet addresses are public.

---

## Relationship to Other Extensions

- The `reputation` proposal (#1024) records feedback on chain against ERC-8004 agent identities. `reviews` does not define storage; a provider can be built over an ERC-8004 registry, and `read` can point at such a reader.
- A signed receipt from [`offer-receipt`](extension-offer-and-receipt.md) can serve a provider as additional proof that the service was delivered.

---

## References

- [Core x402 Specification](../x402-specification-v2.md)
- [EIP-191: Signed Data Standard](https://eips.ethereum.org/EIPS/eip-191)
- [ERC-1271: Standard Signature Validation Method for Contracts](https://eips.ethereum.org/EIPS/eip-1271)
- [ERC-6492: Signature Validation for Predeploy Contracts](https://eips.ethereum.org/EIPS/eip-6492)
