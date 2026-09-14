# Pontmore Coordination Profiles

Coordination profiles define application semantics on top of [PIP-02](../PIP-02-coordination-event-chains.md). They are protocol specifications, not additional PIPs and not executable plugins.

## Repository Layout

A Pontmore-maintained profile identifier maps to a flat, readable Markdown filename:

```text
<namespace>/<name>@<version>
        ->
profiles/<name>-v<version>.md
```

For example, [`pontmore/swap@1`](./swap-v1.md) is defined by `profiles/swap-v1.md`. The canonical ID retains the namespace; the file path does not repeat this repository's `pontmore` namespace.

The `pontmore` namespace is reserved for profiles maintained in this repository. Rules for globally identifying externally maintained profile namespaces are not yet standardized; external profiles MUST NOT use the `pontmore` namespace.

## Required Sections

Every profile specification MUST define:

- status and exact profile identifier
- compatible PIP-02 version
- purpose and non-goals
- terms schema
- participant roles and root constraints
- action, signer, precondition, and action-data rules
- deterministic profile-state reconstruction
- settlement- and refund-authorization gates
- deadlines, expiry, and recovery paths
- dispute classes and permitted resolution effects
- private-payload and commitment rules
- terminal behavior
- conformance and adversarial vectors

A profile MAY narrow a PIP-02 permission. It MUST NOT weaken a PIP-02 kernel invariant, redefine a `core/` action, add authority not bound by the root, or require downloaded executable code.

## Lifecycle

New profiles are proposed through ordinary repository review. Experimental profiles MUST say which definitions remain unsettled and MUST NOT claim stable interoperability.

Before a profile is marked stable, it SHOULD have:

- at least two independent implementations
- successful exchange of normal settlement and refund histories
- shared signature, linkage, authorization, replay, fork, expiry, and privacy vectors

Once a profile version is stable and used for economic action, its meaning is immutable. A semantic change requires a new positive integer version and a new file. Active coordination roots remain pinned to the version they accepted.

During the experimental phase, implementations SHOULD additionally identify the repository commit used for testing. A repository commit is development provenance; it does not replace the profile ID and version in a PIP-02 root.

## Active Profiles

| Profile | Status | PIP-02 version | Purpose |
| --- | --- | ---: | --- |
| [`pontmore/swap@1`](./swap-v1.md) | Experimental | 2 | Bilateral fiat/Bitcoin swap coordination |
