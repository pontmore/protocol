# PIP-02: Swap State Machine

## Status

- Status: Draft
- Implementation: Required
- Scope: swap execution events and append-only lifecycle
- Related:
  - [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)
  - [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md)

## Purpose

This document defines the event model and state machine for swaps.

## Core Model

A swap is represented by:

- one immutable root event that defines the requested trade
- a sequence of immutable transition events that advance the swap state
- optional private Gift Wrap messages that carry non-public execution payloads
- optional replaceable snapshot events for fast lookup

The append-only transition log is the source of protocol history.
The snapshot is an optimization.
The private message lane is a companion channel rather than an alternate source of truth.

## Event Types

### Swap Request

- kind: `7300`
- immutable regular event

Required content:

- `version`
- `swap_id`
- `swap_type`
- `agent`
- `customer`
- `escrow_reference`
- `fiat`
- `bitcoin`
- `expiry`

### Transition

- kind: `7301`
- immutable regular event

Required content:

- `swap_id`
- `state`
- `prev_state`
- `actor_role`
- `reason`
- `created_at`

### Evidence

- kind: `7302`
- immutable regular event

Examples:

- fiat transfer reference
- bank confirmation
- payout proof
- escrow funding proof
- redacted external settlement proof
- encrypted invoice reference
- consented disclosure reference

### Dispute

- kind: `7303`
- immutable regular event

Dispute grammar is part of the swap lifecycle, while dispute policy is defined in [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md).

### Note

- kind: `7304`
- immutable regular event

Optional human-readable operational note tied to a swap.

### Snapshot

- kind: `30362`
- addressable

Provides a replaceable materialized view of current state for fast lookup.

## Companion Private Message Lane

Some swap execution payloads SHOULD NOT be published in raw form on the public state machine.

Pontmore therefore permits a companion private message lane using Nostr Gift Wrap for:

- invoices
- bank account details
- payout instructions
- delivery acknowledgements
- sensitive evidence payloads

The private message lane is for transport of non-public payloads.
It does not replace the public swap request, transition, evidence, dispute, or snapshot events.

### Constrained Access Channels

Agents may expose constrained access channels such as USSD, SMS, or chat menus in their agent definition.

Those channels are implementation surfaces. They do not alter the public swap event grammar.

Actions initiated through a constrained channel SHOULD map to the same public event types:

- swap requests use kind `7300`
- state changes use kind `7301`
- evidence references use kind `7302`
- disputes use kind `7303`
- operational notes use kind `7304`
- snapshots use kind `30362`

Transport-specific session identifiers, phone numbers, short codes, menu state, callback payloads, and gateway metadata MUST NOT be treated as canonical protocol state.

If constrained-channel execution requires private instructions or acknowledgements, implementations SHOULD carry those details through the companion private message lane and publish only the minimum public transition, evidence reference, dispute, note, or snapshot needed for interoperability.

### Private Lane Rules

- private messages SHOULD be correlated to a swap by `swap_id` or a deterministic reference derived from it
- private messages SHOULD be versioned
- private messages MAY include hashes or opaque references that are later cited by public evidence events
- acknowledgements exchanged in the private lane MAY be summarized publicly through transition or note events
- implementations SHOULD minimize the exposure of payer details, banking coordinates, invoice secrets, and payout destinations

## Evidence Handling

Public evidence events SHOULD default to reference-style publication rather than raw payload disclosure.

Recommended evidence publication modes:

- reveal by reference
  - publish a hash, opaque pointer, or encrypted payload reference without exposing the underlying artifact
- reveal by consent
  - disclose raw or decryptable evidence only when the relevant participant or policy trigger authorizes it
- reveal by necessity
  - disclose the minimum subset needed for dispute resolution or settlement finality

Raw invoices, bank details, screenshots, and settlement secrets SHOULD remain in the private message lane unless a dispute policy or escrow policy requires targeted disclosure.

## State Machine Properties

- the request event is immutable
- every transition is append-only
- sequence coherence matters
- immutable history is authoritative over snapshots
- private Gift Wrap payloads are supplementary and never override public history

## Participant Responsibilities

### Agent

- keep relay preferences available
- publish evidence or confirmations in a timely way
- reference usable escrow declarations

### Customer

- reference a valid current agent definition
- select a declared escrow descriptor
- provide required settlement instructions through the appropriate public or private lane
- submit payment proof when required

### Escrow operator

- expose enough public information to be referenced in protocol flows
- publish or validate settlement transitions
- publish resolution transitions when acting as arbiter
- avoid unnecessary publication of raw private settlement payloads
