# Agent Guidance

This repository contains the Pontmore protocol specifications. Pontmore is a Nostr-native protocol family for agent identity, capability discovery, escrow declaration, and swap lifecycle coordination.

## Scope

Work in this repository should stay focused on protocol specification text. Do not add application implementation plans, product UX, database schemas, deployment notes, or SDK instructions unless the user explicitly asks for protocol text that requires them.

## Terminology

In protocol text, `Agent` means a Pontmore protocol participant that publishes capabilities and can conduct swaps.

When referring to automation working on this repository, use explicit terms such as `AI agent`, `coding assistant`, or `repository automation`.

## Spec Structure

The active PIP series contains three composable PIPs:

- `PIP-00-agent-definition.md`: public Agent capability discovery and protocol-resource references
- `PIP-01-escrow-descriptor.md`: expiring public escrow compatibility and service-schema descriptor
- `PIP-02-coordination-event-chains.md`: experimental immutable coordination roots and append-only linked actions

Pontmore-maintained coordination profiles use `profiles/<name>-v<version>.md`. Their canonical IDs retain the `pontmore/` namespace. They are versioned protocol specifications, not additional PIPs or executable plugins.

Read `README.md` first, then read only the PIPs directly relevant to the requested change.

## Implementation

When reading these specs to build or review an implementation in another repository:

1. Start with `README.md` to understand the protocol definition and active PIP set.
2. Select a conformance profile and read its required PIPs in order: `PIP-00`, then `PIP-01`, then `PIP-02` where applicable.
3. Treat Nostr identity, public Agent definitions, escrow descriptors, coordination roots, coordination actions, and pinned profiles as the protocol surface.
4. Model operator accounts, dashboards, indexes, moderation tools, and private databases as implementation overlays, not canonical protocol state.
5. Keep public protocol facts separate from private operator judgments, internal notes, KYC data, payment instructions, screenshots, and local account records.
6. If an implementation needs local conveniences such as sessions, API keys, indexes, queues, or webhooks, derive them from the protocol instead of redefining the protocol around them.

Implementation plans should identify which PIP each protocol behavior comes from. If the specs are silent, call that out as an implementation assumption instead of presenting it as Pontmore behavior.

## Contribution

- Keep each change atomic by protocol component.
- Keep one PIP as the primary source of truth for each rule.
- Update related PIPs only where consistency requires it.
- Update `README.md` when adding, removing, or renumbering a PIP.
- Preserve the distinction between canonical public protocol state and operator-layer overlays.
- Prefer documenting draft conventions or open questions over implying false finality.

## Final Checks

Before finishing, verify that:

- cross-links point to real files
- PIP numbers and filenames agree
- required/recommended/optional labels match the intended implementation burden
- new normative statements do not contradict related PIPs
- no product-specific assumptions have leaked into protocol text
