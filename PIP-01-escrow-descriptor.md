# PIP-01: Escrow Descriptor

## Status

- Status: Draft
- Implementation: Required
- Scope: public escrow interop descriptor for discovery, compatibility, and service schema discovery
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)
  - [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md)

## Purpose

This document defines the public escrow descriptor event referenced by agent definitions and swaps.

An escrow descriptor is a compatibility object. It declares the public facts needed to discover an escrow configuration, decide whether a client can use it, reference it from Pontmore swap flows, and locate the service schema.

PIP-01 does not define escrow service behavior. Internal state machines, endpoint contracts, authentication protocols, request payloads, response payloads, error formats, and operation-specific authorization rules belong to the referenced service schema.

## Event Type

- kind: `30361`
- addressable
- `d` tag: stable identifier for one escrow configuration

## Function

The escrow descriptor tells counterparties and operators:

- what escrow mechanism is used
- which settlement or invoice networks are supported
- what public funding and dispute rules are advertised
- how the escrow instance is referenced
- what timeout and dispute assumptions apply
- where to find the service schema, if one is advertised

The public swap lifecycle remains defined by [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md). Dispute and timeout fallback policy remains defined by [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md).

## Minimum Content

`content` is JSON and MUST be versioned.

Minimum expected fields:

- `version`
- `escrow_type`
- `networks`
- `funding_rules`
- `dispute_rules`
- `reference_format`
- `updated_at`

## Descriptor Use Levels

An escrow descriptor has two intended use levels:

- **compatibility and discovery** - the descriptor declares which networks, assets, reference formats, funding rules, and dispute policy an escrow supports, so that agents and swaps can select it
- **service schema discovery** - the descriptor additionally points to a machine-readable schema

A descriptor that omits the `service` block is sufficient for compatibility and discovery and for use inside Pontmore swap flows where the swap state machine in [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md) carries public execution state.

A descriptor that includes a `service` block does not make PIP-01 the source of truth for service behavior. PIP-01 only declares `service.schema`. Clients MUST validate the referenced schema before relying on any service behavior.

## Service Schema

The optional `service` content field contains only the service schema pointer.

When `service` is present, it MUST contain only:

- `schema`
  - object containing the service schema pointer
- `schema.type`
  - schema language identifier
  - initial supported values are `openapi` and `asyncapi`
- `schema.url`
  - absolute `https://` URL for the schema artifact

Initial PIP-01 schema types are intentionally limited to:

- `openapi`
- `asyncapi`

`smithy`, `protobuf`, and generic `json` schema pointers are out of scope for the initial registry. They MAY be added by a later PIP or revision if there is concrete interoperability need.

### Schema Requirements

The referenced schema MUST define all service behavior, including:

- transport or protocol
- endpoint or server information
- authentication
- operations
- request payloads
- response payloads
- error format
- operation-specific authorization rules

If the service exposes status values, operation names, funding flows, release decisions, split outcomes, idempotency keys, or participant-binding rules, those details MUST be defined by the referenced schema. They are not canonical PIP-01 protocol state.

### Schema Fetch Safety

Clients MUST apply schema fetch safety checks before dereferencing `service.schema.url`.

The schema URL:

- MUST use `https://`
- MUST NOT resolve to private, loopback, link-local, multicast, or otherwise unsafe network destinations
- SHOULD resolve to an immutable or versioned schema artifact

Clients SHOULD use bounded fetches, redirect limits, content-type checks, and response-size limits when retrieving schemas.

Clients MAY reject unsupported schema types, unsupported schema-language versions, unsafe schema URLs, mutable schema URLs, or schemas that do not match the client's trust and capability requirements.

## Public/Private Boundary

The descriptor is public protocol state. It MUST NOT include:

- wallet identifiers
- custody backend identifiers
- private payment credentials
- internal account details
- private routing state
- operator-internal API keys or bearer secrets
- private payment instructions
- raw invoices
- raw Cashu token strings
- settlement secrets
- internal review notes

These are operator-layer implementation details or private execution payloads. Their absence is what allows the same descriptor to be published openly without exposing operator internals.

Pontmore implementations SHOULD carry non-public execution payloads through private channels such as the companion Gift Wrap lane described in [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md).

## Network Declaration

An escrow descriptor MUST declare every settlement or invoice network supported by the escrow configuration.

The canonical content field is:

- `networks`
  - non-empty array of lowercase network identifiers
  - examples: `bitcoin`, `lightning`, `cashu`

Each value in `networks` SHOULD also be emitted as a repeated `network` tag for relay filtering:

```text
["network", "bitcoin"]
["network", "lightning"]
```

The `networks` content array is the canonical declaration. The repeated tags are an index and discovery aid. Clients MUST NOT treat a repeated `network` tag as supported unless it also appears in `content.networks`.

## Funding Rules

`funding_rules` declares descriptor-level compatibility facts about how an escrow expects funding to be satisfied.

Escrow funding is described as an `m of n` requirement.

- `funding_rules.funding_threshold` is `m`: the minimum number of declared funding participants whose funding must be confirmed before the escrow is considered funded
- `funding_rules.participant_count` is `n`: the total number of declared funding participants for the escrow

`funding_threshold` MUST be an integer greater than or equal to `1`.

