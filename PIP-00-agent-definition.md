# PIP-00: Agent Definition

## Status

- Status: Draft
- Implementation: Required
- Scope: agent discovery and public capability declaration
- Related:
  - [PIP-01-escrow-descriptor.md](./PIP-01-escrow-descriptor.md)

## Purpose

This document defines the public agent definition event used for discovery and capability declaration.

## Event Type

- kind: `30360`
- addressable
- `d` tag: stable identifier, usually `agent`

## Discovery Inputs

An agent is discoverable through:

- a Nostr pubkey
- a normal Nostr profile event
- a relay list event
- an agent definition event

## Required Tags

- `d`
- `t`, `agent`
- `relay`
- `a`
  - reference to the default escrow descriptor event

## Recommended Tags

- `f`
  - fiat currency supported by this profile
- `t`, `<access-channel>`
  - transport or access surface advertised by this profile
  - examples: `ussd`, `web`, `api`, `telegram`
- `r`
  - operator or service URLs
- `client`
  - optional handler hint for compatible clients

## Content Schema

`content` is JSON and MUST be versioned.

Minimum expected fields:

- `version`
- `name`
- `about`
- `capabilities`
- `pricing_policy`
- `escrow`
- `updated_at`

## Capability Surface

The agent definition should describe:

- supported swap types
- supported fiat currencies
- supported payment channels
- supported settlement networks
- regions or markets served
- min and max limits
- pricing or margin policy references
- public access channels

## Multiple Profiles

An identity may publish multiple agent definitions with different `d` tags for different contexts.

Suggested examples:

- `agent`
- `agent:ke`
- `agent:staging`

Clients should treat `agent` as the default public profile unless a more specific profile is requested.

## Access Channels

An access channel is a public hint about how a counterparty or client can reach an Agent through an implementation surface.

Examples include:

- `ussd`
- `web`
- `api`
- `telegram`

Access channels are discovery metadata. They do not create a separate Pontmore identity, swap state machine, escrow descriptor, or dispute process.

When an Agent supports a constrained transport such as USSD, the agent definition SHOULD advertise it in both places:

- a repeated `t` tag for relay filtering
- `content.capabilities.access_channels`

Example:

```text
["t", "agent"]
["t", "ussd"]
```

```json
{
  "capabilities": {
    "access_channels": ["ussd"]
  }
}
```

Access channel identifiers MUST be lowercase ASCII tokens.

The agent definition MUST NOT publish private transport routing details such as phone numbers, short codes, session identifiers, API keys, webhooks, KYC identifiers, private payment instructions, or operator-only account references.

Implementations MAY publish non-sensitive service URLs through `r` tags when the URL is intended for public discovery.

## Example: USSD-Accessible Agent

This example shows an Agent that is discoverable by the default `agent` profile and advertises USSD as an access channel.

```json
{
  "kind": 30360,
  "pubkey": "<agent-pubkey>",
  "tags": [
    ["d", "agent"],
    ["t", "agent"],
    ["t", "ussd"],
    ["relay", "wss://relay.example"],
    ["a", "30361:<agent-pubkey>:escrow"],
    ["f", "<fiat-currency>"],
    ["client", "<client-id>"]
  ],
  "content": {
    "version": "PIP-00-draft",
    "name": "Example USSD Agent",
    "about": "Agent offering swaps through a USSD access channel",
    "capabilities": {
      "swap_types": ["fiat-to-btc", "btc-to-fiat"],
      "fiat_currencies": ["<fiat-currency>"],
      "payment_channels": ["mobile_money"],
      "settlement_networks": ["lightning", "bitcoin"],
      "regions": ["<region-code>"],
      "access_channels": ["ussd"],
      "limits": {
        "min": "<minimum-fiat-amount>",
        "max": "<maximum-fiat-amount>"
      }
    },
    "pricing_policy": "<pricing-policy-reference>",
    "escrow": {
      "descriptor": "30361:<agent-pubkey>:escrow",
      "notes": ""
    },
    "updated_at": "<iso8601-timestamp>"
  }
}
```
