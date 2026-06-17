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
- access-channel routing details such as phone numbers, USSD session identifiers, service codes, callback payloads, and gateway metadata

## Disclosure Rules

Dispute handling SHOULD follow a private-by-default evidence rule.

Recommended disclosure order:

1. reference only
2. redacted excerpt
3. full artifact by participant consent
4. full artifact by explicit dispute-policy necessity

Operators SHOULD disclose the minimum evidence needed to justify the outcome.
When raw evidence is disclosed, the public protocol surface SHOULD prefer a reference to the disclosure act, consent record, or redacted derivative rather than unrestricted republication of the full artifact.
