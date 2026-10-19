# `pontmore/swap@1`: Bilateral Fiat/Bitcoin Swap Profile

## Status

- Status: Experimental
- Profile identifier: `pontmore/swap@1`
- Compatible kernel: [PIP-02 version 2](../PIP-02-coordination-event-chains.md)
- Interoperability claim: none until shared vectors and independent implementations exist

## Purpose

This profile defines the public terms and actions needed to reconstruct a bilateral fiat/Bitcoin swap on a PIP-02 coordination event chain.

It preserves `swap` as the application domain term. It defines standing swap offers as NIP-69 peer-to-peer order events. It does not define matching, market price discovery or price formation, bank or mobile-money instructions, escrow service operations, private evidence evaluation, or user experience. The deterministic fiat-to-sats conversion from a bound offer's exact `price` is defined under [Offers](#offers).

## Participant Roles

Every root MUST bind exactly one pubkey to each application role:

- `swap/agent`
- `swap/customer`

The two application roles MUST use different pubkeys. The root MUST also satisfy PIP-02's `core/escrow` and, when disputes are enabled, `core/resolver` authority binding.

The root signer is the proposer. The other application participant is the accepting participant and MUST publish `core/accept`. The proposal is accepted after that valid action. The accepting participant MAY instead publish `core/decline` before acceptance.

## Terms

The root's `terms` object MUST contain:

- `direction`
  - `fiat_to_btc`: the customer sends fiat and the Agent provides Bitcoin
  - `btc_to_fiat`: the Agent sends fiat and the customer provides Bitcoin
- `fiat`
  - `currency`: uppercase ISO 4217 currency code
  - `amount`: positive base-10 decimal string with no exponent
- `bitcoin`
  - `amount`: positive base-10 integer string
  - `unit`: `sat`
  - `network`: network identifier compatible with the accepted PIP-01 descriptor
- `payment_channel`
  - versioned identifier whose detailed payment instructions remain private
- `deadlines`
  - `fiat_pay_by`: Unix timestamp by which `swap/fiat_sent` may be recorded
  - `fiat_confirm_by`: later Unix timestamp by which `swap/fiat_confirmed` may be recorded without dispute

The root's PIP-02 `expires_at` is the acceptance deadline and MUST be earlier than `terms.deadlines.fiat_pay_by`. `fiat_pay_by` MUST be earlier than `fiat_confirm_by`.

Example:

```json
{
  "direction": "fiat_to_btc",
  "fiat": {
    "currency": "KES",
    "amount": "1000"
  },
  "bitcoin": {
    "amount": "60000",
    "unit": "sat",
    "network": "lightning"
  },
  "payment_channel": "mpesa-ke-kes@1",
  "deadlines": {
    "fiat_pay_by": 1780001800,
    "fiat_confirm_by": 1780003600
  }
}
```

Amounts are exact terms, not floating-point values. The profile does not define market price discovery, price formation, or fee calculation. The deterministic conversion from a bound offer's exact `price` to `terms.bitcoin.amount` is specified under [Offers](#offers). When a quote supplies the terms, the root MUST bind it using `commitments.quote`.

This profile permits two commitment keys:

- `private_terms`
  - commits to the exact bytes of the agreed private payment terms
- `quote`
  - commits to the NIP-69 offer event through its NIP-01 canonical serialization, as defined under [Committed Bytes](#committed-bytes); that offer is the standing, signed, expiring public price accepted through the root

Both use PIP-02's `sha256-bytes@1` algorithm. Before publishing `core/accept`, the accepting participant MUST possess the committed bytes, verify their digest, validate the quote signature (for a NIP-69 offer, the Nostr event signature by the `swap/agent` pubkey), confirm that the quote matches the root terms, and confirm that the quote has not expired at the time of `core/accept` (offer `expires_at` is later than the `core/accept` action's `created_at`).