`participant_count` MUST be an integer greater than or equal to `funding_threshold`.

A `1 of 1` funding rule represents a single-funder escrow. A `2 of 2` funding rule represents a two-party escrow where both declared funders must fund. A `1 of 2` funding rule represents one participant funding on behalf of a two-party escrow. Other values represent threshold funding.

The descriptor declares only the funding cardinality required for compatibility. PIP-01 does not define a canonical standalone funding state machine. The referenced service schema MUST define how participants are identified, how funding instructions are retrieved, how partial funding is represented, and how timeout cancellation or refund works.

If `participant_count` is greater than `1`, the descriptor or referenced service schema MUST define how partially funded escrows can be canceled or refunded after timeout. A descriptor MUST NOT imply that partially funded capital can remain locked indefinitely with no timeout or fallback path.

## Dispute Rules and Timeout Fallback

`dispute_rules` declares descriptor-level compatibility facts about the applicable dispute policy.

`dispute_rules.policy` MUST be a non-empty identifier for the applicable dispute policy. The canonical initial value is `pip03`, which declares conformance with [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md). Any other policy identifier MUST be defined by a later PIP or by the referenced service schema.

PIP-01 does not define release, refund, cancellation, or partial-outcome semantics. For Pontmore swaps, public release and dispute lifecycle behavior is defined by [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md) and [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md). For service use, release and refund behavior belongs to the referenced service schema.

Timeout and refund fallback metadata advertised by an escrow descriptor MUST be compatible with [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md). PIP-03 is the source of truth for timeout classes and fallback resolution policy.

When a descriptor advertises a timeout class, the descriptor or referenced service schema MUST identify the applicable fallback resolution required by PIP-03. A `mutual_consent`-only timeout path with no fallback is not a valid terminal policy.

## Reference Format

`reference_format` declares how swaps and clients refer to an escrow instance or escrow claim.

Examples include:

- `bolt11`
- `bolt11_or_custodial_escrow_reference`
- `cashu_v4_token`
- opaque service-defined references

## Descriptor Example

```json
{
  "version": 1,
  "escrow_type": "custodial_escrow",
  "networks": ["bitcoin", "lightning"],
  "funding_rules": {
    "funding_threshold": 1,
    "participant_count": 1
  },
  "dispute_rules": {
    "policy": "pip03"
  },
  "reference_format": "bolt11_or_custodial_escrow_reference",
  "service": {
    "schema": {
      "type": "openapi",
      "url": "https://escrow.example.com/pontmore-escrow.openapi.json"
    }
  },
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

When `escrow_type` is `lightning_hold_invoice`, `networks` MUST include `lightning`. The descriptor MUST expose enough public compatibility data for clients to know that the escrow reference is a Lightning hold-invoice reference and to evaluate the advertised funding, dispute, timeout, and service-schema compatibility.

Raw invoice payloads, settlement secrets, preimages, and private payout instructions MUST stay out of the public descriptor. Subtype-specific service mechanics belong to the referenced service schema.

## Canonical Subtype: `custodial_escrow`

`custodial_escrow` is a canonical escrow subtype for swaps where an escrow operator takes custody of the settlement asset or invoice claim and releases or refunds it according to public protocol state or the referenced service schema.

This subtype is network-generic. The top-level `networks` array is the canonical supported-network declaration for the descriptor. Subtype-specific service mechanics belong to the referenced service schema.

Raw invoices, private payment instructions, operator account details, custody internals, and private reconciliation records SHOULD stay out of the public descriptor.

## Canonical Subtype: `cashu_escrow`

`cashu_escrow` is a canonical escrow subtype where funds are held as Cashu ecash tokens locked to the escrow operator's pubkey using NUT-11 spending conditions, with a refund pubkey and locktime.

When `escrow_type` is `cashu_escrow`, `networks` MUST include `cashu`. The descriptor MUST expose enough public compatibility data for clients to know that the escrow reference is a Cashu escrow reference and to evaluate the advertised funding, dispute, timeout, and service-schema compatibility. Subtype-specific service mechanics belong to the referenced service schema.

Raw Cashu token strings, mint credentials, preimages, and private payout instructions SHOULD stay out of the public descriptor. Only opaque references or hashes SHOULD appear in public evidence events unless targeted disclosure is required by the applicable dispute policy.

## Selection Rules

Every agent profile should declare:

- at least one usable escrow configuration
- one default escrow configuration

That declared escrow must be usable without out-of-band negotiation at swap time.

For service use, an application SHOULD select a descriptor whose `service.schema.type` is supported, whose `service.schema.url` passes fetch-safety checks, and whose referenced schema matches the application's supported capabilities and trust constraints.

A descriptor without `service.schema` provides no PIP-01 service schema pointer.

## Open Questions

1. **Additional schema types**
   - Should future revisions add `smithy`, `protobuf`, or another schema type once a concrete implementation needs it?

2. **Funding cardinality metadata**
   - What additional descriptor-level metadata, if any, is needed for clients to reject unsafe multi-party funding before fetching the service schema?

3. **Custodial accountability references**
   - Should `custodial_escrow` descriptors advertise a public accountability reference, such as proof of reserve, attestation, or collateral, without leaking operator-private implementation details?
