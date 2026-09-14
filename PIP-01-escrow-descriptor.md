# PIP-01: Escrow Descriptor

## Status

- Status: Draft
- Implementation: Required for the Escrow Discovery and Swap Coordination conformance profiles
- Scope: public escrow compatibility and service-schema discovery
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)

## Purpose

This document defines the public escrow descriptor event referenced by Agent definitions and coordination roots.

An escrow descriptor is a compatibility object. It identifies an escrow mechanism, its supported networks, its selection lifetime, and an optional machine-readable service schema. It is not an escrow service contract, administrative state machine, fee language, or record of an escrow instance.

## Event Type

- kind: `30361`
- addressable
- `d` tag: stable identifier for one escrow configuration

## Minimum Content

`content` MUST be a JSON object with these fields:

- `version`
  - integer descriptor schema version
- `escrow_type`
  - non-empty, lowercase mechanism identifier
- `networks`
  - non-empty array of lowercase settlement or invoice network identifiers
- `expires_at`
  - Unix timestamp after which the descriptor cannot be selected for a new coordination

The descriptor MAY include `service` as defined below.

Example:

```json
{
  "version": 1,
  "escrow_type": "cashu_escrow",
  "networks": ["cashu"],
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://example.com/escrow-v1.json"
    }
  },
  "expires_at": 1780000000
}
```

The event MUST include a `d` tag. Each value in `content.networks` SHOULD also appear as a repeated `t` tag for relay filtering:

```text
["d", "cashu-main"]
["t", "pontmore-network:cashu"]
```

`content.networks` is canonical. Clients MUST reject a descriptor when its `pontmore-network:` tags claim values absent from `content.networks`.

## Descriptor Lifecycle

A descriptor is selectable for a new coordination only when:

- it is the current addressable event at its kind, pubkey, and `d` coordinate; and
- the client's validation time is earlier than `expires_at`.

The publisher renews or changes a descriptor by republishing the addressable event. The publisher immediately expires it by publishing a replacement whose `expires_at` is not later than the replacement event's `created_at`.

A coordination MUST bind both the descriptor coordinate and the exact descriptor event ID it accepted. Republishing or expiring the address does not alter an existing coordination.

Replacement pointers, disable reasons, operator-facing status, and administrative history are outside PIP-01.

## Service Schema

When `service` is present, it MUST contain only `schema`. `service.schema` MUST contain:

- `type`
  - `openapi` or `asyncapi`
- `url`
  - absolute `https://` URL for the schema artifact

The referenced schema owns all service behavior, including:

- participant creation and binding
- transport, endpoints, authentication, and authorization
- funding instructions, funding status, and partial-funding recovery
- release, refund, cancellation, and partial outcomes
- idempotency, errors, and reference formats
- resolver binding and authorization
- supported dispute-resolution effects
- evidence-submission operations
- timeout and recovery behavior
- reconciliation
- fees and exact quotes

Clients MUST validate the referenced schema before relying on service behavior. A descriptor without `service.schema` supplies compatibility facts but no PIP-01 service interface.

Additional schema languages MAY be added by a later revision when concrete interoperability requires them. Clients MAY reject an unsupported schema type or version.

### Schema Fetch Safety

Clients MUST apply fetch-safety checks before dereferencing `service.schema.url`.

The URL:

- MUST use `https://`
- MUST NOT resolve to a private, loopback, link-local, multicast, or otherwise unsafe destination
- SHOULD identify an immutable or versioned artifact

Clients SHOULD apply bounded fetches, redirect limits, content-type checks, and response-size limits. They MAY reject mutable or unsafe schema URLs and schemas outside their trust or capability requirements.

## Fees and Quotes

PIP-01 does not define a fee-calculation language. A descriptor MUST NOT require clients to derive an exact charge from a mutable pricing formula.

When a service charges a fee or the economic terms can vary, its schema SHOULD define a signed, expiring quote containing exact amounts. The schema owns the quote format and signature verification rules. A coordination root binds the accepted quote through its `commitments.quote` field; it does not copy a mutable pricing policy from the descriptor.

## Public and Private Boundary

The descriptor is public protocol state. It MUST NOT include:

- wallet or custody-backend identifiers
- private credentials, API keys, or bearer secrets
- private payment or payout instructions
- raw invoices or Cashu token strings
- settlement secrets or preimages
- internal routing, review, scoring, or reconciliation records
- evidence payloads

Those facts belong in private application channels or in the service represented by the referenced schema.

## Escrow Types

The initial escrow type identifiers are:

- `lightning_hold_invoice`
  - `networks` MUST include `lightning`
- `custodial_escrow`
  - network-generic; `networks` declares the supported networks
- `cashu_escrow`
  - `networks` MUST include `cashu`

These identifiers communicate mechanism compatibility only. Subtype operations, reference formats, funding rules, timeouts, dispute behavior, and settlement mechanics belong to the referenced service schema or the selected coordination profile.

## Selection Rules

Before selecting a descriptor, a client MUST validate:

- the event signature and address
- the descriptor version
- `expires_at`
- the required network and escrow type
- any service-schema requirements of the selected coordination profile

For a service-backed flow, the client MUST also support and validate `service.schema`.

A client MUST NOT infer availability, solvency, custody safety, resolver trustworthiness, or successful operation from a signed descriptor. Those are application trust decisions, not PIP-01 facts.

## Open Questions

1. Which additional schema languages have demonstrated interoperability need?
2. Does any compatibility fact need relay filtering strongly enough to justify a new canonical descriptor field rather than remaining in a service schema?
