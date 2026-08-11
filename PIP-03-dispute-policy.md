# PIP-03: Dispute Policy

## Status

- Status: Draft
- Implementation: Required
- Scope: operator-layer dispute and timeout policy for Nostr-native swaps
- Related:
  - [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)

## Purpose

This document defines how an operator should resolve disputes and timeouts for swaps that are modeled as Nostr-native state machines.

## Core Rule

- **the swap state machine is public**
- **the dispute process is operator-governed**

## Dispute Classes

- payment not received
- payment amount mismatch
- payout not sent
- payout amount mismatch
- escrow funding failure
- conflicting external confirmations
- fraud or impersonation risk
- timeout and abandonment

## Timeout Classes

- request expiry
- funding timeout
- payment proof timeout
- payout timeout
- resolution timeout
- refund-trigger timeout

Each timeout class MUST bind to exactly one non-`mutual_consent` fallback resolution:

- payment proof timeout -> confirming the agent claim
- payout timeout -> confirming the customer claim
- resolution timeout -> escalating to manual review
- refund-trigger timeout -> the descriptor-declared fallback resolution from [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)

## Evidence Categories

- payment reference
- transfer receipt
- escrow funding proof
- payout proof
- operator note
- external confirmation reference
- redacted document hash
- encrypted payload reference
- participant consent record

## Resolution Modes

An operator may resolve disputes by:

- confirming the customer claim
- confirming the agent claim
- splitting outcome where escrow policy allows it
- cancelling and refunding
- escalating to manual review

### Timeout Fallback

A timeout class (for example `resolution timeout` or an escrow's `refund_trigger` such as `timeout_requires_mutual_consent`) MUST NOT leave the escrow in a permanent deadlock where neither participant will consent and no other resolution path is defined. When a timeout elapses, the operator MUST resolve the swap using the fallback resolution bound to that timeout class; a `mutual_consent`-only path with no fallback is not a valid terminal policy. The escrow descriptor referenced by the swap MUST declare the applicable fallback resolution for every timeout class it advertises, and compatibility validation MUST use that explicit binding rather than inferring a fallback from the trigger name alone (see the Refund-Trigger Fallback section of [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)).

## Public-Protocol Boundary

### Public

- dispute opened
- dispute escalated
- dispute resolved
- public evidence references
- resolution actor and policy id

### Usually private

- raw invoices
- bank details
- raw screenshots
- internal notes
- private documents
- internal scoring logic
- third-party payloads that contain sensitive data

## Disclosure Rules

Dispute handling SHOULD follow a private-by-default evidence rule.

Recommended disclosure order:

1. reference only
2. redacted excerpt
3. full artifact by participant consent
4. full artifact by explicit dispute-policy necessity

Operators SHOULD disclose the minimum evidence needed to justify the outcome.
When raw evidence is disclosed, the public protocol surface SHOULD prefer a reference to the disclosure act, consent record, or redacted derivative rather than unrestricted republication of the full artifact.
