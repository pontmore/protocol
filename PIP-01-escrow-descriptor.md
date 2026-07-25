# PIP-01: Escrow Descriptor

## Status

- Status: Draft
- Implementation: Required
- Scope: public escrow declaration for swap compatibility and execution assumptions
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)

## Purpose

This document defines the public escrow descriptor event referenced by agent definitions and swaps.

## Event Type

- kind: `30361`
- addressable
- `d` tag: stable identifier for one escrow configuration

## Function

The escrow descriptor tells counterparties and operators:

- what escrow mechanism is used
- which settlement or invoice networks are supported
- what the funding and release rules are
- how the escrow instance is referenced
- what timeout and dispute assumptions apply

## Minimum Content

`content` is JSON and MUST be versioned.

Minimum expected fields:

- `version`
- `escrow_type`
- `networks`
- `funding_rules`
- `release_rules`
- `dispute_rules`
- `reference_format`
- `updated_at`

## Network Declaration

An escrow descriptor MUST declare every settlement or invoice network supported by the escrow configuration.

The canonical content field is:

- `networks`
  - non-empty array of lowercase network identifiers
  - examples: `bitcoin`, `lightning`

Each value in `networks` SHOULD also be emitted as a repeated `network` tag for relay filtering:

```text
["network", "bitcoin"]
["network", "lightning"]
```

The `networks` content array is the canonical declaration. The repeated tags are an index and discovery aid. Clients MUST NOT treat a repeated `network` tag as supported unless it also appears in `content.networks`.

## Example: Bitcoin and Lightning

```json
{
  "version": 1,
  "escrow_type": "lightning_hold_invoice",
  "networks": ["bitcoin", "lightning"],
  "funding_rules": {
    "required_confirmation": "invoice_held"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "bolt11",
  "invoice_network": "lightning",
  "payout_network": "bitcoin",
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include:

```text
["d", "default"]
["network", "bitcoin"]
["network", "lightning"]
```

## Canonical Subtype: `lightning_hold_invoice`

`lightning_hold_invoice` is a canonical escrow subtype for swaps that use a Lightning hold invoice as the escrow lock.

When `escrow_type` is `lightning_hold_invoice`, the descriptor SHOULD include the following additional fields:

- `invoice_network`
- `invoice_asset`
- `invoice_currency`
- `invoice_amount_rule`
- `hold_expiry_rule`
- `settle_authority`
- `cancel_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `preimage_visibility`
- `payout_network`

### Field Intent

- `invoice_network`
  - Lightning network on which the hold invoice is issued
- `invoice_asset`
  - asset locked by the hold invoice, usually bitcoin-denominated Lightning liquidity
- `invoice_currency`
  - invoice denomination convention used by the escrow provider
- `invoice_amount_rule`
  - whether the invoice amount is exact, bounded, or derived from the swap request
- `hold_expiry_rule`
  - timeout rule for unpaid or unresolved hold invoices
- `settle_authority`
  - which participant or operator may settle the invoice
- `cancel_authority`
  - which participant or operator may cancel the invoice
- `release_rules.release_trigger`
  - public condition required before settlement is valid
- `release_rules.refund_trigger`
  - public condition required before cancellation or refund is valid
- `preimage_visibility`
  - whether the preimage is expected to remain operator-local, participant-visible, or public by reference only
- `payout_network`
  - expected payout path after successful release

### Lifecycle Rules

For `lightning_hold_invoice`, the escrow lifecycle SHOULD follow these phases:

1. invoice issued
2. invoice held
3. release condition satisfied
4. settled or canceled

The public swap state machine SHOULD record:

- when the hold invoice becomes the active escrow reference
- when funding is confirmed
- when release authority is exercised
- when cancellation or refund authority is exercised

Raw invoice payloads, settlement secrets, and other sensitive Lightning material SHOULD stay in the companion private message lane unless explicit disclosure is required.

## Canonical Subtype: `custodial_escrow`

`custodial_escrow` is a canonical escrow subtype for swaps where an escrow operator takes custody of the settlement asset or invoice claim and releases or refunds it according to public protocol state.

This subtype is intentionally network-generic. The top-level `networks` array remains the canonical supported-network declaration for the descriptor. Network-specific implementation details MUST be represented under `implementations` and MUST NOT expand the supported network set beyond `content.networks`.

When `escrow_type` is `custodial_escrow`, the descriptor MUST include the following additional fields:

- `custody_authority`
- `release_authority`
- `refund_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `implementations`

### Custodial Field Intent

- `custody_authority`
  - participant or operator role that controls the custodied escrow asset or claim while the swap is active
- `release_authority`
  - participant or operator role that may release the escrow after `release_rules.release_trigger` is satisfied
- `refund_authority`
  - participant or operator role that may refund or cancel the escrow after `release_rules.refund_trigger` is satisfied
- `release_rules.release_trigger`
  - public condition required before release is valid
- `release_rules.refund_trigger`
  - public condition required before refund or cancellation is valid
- `implementations`
  - non-empty array of network-specific implementation profiles

Each implementation entry MUST include:

- `network`
  - one network identifier present in top-level `networks`

Implementation entries MAY include network-specific fields such as:

- `invoice_asset`
- `invoice_currency`
- `invoice_amount_rule`
- `invoice_expiry_rule`
- `payout_network`
- `reference_format`

If an implementation entry includes `reference_format`, it overrides the top-level `reference_format` for that implementation. Otherwise, clients MUST use the top-level `reference_format`.

Clients MUST ignore implementation entries whose `network` value is not present in top-level `networks`. If no valid implementation entries remain, clients MUST treat the descriptor as unusable.

### Example: Lightning Custodial Invoice

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["lightning"],
  "funding_rules": {
    "required_confirmation": "invoice_paid"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed",
    "refund_trigger": "timeout_or_dispute_refund_decision"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "custody_authority": "escrow_operator",
  "release_authority": "escrow_operator",
  "refund_authority": "escrow_operator",
  "implementations": [
    {
      "network": "lightning",
      "invoice_asset": "BTC",
      "invoice_currency": "sats",
      "invoice_amount_rule": "derived_from_swap_request",
      "invoice_expiry_rule": "expires_if_unpaid_before_funding_timeout",
      "payout_network": "lightning"
    }
  ],
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include only networks present in `content.networks`:

```text
["d", "default"]
["network", "lightning"]
```

### Custodial Lifecycle Rules

For `custodial_escrow`, the escrow lifecycle SHOULD follow these phases:

1. escrow reference issued
2. custody funded or claim accepted
3. release or refund condition satisfied
4. released, refunded, or canceled

The public swap state machine SHOULD record:

- when the custodial escrow reference becomes active
- when custody funding or claim acceptance is confirmed
- when release authority is exercised
- when refund or cancellation authority is exercised

Raw invoices, private payment instructions, operator account details, and custody internals SHOULD stay in the companion private message lane unless explicit disclosure is required.

## Canonical Subtype: `cashu_escrow`

`cashu_escrow` is a canonical escrow subtype where funds are held as Cashu ecash tokens locked to the escrow operator's pubkey using NUT-11 spending conditions, with a refund pubkey and locktime. It composes on the `custodial_escrow` pattern, with the operator acting as release and refund authority, but the custody is on the bearer instrument itself rather than a relational balance held by the operator.

When `escrow_type` is `cashu_escrow`, the descriptor MUST include the following additional fields:

- `custody_authority`
- `release_authority`
- `refund_authority`
- `release_rules.release_trigger`
- `release_rules.refund_trigger`
- `implementations`

Each implementation entry describing the Cashu escrow lock MUST include:

- `network`
  - one network identifier present in top-level `networks`; for the Cashu escrow lock, MUST be `cashu`
- `mint_url`
  - URL of the Cashu mint used to issue and redeem tokens
- `lock_mechanism`
  - locking primitive; MUST be `p2pk_timelock` (NUT-11 P2PK with locktime and refund pubkey)

Each implementation entry describing the Cashu escrow lock SHOULD include:

- `invoice_expiry_rule`
  - SHOULD be `p2pk_timelock_expiry`
- `reference_format`
  - SHOULD be `cashu_v4_token`
- `payout_network`
  - expected payout path after successful release, typically `lightning`

### Cashu Field Intent

- `network`
  - network identifier for the Cashu implementation entry; MUST be `cashu` and MUST appear in top-level `networks`
- `mint_url`
  - identifies the Cashu mint responsible for issuing and redeeming the locked tokens
- `lock_mechanism`
  - declares the NUT-11 P2PK timelock primitive that binds the tokens to the operator pubkey, refund pubkey, and locktime
- `invoice_expiry_rule`
  - declares that the escrow expires according to the P2PK locktime
- `reference_format`
  - declares that the escrow is referenced as a Cashu v4 token
- `payout_network`
  - declares the expected payout path after successful release

### Example: Cashu P2PK Timelock

```json
{
  "version": 1,
  "escrow_type": "cashu_escrow",
  "networks": ["cashu", "lightning"],
  "funding_rules": {
    "required_confirmation": "cashu_token_locked_to_escrow"
  },
  "release_rules": {
    "release_trigger": "counterparty_fiat_payment_confirmed",
    "refund_trigger": "timeout_or_dispute_refund_decision"
  },
  "dispute_rules": {
    "policy": "operator_resolved"
  },
  "reference_format": "cashu_v4_token",
  "custody_authority": "escrow_operator",
  "release_authority": "escrow_operator",
  "refund_authority": "escrow_operator",
  "implementations": [
    {
      "network": "cashu",
      "invoice_asset": "BTC",
      "invoice_currency": "sats",
      "invoice_amount_rule": "derived_from_swap_request",
      "invoice_expiry_rule": "p2pk_timelock_expiry",
      "payout_network": "lightning",
      "mint_url": "https://mint.example.com",
      "lock_mechanism": "p2pk_timelock",
      "reference_format": "cashu_v4_token"
    }
  ],
  "updated_at": 1775559028
}
```

The matching event tags SHOULD include only networks present in `content.networks`:

```text
["d", "default"]
["network", "cashu"]
["network", "lightning"]
```

### Cashu Lifecycle Rules

For `cashu_escrow`, the escrow lifecycle SHOULD follow these phases:

1. locked token reference issued
2. lock verified against declared mint and operator pubkey
3. release or refund condition satisfied
4. released, refunded, or locktime-expired

The public swap state machine SHOULD record:

- when the locked token reference becomes active
- when funding lock is confirmed
- when release authority is exercised
- when refund or cancellation authority is exercised

Raw Cashu token strings, mint credentials, and preimages SHOULD stay in the companion private message lane. Only opaque references or hashes SHOULD appear in public evidence events.

## Selection Rules

Every agent profile should declare:

- at least one usable escrow configuration
- one default escrow configuration

That declared escrow must be usable without out-of-band negotiation at swap time.

## Open Question

Additional escrow mechanisms beyond `lightning_hold_invoice`, `custodial_escrow`, and `cashu_escrow` may still need their own canonical subtype-specific schemas.

`cashu_escrow` introduces a dependency on a specific Cashu mint. Mint trust, mint selection, and mint failure modes are implementation assumptions rather than Pontmore protocol state. [Nimdolf](https://github.com/cashubtc/nuts/pull/390) is a possible future direction for mint liveness failover, but this proposal does not bind to it.