The NIP-69 offer is a standing public price commitment, not a per-coordination service quote under [PIP-01](../PIP-01-escrow-descriptor.md). PIP-01's service-quote rule applies when a service schema defines a signed, expiring quote for variable fees or terms; a fixed `price` signed in a public offer is not that. The accepted escrow service schema does not own or validate this offer. This profile does not define a service-fee quote commitment key; a future profile revision that requires one MUST add a distinct key and MUST NOT reuse `quote`.

## Offers

A standing swap offer is a [NIP-69](https://github.com/nostr-protocol/nips/blob/master/69.md) kind `38383` addressable event signed by the `swap/agent` identity key. It advertises a price and fiat-amount range for one currency and direction so that other Nostr P2P clients can discover it.

Offers are optional. A root MAY be proposed without one. When a root's terms come from an offer, the root binds the exact offer event through `commitments.quote`.

### Offer Event

- kind: `38383`
- addressable
- `d`: `pontmore/swap@1:<fiat-currency>:<k>`

One offer per currency and direction. The addressable `d` slot replaces earlier offers at the same coordinate.

NIP-69 tags carry the shared listing data:

| Tag | Value |
| --- | --- |
| `k` | `sell` when `fiat_to_btc` (Agent provides Bitcoin); `buy` when `btc_to_fiat` |
| `f` | ISO 4217 fiat currency code |
| `s` | `pending` while active; `canceled` to withdraw |
| `amt` | `0` (see [NIP-69 `amt` Deviation](#nip-69-amt-deviation)) |
| `fa` | exactly two positive base-10 decimal strings with no exponent: minimum and maximum fiat amount |
| `pm` | human-readable payment method names |
| `premium` | informational premium percentage (optional) |
| `network` | Bitcoin network (`mainnet`, `testnet`, etc.); informational for Pontmore binding — the settlement network is constrained by the accepted escrow descriptor |
| `layer` | settlement layer (`lightning`, `onchain`, `liquid`, etc.); MUST match `terms.bitcoin.network` |
| `expires_at` | Unix timestamp after which the offer is no longer selectable |
| `expiration` | optional relay deletion timestamp (NIP-40); when present, relays SHOULD delete the offer after this time |
| `y` | `pontmore` |
| `z` | `order` |

`network` identifies the Bitcoin network (for example `mainnet` or `testnet`). `layer` identifies the settlement layer (for example `lightning` or `onchain`). The two are distinct: a `testnet` offer on the `lightning` layer settles on Bitcoin testnet over Lightning. The accepted escrow descriptor's `networks` constrain the settlement network; the offer's `network` tag is informational and does not override that constraint.

Pontmore profile tags, ignored by other NIP-69 clients, carry what binding needs:

| Tag | Value |
| --- | --- |
| `protocol` | `pontmore/swap@1` |
| `price` | exact fiat amount per BTC, as a positive base-10 decimal string with no exponent |
| `channels` | one tag per versioned payment-channel identifier |
| `a` | escrow descriptor address `30361:<publisher-pubkey>:<descriptor-d-tag>` with optional relay hint and `escrow` marker |
| `p` | resolver pubkey with optional relay hint and `core/resolver` role |

`content` MUST be empty. Payment instructions MUST NOT appear in the offer.

### Direction Mapping

| `terms.direction` | `k` |
| --- | --- |
| `fiat_to_btc` | `sell` |
| `btc_to_fiat` | `buy` |

### Binding Rules

A root's `commitments.quote` matches the offer only if:

1. the offer is signed by the root's `swap/agent` pubkey;
2. the offer's `protocol` tag equals `pontmore/swap@1`;
3. the offer's `d` tag equals `pontmore/swap@1:<f>:<k>` for its `f` and `k` tags;
4. `s` is `pending` and `expires_at` is later than the root's `created_at`;
5. `k` matches `terms.direction` per the direction mapping;
6. `f` matches `terms.fiat.currency`;
7. `layer` matches `terms.bitcoin.network`;
8. `terms.payment_channel` appears in the offer's `channels`;
9. `terms.fiat.amount` is within the offer's `fa` range (inclusive);
10. `terms.bitcoin.amount` equals `floor(fiat.amount × 100000000 ÷ price)`, computed with arbitrary-precision decimal or rational arithmetic and explicit floor rounding, not floating-point;
11. the offer's `a` escrow address matches the root's descriptor coordinate;
12. the offer's `p` resolver matches the root's `core/resolver` binding.

### Committed Bytes

The committed bytes for `commitments.quote` are the UTF-8 encoding of the NIP-01 canonical event serialization of the offer event: the JSON array `[0, <pubkey>, <created_at>, <kind>, <tags>, <content>]` whose SHA-256 is the Nostr event ID. The media type is `application/json`, and the canonicalization is the explicit NIP-01 event serialization. The `sha256-bytes@1` digest is `sha256:` followed by that event ID (64 lowercase hexadecimal characters). Before publishing `core/accept`, the accepting participant MUST verify the offer event signature, confirm the digest matches `commitments.quote`, confirm that the offer's `expires_at` is later than the `core/accept` action's `created_at`, and check all binding rules above.

### Freshness and Privacy

When the Agent is the non-proposing participant, it MAY decline a root whose quote is superseded by a later offer at the same `d` coordinate, or whose price is materially worse for the Agent than its current price. When the Agent is the proposer, it MAY cancel before acceptance under [Kernel Authorization](#kernel-authorization).

Agents SHOULD keep offer lifetimes short to reduce stale-quote risk.

Agents SHOULD NOT publish `fa` maximums that reveal exact liquidity. An Agent MAY round a liquidity-capped maximum down to a coarse value.

### NIP-69 `amt` Deviation

NIP-69 defines `amt` `0` as an amount obtained from a public API after the taker accepts the order. This profile instead derives the Bitcoin amount from the offer's exact `price` at binding time (binding rule 10). NIP-69 aggregators that do not implement `pontmore/swap@1` will see `amt` `0` but will not know the `price` tag or the conversion rule; they can list the offer but cannot quote a satoshi amount for it.

### Interoperability

NIP-69 aggregators can list Pontmore offers alongside orders from other platforms. Clients such as Mostro cannot take a Pontmore offer without implementing `pontmore/swap@1`, because escrow and coordination differ: Mostro uses coordinator-held hold invoices, while Pontmore binds participant escrow on a PIP-02 chain.

Mostro's own clients may list only orders signed by their Mostro instance. The interoperability gain is aggregator visibility, not cross-platform settlement.

NIP-69 models one order per trade with a status lifecycle (`pending`, `in-progress`, `success`, `canceled`, `expired`). A Pontmore offer is a standing quote that stays `pending` until it is republished or withdrawn as `canceled`; it never transitions to `in-progress` or `success`. Clients that expect those transitions will never see them on a Pontmore offer.

### Offer Open Questions

1. Should the price-freshness rule be normative (for example a maximum tolerance), or remain Agent policy?
2. Should `channels` reference a named registry, or remain any versioned identifier?
3. Should NIP-40 `expiration` on a withdrawn (`canceled`) offer be required so relays drop it?

## Direction-Derived Roles

Implementations derive economic roles from `direction`:

| Direction | Fiat sender | Fiat receiver | Bitcoin provider | Bitcoin recipient |
| --- | --- | --- | --- | --- |
| `fiat_to_btc` | `swap/customer` | `swap/agent` | `swap/agent` | `swap/customer` |
| `btc_to_fiat` | `swap/agent` | `swap/customer` | `swap/customer` | `swap/agent` |

These derived roles do not create new signing keys. They select one of the pubkeys already bound as `swap/agent` or `swap/customer`.

## Profile Actions

This profile defines two public actions:

| Action | Authorized signer | Preconditions | Required data | Derived effect |
| --- | --- | --- | --- | --- |
| `swap/fiat_sent` | Fiat sender | Accepted, secured, not disputed, before `fiat_pay_by` | `payment_reference` | Fiat payment claimed sent |
| `swap/fiat_confirmed` | Fiat receiver | Valid `swap/fiat_sent`, not disputed, before `fiat_confirm_by` | `payment_reference` | Fiat receipt confirmed |

`data.payment_reference` MUST be a safe opaque identifier or a versioned commitment. It MUST NOT contain raw payment instructions, account details, phone numbers, receipts, credentials, or screenshots. The `swap/fiat_confirmed` reference MUST identify the same committed payment as the `swap/fiat_sent` reference.

An action is valid only once. A second otherwise valid instance is a replay or fork under PIP-02.

## Kernel Authorization

This profile applies these rules to PIP-02 actions:

| Kernel action | Authorized signer | Additional profile condition |
| --- | --- | --- |
| `core/accept` | Non-proposing application participant | Before root `expires_at` |
| `core/decline` | Non-proposing application participant | Before acceptance |
| `core/cancel` | Proposer | Before acceptance |
| `core/cancel` | Either application participant | After acceptance but before `core/secure` |
| `core/expire` | Either application participant | Root was not accepted before `expires_at` |
| `core/secure` | `core/escrow` | Swap accepted; Bitcoin security condition satisfied |
| `core/authorize_settlement` | Fiat receiver | Valid `swap/fiat_confirmed` by the same signer |
| `core/settle` | `core/escrow` | Settlement authorized |
| `core/authorize_refund` | Bitcoin provider | Secured; `fiat_pay_by` elapsed with no valid `swap/fiat_sent` |
| `core/refund` | `core/escrow` | Refund authorized |
| `core/open_dispute` | Either application participant | Accepted and no terminal outcome |
| `core/resolve_dispute` | `core/resolver` | Disputed; effect supported by the accepted service |

After `swap/fiat_sent`, absence of `swap/fiat_confirmed` does not select a default winner. Either participant may open a dispute after `fiat_confirm_by`; the bound resolver and accepted escrow service determine a permitted recovery effect.

This profile permits the PIP-02 resolution effects `resume`, `authorize_settlement`, `authorize_refund`, and `cancel`, provided the accepted escrow service supports the selected effect.

## Reconstruction

The normal settlement history is:

```text
proposed
  -> core/accept
accepted
  -> core/secure
secured
  -> swap/fiat_sent
fiat_sent
  -> swap/fiat_confirmed
fiat_confirmed
  -> core/authorize_settlement
settlement_authorized
  -> core/settle
settled
```

The no-payment refund history is:

```text
secured
  -> fiat_pay_by elapses without swap/fiat_sent
  -> core/authorize_refund
refund_authorized
  -> core/refund
refunded
```

Dispute state overlays the pre-dispute state as specified by PIP-02. A `resume` resolution returns to that state. Other resolution effects permit only the corresponding kernel recovery path.

## Dispute Classes

An optional `class` field in `core/open_dispute.data` MAY contain one of:

- `fiat_not_received`
- `incorrect_fiat_amount`
- `payment_reference_invalid`
- `escrow_not_secured`
- `bitcoin_not_released`
- `conflicting_confirmation`
- `timeout`

The class is a routing claim, not proof and not a default outcome. Evidence requirements and evaluation belong to the accepted escrow service or application policy.

## Private Boundary

Payment instructions, payer and payee identifiers, account or phone details, invoices, receipts, screenshots, raw tokens, preimages, and settlement secrets MUST remain outside public root and action content.

When private terms or evidence affect validation, the root or action MUST carry a commitment allowed by PIP-02. The corresponding private payload MUST identify `pontmore/swap@1`, the coordination root, its intended participants, and its commitment scheme.

## Terminal States

The PIP-02 terminal states apply. This profile defines no additional final economic outcome. `settled` and `refunded` remain mutually exclusive.

## Conformance Vectors

Before this profile can be stabilized, shared vectors MUST cover at least:

- both directions
- acceptance before and after expiry
- descriptor expiry before and after root creation
- normal settlement
- no-payment refund
- disputed settlement and refund authorization
- mismatched payment references
- unauthorized signers
- duplicate and out-of-order actions
- sibling forks at every economic gate
- attempted settlement and refund in the same history
- public/private data separation
- offer binding with matching and non-matching fields
- expired and superseded offers
- Bitcoin amount derived from price

Those vectors are not yet included. Implementations MUST describe support for this profile as experimental.
