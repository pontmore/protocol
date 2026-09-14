# PIP-02: Coordination Event Chains

## Status

- Status: Draft
- Maturity: Experimental pending independent implementation evidence
- Implementation: Required for the Swap Coordination conformance profile
- Scope: immutable coordination roots and append-only linked actions
- Related:
  - [PIP-00-agent-definition.md](./PIP-00-agent-definition.md)
  - [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)

## Purpose

A coordination is a bounded, profile-defined process between identified participants. It is represented by one immutable root event followed by append-only, cryptographically linked action events.

PIP-02 defines the shared event kernel: identity, linkage, authorization, expiry, economic safety, dispute effects, and reconstruction. A pinned coordination profile defines the application's public terms, private payload schemas, domain actions, and completion rules.

```text
PIP-02 coordination event chain
        +
versioned coordination profile
        =
application-specific lifecycle
```

`Coordination` is protocol terminology. Applications retain their domain language. A PontSwap coordination remains a `swap`; a PactAgent coordination may remain a `service agreement`.

PIP-02 is not a universal workflow language. Profiles are specifications and schemas, not remotely downloaded executable code.

## Event Types

PIP-02 defines only two event kinds:

- kind `7300`: immutable coordination root
- kind `7301`: immutable coordination action

Evidence, disputes, and notes do not receive separate event kinds. Evidence is referenced by an action, dispute changes are actions, and operational notes remain private or application-local. Snapshots are non-canonical indexer materialized views.

## Coordination Root

The root event ID is the `coordination_id`. The event `created_at` is its creation time, and its signer is the proposing participant. These facts MUST NOT be duplicated in content.

### Required tags

A root MUST contain:

- one role-bearing `p` tag for each participant and authority
- one exact-event `e` tag for the accepted escrow descriptor revision
- one addressable `a` tag for the descriptor coordinate

Participant tags use:

```text
["p", "<pubkey>", "<relay-hint>", "<profile-defined-role>"]
```

The relay hint MAY be empty. Application roles MUST be namespaced and defined by the pinned profile. PIP-02 reserves these authority roles:

- `core/escrow`
  - pubkey authorized to publish `core/secure`, `core/settle`, and `core/refund`
- `core/resolver`
  - pubkey authorized to publish `core/resolve_dispute`

A root MUST bind exactly one `core/escrow` pubkey. It MUST bind exactly one `core/resolver` pubkey when the profile permits `core/open_dispute`. A pubkey MAY hold more than one role only when the profile explicitly permits it.

Escrow references use:

```text
["e", "<descriptor-event-id>", "<relay-hint>", "escrow-version"]
["a", "30361:<publisher-pubkey>:<descriptor-d-tag>", "<relay-hint>", "escrow"]
```

The exact event and address references MUST identify the same descriptor. The descriptor event `created_at` MUST NOT be later than the root event `created_at`, and the descriptor MUST have been selectable under [PIP-01](./PIP-01-escrow-descriptor.md) when the root was created.

Before publishing `core/accept`, each accepting participant MUST verify that the exact descriptor revision remains current and unexpired at the action's `created_at`, and MUST validate the bound `core/escrow` and `core/resolver` identities under the accepted service schema or profile. The descriptor publisher is not implicitly either authority unless the schema or profile says so. Once acceptance is complete, later descriptor replacement or expiry does not alter the coordination.

### Required content

`content` MUST be a JSON object containing:

- `version`
  - PIP-02 content version; this generalized format is version `2`
- `profile`
  - profile identifier in the form `<namespace>/<name>@<positive-integer-version>`
- `terms`
  - object validated by the pinned profile
- `expires_at`
  - Unix timestamp after which an unaccepted coordination may expire

`content` MAY also contain `commitments`, an object whose profile-defined keys map to commitment objects. Each commitment object MUST contain `algorithm` and `digest`. Algorithm identifiers MUST include their own version.

PIP-02 initially defines `sha256-bytes@1`: hash the exact committed byte sequence with SHA-256, and encode `digest` as `sha256:` followed by 64 lowercase hexadecimal characters. The profile MUST define the meaning and media type of the bytes under each commitment key. No JSON reserialization or implicit canonicalization is permitted.

Example shape:

```json
{
  "version": 2,
  "profile": "pontmore/swap@1",
  "terms": {
    "direction": "fiat_to_btc",
    "fiat": {
      "currency": "KES",
      "amount": "1000"
    },
    "bitcoin": {
      "amount": "60000",
      "unit": "sat",
      "network": "lightning"
    },
    "payment_channel": "mpesa-ke-kes@1",
    "deadlines": {
      "fiat_pay_by": 1780001800,
      "fiat_confirm_by": 1780003600
    }
  },
  "expires_at": 1780000000
}
```

Every root MUST pin one exact profile ID and version. Implementations MUST reject unsupported PIP-02 or profile versions before acceptance or economic action. An active coordination MUST NOT adopt a later profile revision.

