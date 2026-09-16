---
id: ADR-0011
title: Identifiers are ULIDs generated in PHP
status: accepted
created: 2026-05-14
decided: 2026-05-14
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0011: Identifiers are ULIDs generated in PHP

## Context

Entities need an identity model, and every entity type needs an identifier of its own.

Some rows need their identifier to exist before they are inserted, as outbox rows and queued jobs do.

The panel depends on `symfony/uid`, added for generating identifiers.

## Decision

Entity identifiers are ULIDs, generated in PHP through `symfony/uid` before the row is inserted. Each entity type has
its own identifier class, extending an abstract readonly `EntityId`. A ULID is validated when an identifier is
constructed, and `EntityId::make()` generates a new identifier.

## Alternatives

No alternatives were weighed.

## Consequences

Easier:

- An identifier will exist before its row is inserted, as outbox rows and queued jobs need.
- Generating identifiers will not depend on the database, or on the features of any version of it.

Harder:

- A ULID will not be usable as a secret. It embeds a timestamp and a monotonic counter, so one is guessable from a
  known one, and identifiers that act as bearer credentials, such as session identifiers, will need a different
  generator.

Constrained:

- Every entity will be identified by a ULID generated in PHP, never by a value the database generates.

## Sources

- Issue [#32], Engine - Entities, revision of 2026-05-14 in its edit history: the `EntityId` design and the decision,
  which is both `created` and `decided`. Its original text of 2026-04-18 asks only for a simple ORM, with no identity
  model.
- [`composer.json` at 7fad8cd](https://github.com/thegamepanel/panel/blob/7fad8cd/composer.json), the initial
  commit: `symfony/uid`, added for generating identifiers before the design was written.
- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19: the reason, that an identifier must exist before the insert
  for outbox rows and queued jobs, and generation in PHP not needing any database feature. It was first written down
  here, three months after the decision.
- Issue [#67], Engine - Sessions, 2026-08-22: why a ULID is not usable as a session identifier, taken from what
  followed rather than anticipated.

[#32]: https://github.com/thegamepanel/panel/issues/32
[#51]: https://github.com/thegamepanel/panel/issues/51
[#67]: https://github.com/thegamepanel/panel/issues/67
