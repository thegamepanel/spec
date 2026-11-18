---
id: ADR-0022
title: Store client addresses truncated by default
status: proposed
created: 2026-09-16
decided:
depends: []
updates: []
obsoletes: []
---

# ADR-0022: Store client addresses truncated by default

## Context

Some of the records the panel keeps are there to be read afterwards. A session record is read to tell one session from
another: whose it is, when it was last used, and whether it looks like the person whose account it belongs to. Tokens
and audit records will be read for the same reason. Each would naturally record where its request came from, and the
HTTP transport in [RFC-0010](../rfc/0010-http-transport.md) represents a client address with a single value object,
which all three will use.

A client address identifies a person, and several jurisdictions treat it as personal data in its own right. The panel
is self-hosted, so whoever runs an installation answers for what it stores, not the project.

Hashing does not avoid this. Addresses come from a small space, so every possible one can be hashed and compared in
seconds, and a plain hash of one is reversible. A keyed hash cannot be reversed that way, but nobody reading the record
can make sense of it either, and being read is the only reason the address was stored.

## Decision

I will store a client address truncated, keeping the part that identifies the network and dropping the part that
identifies the machine. An operator will be able to configure full addresses instead, or none at all. An address will
be recorded for a person to read, and nothing will use one to decide whether a request is legitimate.

## Alternatives

**Storing the full address by default.** It is the most useful form when investigating an incident. I rejected this
because it is also the most identifying form to hold for every session of every user, on an installation whose operator
may not have considered what holding it means.

**Storing a hash of the address.** I rejected this because a plain hash can be reversed by hashing every possible
address. A keyed hash cannot be reversed, but it cannot be read either, and being read is the point.

**Storing nothing.** I rejected this because sessions could then be told apart only by their user agent, and telling
them apart is what makes the record worth reading.

## Consequences

Easier:

- A record will show enough for someone to recognise their own session, or to spot one that does not belong, without
  holding the exact address it came from.
- An operator who needs full addresses, and has grounds for keeping them, will choose them deliberately rather than
  discover that the panel kept them anyway.

Harder:

- Two clients on one network will look alike, so an investigation that needs to tell them apart will depend on the
  operator having chosen full addresses beforehand.

Constrained:

- Nothing will be allowed to bind a session, a token or any other credential to the address it was created from. Binding
  would sign out anyone whose address changes, which happens to anyone moving between networks.
- Everything that records an address will follow this, starting with sessions and extending to tokens and audit
  records.
- A record holding an address will last no longer than the thing it belongs to, so the address will go when its
  session expires and is pruned.
