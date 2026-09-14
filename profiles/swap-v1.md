# `pontmore/swap@1`: Bilateral Fiat/Bitcoin Swap Profile

## Status

- Status: Experimental
- Profile identifier: `pontmore/swap@1`
- Compatible kernel: [PIP-02 version 2](../PIP-02-coordination-event-chains.md)
- Interoperability claim: none until shared vectors and independent implementations exist

## Purpose

This profile defines the public terms and actions needed to reconstruct a bilateral fiat/Bitcoin swap on a PIP-02 coordination event chain.

It preserves `swap` as the application domain term. It does not define matching, offers, price discovery, bank or mobile-money instructions, escrow service operations, private evidence evaluation, or user experience.

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

Amounts are exact terms, not floating-point values. The profile does not define exchange-rate or fee calculation. When a quote supplies the terms, the root MUST bind it using `commitments.quote`.

This profile permits two commitment keys:

- `private_terms`
  - commits to the exact bytes of the agreed private payment terms
- `quote`
  - commits to the exact bytes of the signed, expiring quote

Both use PIP-02's `sha256-bytes@1` algorithm. Before publishing `core/accept`, the accepting participant MUST possess the committed bytes, verify their digest, validate any quote signature under the accepted service schema, confirm that the quote matches the root terms, and confirm that the quote has not expired.

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

Those vectors are not yet included. Implementations MUST describe support for this profile as experimental.
