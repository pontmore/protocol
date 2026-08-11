# Pontmore Protocol PIPs

Pontmore is a Nostr-native protocol family for agent identity, capability discovery, escrow declaration and service interface, and swap lifecycle coordination.

This repository is the canonical landing page for the Pontmore protocol and contains the `PIP` series: `Pontmore Improvement Proposal` documents that define the protocol family.

## Terminology Note

In this repository, `Agent` refers to a Pontmore protocol participant that publishes capabilities and can conduct swaps.

References to automation working on this repository are written explicitly as `AI agent`, `coding assistant`, or `repository automation` to avoid confusion with the protocol term.

## PIP Series

The PIPs are ordered by dependency and implementation priority. At this stage, the repository contains only the required PIPs for the minimum interoperable protocol core.

Read `PIP-00` through `PIP-03` first.

- [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - Implementation: `Required`
  - public agent capability and discovery record

- [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)
  - Implementation: `Required`
  - public escrow declaration and standalone-capable service interface referenced by agents and swaps; the service interface is optional during swap execution

- [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)
  - Implementation: `Required`
  - request, transition, evidence, dispute, note, and snapshot event lifecycle

- [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md)
  - Implementation: `Required`
  - dispute classes, timeout classes, evidence boundary, and resolution modes

## Design Direction

Pontmore is designed around a small set of protocol positions:

- Nostr pubkey is the canonical agent identity
- agent capabilities and discovery metadata are published on Nostr
- swaps are modeled as Nostr-native state machines
- operators may provide business, compliance, indexing, automation, and UX overlays without becoming the identity root

The practical inversion is:

- from `application account owns agent`
- to `Nostr identity is agent, application accounts wrap it`

## Implementation Baseline

The first interoperable implementation baseline is:

- [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
- [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)
- [PIP-02-swap-state-machine.md](./PIP-02-swap-state-machine.md)
- [PIP-03-dispute-policy.md](./PIP-03-dispute-policy.md)

These define the public discovery model, escrow declaration and service interface model, swap event lifecycle, and dispute boundary needed for a usable Pontmore-compatible implementation.


## Contribution

This repository is for protocol specification work, not application implementation.

Changes in this repository should remain:

- protocol-focused
- atomic by component
- free of implementation-repo details
- free of application-specific database or migration logic

When contributing:

1. Read this `README` first.
2. Read only the PIPs directly relevant to the requested change.
3. Keep one PIP as the primary source of truth for each rule.
4. Update related PIPs only where consistency requires it.
5. Update this `README` when adding a new PIP.

If a design is still unresolved, prefer documenting a draft convention or open question rather than implying false finality.

The repository is intentionally small. Humans and agents should preserve that property by avoiding auxiliary process documents unless explicitly needed.