The profile MUST define its `terms`, participant roles, accepting roles, deadlines and recovery conditions, and any commitment keys and algorithms it permits. Commitment algorithms and canonical encodings MUST be explicitly versioned; a label without a defined canonicalization algorithm is invalid.

## Coordination Action

An action event records one claim. Its signer identifies the actor. Its event `created_at` is the action time.

### Required tags

Every action MUST contain exactly:

- one root reference:

```text
["e", "<coordination-root-id>", "<relay-hint>", "root"]
```

- one predecessor reference:

```text
["e", "<predecessor-event-id>", "<relay-hint>", "prev"]
```

For the first action, the predecessor is the root event. Later actions reference the immediately preceding action. Relay hints MAY be empty.

An action MAY repeat role-bearing `p` tags as routing hints. The root remains the canonical participant and role binding; action tags MUST NOT add or change authority.

### Required content

`content` MUST contain:

- `version`
  - value `2`
- `action`
  - a kernel or pinned-profile action identifier

`content` MAY also contain `data`, whose complete schema is defined by this PIP for a kernel action or by the pinned profile for a profile action. Evidence and private-payload references are expressed only through that action-defined schema; there is no second generic extension object.

Any kernel action MAY include `data.evidence`, an array of reference objects. Each object MUST contain:

- `type`
  - `event`, `commitment`, or `opaque`
- `value`
  - event ID, versioned digest, or service-defined opaque reference, respectively

Profiles MAY narrow the permitted evidence types. They MUST NOT interpret an evidence reference as proof merely because it is present.

Example:

```json
{
  "version": 2,
  "action": "core/accept"
}
```

The action content MUST NOT duplicate the coordination ID, actor, creation time, or predecessor. Human-readable reasons are optional application data, never a condition for validating a kernel action.

## Chain Validation

To accept an action, an implementation MUST:

1. validate the root and action Nostr event IDs and signatures;
2. require the action's `root` reference to identify the root being evaluated;
3. require the `prev` reference to identify the current chain tip;
4. reject duplicate event IDs and replayed semantic actions where this PIP or the profile permits the action only once;
5. derive signer authority from the root's bound pubkeys and the pinned profile;
6. validate the action and its `data` under this PIP or the pinned profile;
7. enforce the kernel invariants before deriving the next state.

Relay arrival order and event timestamps do not establish chain order. The `prev` link does.

State is derived by replaying the validated root and actions. No snapshot, database row, private message, relay order, or operator note overrides the validated chain.

## Kernel Actions

PIP-02 reserves the `core/` namespace. It defines these actions:

- `core/accept`
  - a profile-designated participant accepts the pinned terms
- `core/decline`
  - a profile-designated participant declines before acceptance completes
- `core/secure`
  - the bound escrow authority confirms the profile's security condition
- `core/authorize_settlement`
  - a profile-designated completion authority permits settlement
- `core/settle`
  - the bound escrow authority confirms the final settled outcome
- `core/authorize_refund`
  - a profile-designated recovery authority permits refund
- `core/refund`
  - the bound escrow authority confirms the final refunded outcome
- `core/cancel`
  - an actor authorized by the profile or a dispute resolution cancels before a final economic outcome
- `core/expire`
  - an actor authorized by the profile records that a profile-defined deadline and recovery condition elapsed
- `core/open_dispute`
  - a profile-authorized participant opens a dispute and freezes ordinary progression
- `core/resolve_dispute`
  - the bound resolver records a permitted resolution effect

Profiles MUST NOT redefine these meanings or use the `core/` namespace for new actions.

`core/open_dispute.data` MAY also contain `class`, whose values are defined by the pinned profile.

The `core/resolve_dispute` data MUST contain:

- `policy`
  - identifier of the policy bound through the profile, root terms, or accepted escrow service
- `effect`
  - one of `resume`, `authorize_settlement`, `authorize_refund`, or `cancel`

It MAY also contain `evidence` under the common kernel evidence-reference schema. Apart from `core/open_dispute.data.class`, these resolution fields, and the common `evidence` field, other kernel-action data fields are invalid.

PIP-02 validates resolver authority, linkage, and the resulting sequence. It does not decide whether evidence is credible or which participant deserves funds.

## Kernel Invariants

Every compatible profile and implementation MUST enforce:

- one immutable root and one linear, append-only action chain
- exact PIP-02 and profile version binding
- signer authorization from participants and authorities bound by the root and validated under the profile and accepted escrow service
- no action after a terminal outcome
- no `core/secure`, `core/settle`, or `core/refund` except by the bound `core/escrow` authority
- no `core/authorize_settlement` before `core/secure`
- no `core/settle` before `core/authorize_settlement` or a valid dispute-resolution effect of `authorize_settlement`
- no `core/refund` before `core/authorize_refund` or a valid dispute-resolution effect of `authorize_refund`
- `core/settle` and `core/refund` are mutually exclusive final economic outcomes
- no ordinary profile progress or economic action while disputed
- resolution only by the bound `core/resolver` authority
- explicit profile-defined expiry and recovery paths
- rejection of unsupported versions, invalid commitments, duplicates, replays, and out-of-order actions

