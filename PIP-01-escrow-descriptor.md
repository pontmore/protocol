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

## Selection Rules

Every agent profile should declare:

- at least one usable escrow configuration
- one default escrow configuration

That declared escrow must be usable without out-of-band negotiation at swap time.

## Open Question

Additional escrow mechanisms may still need their own canonical subtype-specific schemas beyond `lightning_hold_invoice`.
