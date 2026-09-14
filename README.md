# Pontmore Protocol PIPs

Pontmore is a Nostr-native protocol family for Agent identity, capability discovery, escrow declaration, and reconstruction of bounded economic coordinations from signed event chains.

This repository is the canonical landing page for the Pontmore protocol and contains the `PIP` series: `Pontmore Improvement Proposal` documents that define the protocol family.

## Terminology Note

In this repository, `Agent` refers to a Pontmore protocol participant that publishes capabilities and can take part in coordinations.

References to automation working on this repository are written explicitly as `AI agent`, `coding assistant`, or `repository automation`.

`Coordination` is the generic PIP-02 protocol term. Applications retain their own domain language: a PontSwap object remains a `swap`, while another application may call its object a `service agreement`.

## Active PIP Series

The active draft series contains three PIPs:

- [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - public capability discovery and protocol-resource references
- [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)
  - expiring escrow compatibility and service-schema descriptor
- [PIP-02-coordination-event-chains.md](./PIP-02-coordination-event-chains.md)
  - experimental immutable roots and append-only linked actions

PIP-02 coordination generalization remains experimental until validated by materially different profiles and independent implementations.

## Conformance Profiles

Pontmore conformance is declared by profile rather than by requiring every implementation to implement every PIP.

| Conformance profile | Required specifications | Provides |
| --- | --- | --- |
| Discovery | PIP-00 | Publish and discover versioned Agent capabilities |
| Escrow Discovery | PIP-00 and PIP-01 | Discover Agents and compatible, unexpired escrow descriptors |
| Swap Coordination | PIP-00, PIP-01, PIP-02, and [`pontmore/swap@1`](./profiles/swap-v1.md) | Reconstruct and validate a swap coordination chain |

An implementation MUST identify every conformance profile and specification version it supports. Supporting one profile does not imply support for another.

The [`pontmore/swap@1`](./profiles/swap-v1.md) profile is not a fourth PIP. It supplies swap-specific terms, roles, actions, and completion rules to the PIP-02 kernel while preserving `swap` as application vocabulary. [The profile registry](./profiles/README.md) defines how Pontmore-maintained profiles are laid out and versioned.

## Design Boundary

Pontmore standardizes:

- Nostr pubkeys as Agent and participant identities
- signed capability and escrow discovery records
- immutable coordination roots and cryptographically linked actions
- shared authorization, replay, fork, expiry, and economic-outcome invariants
- explicit public references and commitments to selected protocol resources

Versioned capability and coordination profiles define domain-specific facts and actions.

Applications and services own:

- execution and private payload transport
- offers, quotes, and business policy
- service operations and internal state
- operator accounts, dashboards, indexes, queues, and databases
- moderation, reputation, evidence evaluation, and private investigation

A signed advertisement proves authorship, not availability, performance, solvency, reputation, price, or trustworthiness.

## Implementation Guidance

To build or review a conforming implementation:

1. Read this README and select a conformance profile.
2. Read that profile's required PIPs in numerical order.
3. Treat public Agent definitions, escrow descriptors, coordination roots, and coordination actions as protocol state.
4. Treat operator accounts, indexes, materialized snapshots, private evidence, sessions, API keys, queues, and webhooks as implementation overlays.
5. Pin exact PIP, capability-profile, coordination-profile, descriptor-event, commitment, and quote versions before economic action.
6. Call any behavior not specified by the selected PIPs and profiles an implementation assumption, not Pontmore behavior.

## Contribution

This repository is for protocol specification work, not application implementation.

Changes should remain protocol-focused, atomic by component, and free of application-specific database, deployment, or UX details.

When contributing:

1. Read this README first.
2. Read only the PIPs directly relevant to the requested change.
3. Keep one PIP as the primary source of truth for each rule.
4. Update related PIPs only where consistency requires it.
5. Update this README when adding, removing, or renumbering a PIP or conformance profile.

If a design is unresolved, document it as draft or experimental rather than implying finality. Keep the repository small and avoid auxiliary process documents unless they are explicitly needed.
