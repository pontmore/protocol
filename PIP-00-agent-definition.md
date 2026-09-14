# PIP-00: Agent Definition

## Status

- Status: Draft
- Implementation: Required for the Discovery conformance profile
- Scope: public capability discovery and protocol-resource references
- Related:
  - [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)

## Purpose

This document defines the public Agent definition event used to discover versioned capabilities and the protocol resources needed to use them.

An Agent definition is a capability index. It is not a service offer, price quote, availability proof, reputation statement, or substitute for standard Nostr profile and relay-list events.

## Event Type

- kind: `30360`
- addressable
- `d` tag: stable identifier for one capability index, usually `agent`

## Discovery Inputs

An Agent is discovered through:

- its Nostr pubkey
- a standard Nostr profile event for human-readable metadata
- a standard relay-list event for relay preferences
- an Agent definition event for Pontmore capabilities

Clients MUST NOT treat fields from the profile or relay-list event as part of the signed Pontmore capability claim.

## Required Tags

Every Agent definition MUST include:

- `["d", "<stable-identifier>"]`
- `["t", "agent"]`
- one `["t", "pontmore-capability:<capability-id>@<version>"]` tag for each capability declared in `content.capabilities`

`content.capabilities` is canonical. Clients MUST reject an Agent definition when a `pontmore-capability:` tag claims an identifier absent from `content.capabilities` or a declared capability has no matching tag.

An Agent definition that refers to an escrow configuration MUST also include an addressable-event reference:

```text
["a", "30361:<escrow-publisher-pubkey>:<descriptor-d-tag>", "<relay-hint>", "escrow"]
```

The relay hint MAY be empty. An `a` tag identifies a descriptor address, not one immutable descriptor revision.

## Content Schema

`content` MUST be a JSON object with these fields:

- `version`
  - integer Agent-definition schema version
- `capabilities`
  - non-empty array of versioned capability identifiers

Each capability identifier MUST have the form `<namespace>/<capability>@<positive-integer-version>`.

Example:

```json
{
  "version": 1,
  "capabilities": [
    "pontmore/swap@1"
  ]
}
```

The following facts are supplied by the Nostr event and MUST NOT be duplicated in content:

- publisher identity: event `pubkey`
- publication time: event `created_at`
- signature: event `sig`

Human-readable `name` and `about` fields belong in the standard Nostr profile event. Relay declarations belong in the standard relay-list event. Prices, margins, limits, inventory, and other commercial terms belong in signed, expiring offers or quotes defined outside PIP-00.

## Capability Profiles

The specification identified by a capability identifier owns any capability-specific discovery facts and their tags. PIP-00 does not define one universal schema for currencies, payment channels, markets, limits, or application actions.

A capability profile MAY define additional content or tags only when independent clients need those facts before invoking the capability. Such extensions MUST be namespaced and versioned by the capability profile.

Tags that repeat capability facts are indexes only. When a profile defines the same fact in canonical content and in a tag, clients MUST reject inconsistent values and MUST NOT combine them into a synthetic claim.

## Multiple Definitions

An identity MAY publish multiple Agent definitions with different `d` tags for different contexts, such as `agent`, `agent:ke`, or `agent:staging`.

Clients SHOULD treat `agent` as the default public definition unless a more specific definition was requested. Each definition is evaluated independently; clients MUST NOT merge capabilities from different `d` addresses unless an application explicitly requests that behavior.

## Trust Boundary

A valid event signature proves that the event publisher made the capability declaration. It does not prove:

- current availability or inventory
- successful performance
- reputation or trustworthiness
- a current price
- authority to move funds
- validity or safety of a referenced service

Clients MUST validate referenced PIPs, profiles, descriptors, schemas, and quotes before taking an economic action.