`declined`, `cancelled`, `expired`, `settled`, and `refunded` are terminal. A profile MAY define additional terminal profile states but MUST NOT weaken a kernel invariant or create a second final economic outcome.

Profiles determine who may authorize settlement or refund and which profile conditions permit those authorizations. They may narrow kernel permissions, but they MUST NOT grant signing authority to a pubkey that was not bound by the root or accepted escrow service.

## Fork Handling

If two otherwise valid actions reference the same predecessor, the coordination is forked. Implementations MUST:

- retain both actions as evidence of the fork
- freeze further economic action
- refuse to select a winner by relay order, `created_at`, or event-ID ordering
- recover only through the bound escrow or resolver authority under the accepted service and profile rules

A recovery record MUST NOT pretend that either fork branch became canonical merely through publication order. If the accepted escrow service cannot safely reconcile the fork, the coordination remains frozen.

## Disputes and Evidence

Opening a dispute freezes ordinary profile progress, settlement authorization, settlement, and refund until `core/resolve_dispute` supplies a valid effect. A valid effect changes the derived state as follows:

- `resume` returns to the pre-dispute state
- `authorize_settlement` derives `settlement_authorized`, after which only the bound escrow may record `core/settle`
- `authorize_refund` derives `refund_authorized`, after which only the bound escrow may record `core/refund`
- `cancel` derives the terminal `cancelled` state

A resolution never moves funds. Final settlement and refund still require the bound escrow authority's corresponding action.

Domain-specific dispute classes and evidence requirements belong to the pinned profile. Resolver selection, reputation, Sybil resistance, staking, bonding, evidence evaluation, private investigation, and default timeout winners are outside Pontmore.

Public action content SHOULD contain only the minimum facts needed to verify the chain. Evidence SHOULD be referenced by a hash, opaque pointer, or encrypted payload reference. Raw invoices, bank details, screenshots, documents, Cashu tokens, preimages, payment credentials, and internal notes MUST NOT be published in PIP-02 content.

Applications MAY exchange private payloads through Nostr Gift Wrap or another profile-defined private channel. Private messages are supplementary: they do not replace, reorder, or override the public chain. When a public action depends on a private payload, the profile MUST define how the payload is committed to and verified.

## Coordination Profiles

A coordination profile MUST define:

- its identifier and version
- terms schema
- participant roles and authorization for every profile action
- accepting roles and when acceptance is complete
- profile action vocabulary and action-data schemas
- profile-state reconstruction
- settlement- and refund-authorization conditions
- deadlines, expiry, and recovery paths
- domain dispute classes and permitted resolution effects
- private payload schemas and commitment rules, when used

A profile MUST be a stable specification that implementations can validate without executing downloaded code. It MUST NOT weaken PIP-02 invariants.

### [`pontmore/swap@1`](./profiles/swap-v1.md)

`pontmore/swap@1` preserves the application term `swap`. It defines swap-specific terms such as direction, fiat amount and currency, Bitcoin amount, unit and network, payment-channel constraints, and swap deadlines.

The profile owns these swap actions:

```text
swap/fiat_sent
swap/fiat_confirmed
```

It also owns swap-specific dispute classes such as `fiat_not_received` and `incorrect_fiat_amount`. These names are not kernel actions or universal Pontmore states.

A typical swap reconstruction is:

```text
core/accept
core/secure
swap/fiat_sent
swap/fiat_confirmed
core/authorize_settlement
core/settle
```

The complete experimental profile specification, including its terms, roles, authorization table, recovery rules, and required vectors, is maintained at [`profiles/swap-v1.md`](./profiles/swap-v1.md). Implementations MUST use that specification rather than infer semantics from the example above.

## Version 1 Compatibility

Earlier draft PIP-02 documents described swap-specific version `1` events. Version `2` does not silently reinterpret them.

Implementations encountering a version `1` root or transition MUST validate it only under the version `1` swap specification they support. They MUST NOT mix version `1` transitions with a version `2` coordination root, translate an active chain in place, or perform an economic action under an unsupported version.

Because Pontmore remains a draft, kinds `7300` and `7301` are reused with an unambiguous content version. Kinds `7302`, `7303`, `7304`, and `30362` are not part of version `2`.

## Validation Before Stabilization

This generalized kernel remains experimental until there is evidence from:

- at least two materially different coordination profiles
- at least two independent implementations exchanging and validating events
- shared vectors for signatures, linkage, authorization, expiry, forks, settlement, and refund
- demonstrated public/private separation
- successful settlement and refund paths
- stable profile identification and version pinning

Until then, implementations SHOULD describe conformance as experimental and MUST pin exact supported versions.
