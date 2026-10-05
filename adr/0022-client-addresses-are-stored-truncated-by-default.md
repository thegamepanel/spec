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

Several records the panel keeps are worth reading afterwards. A session record is read to tell one session from another:
whose it is, when it was last used, and whether it looks like the person whose account it belongs to. Tokens and audit
records will be read for the same reason. Each would naturally record where the request came from, and the
[engine](../GLOSSARY.md#engine) already represents a client address with one value object, which all three use.

A client address identifies a person, and in several jurisdictions is treated as personal data in its own right.
The panel is self-hosted, so whoever runs an installation answers for what it stores, not the project.

Hashing is not a way out. An address is drawn from a small space, so every possible one can be hashed and compared
in seconds, leaving a hash that is reversible. Keying the hash prevents that, and leaves a value nobody reading the
record can make sense of, which is the only reason it was stored.

## Decision

We will store a client address truncated: the part identifying the network is kept, and the part identifying the machine
is dropped. An operator will be able to configure full addresses instead, or none at all. An address will be recorded
for a person to read, and nothing will use one to decide whether a request is legitimate.

## Alternatives

**Storing the full address by default.** It is the most useful form for investigating an incident. We rejected this
because it is also the most identifying form to hold for every session of every user, on an installation whose operator
may not have considered what holding it means.

**Storing a hash of the address.** We rejected this because a plain hash can be reversed by hashing every possible
address. A keyed hash cannot be reversed, but it cannot be read either, and reading it is the point.

**Storing nothing.** We rejected this because sessions could then be told apart only by the user agent, and telling them
apart is what makes the record worth reading.

## Consequences

Easier:

- A record will show enough for someone to recognise their own session, or spot one that does not belong, without
  holding the exact address it came from.
- An operator who needs full addresses, and has a basis for keeping them, will choose them deliberately rather than
  discovering the panel kept them anyway.

Harder:

- Two clients behind one network will look alike, so an investigation needing to tell them apart depends on the
  operator having chosen full addresses beforehand.

Constrained:

- Nothing may bind a session, a token or any other credential to the address it was created from. Binding logs out
  anyone whose address changes, which happens to anyone moving between networks.
- Everything that records an address will follow this, starting with sessions and extending to tokens and audit
  records.
- A record holding an address will live no longer than the thing it belongs to, so the address will go when the
  session expires and is pruned.
