---
id: ADR-0016
title: PostgreSQL 16 is the only supported database
status: accepted
created: 2026-08-19
decided: 2026-08-19
backfilled: 2026-09-16
depends: []
updates: []
obsoletes: [ADR-0007]
---

# ADR-0016: PostgreSQL 16 is the only supported database

## Context

The panel assumes MySQL, or a compatible MariaDB, and supports other drivers inside the database component without
exposing them, per [ADR-0007](0007-mysql-and-mariadb-are-the-default-database.md). MySQL was chosen as the simplest
assumption for people setting the panel up.

The panel is built from scratch and distributed as a binary it controls, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), so what it runs against is the panel's
choice rather than whatever a host happens to provide. Nothing needs it to run on more than one database.

Subsystems that are designed but not built need things a database either provides or does not: delivery of events
between processes, a queue claiming jobs without two workers taking the same one, a scheduler electing one instance
to run a due task, and policies that restrict which rows a request can see.

## Decision

PostgreSQL is the panel's only supported database, at version 16 or above. MySQL and MariaDB are no longer
supported, and the engine commits to one dialect rather than abstracting over two.

The floor is 16 because it is the lowest default across every currently supported mainstream distribution, so nobody
self-hosting has to add a third-party package repository, and it is supported upstream until November 2028.

## Alternatives

**Staying on MySQL or MariaDB**, as before. The reason recorded for moving is that PostgreSQL offers more of what
the panel needs, and that nothing forces the simpler assumption MySQL was chosen for, now that the panel is built
from scratch and ships as its own binary. The comparison itself is not recorded.

**Supporting both**, by abstracting over the two dialects. There is no requirement to run anywhere but where the
panel is installed, so the abstraction would cost more than it gives, and neither dialect would be used properly.

## Consequences

Easier:

- Subsystems that are already designed will be built on what the database provides: notifications for delivering
  events, skip-locked reads for claiming queued jobs, advisory locks for electing a scheduler, and session variables
  carrying the identity that row-level policies read.
- Schema changes will run inside transactions, so a migration that fails part way rolls back entirely, which makes
  the migration runner simpler.
- Nobody self-hosting will have to add a package repository to install a supported version.

Harder:

- Everything built on MySQL's dialect will have to be replaced, not merely extended: its types, its upsert and
  replace statements, and its full-text search.
- The database layer's tests will have to be re-earned rather than preserved, since the layer is rewritten.

Constrained:

- Features newer than the floor cannot be relied on. `JSON_TABLE` arrives in 17, and native `uuidv7()` and virtual
  generated columns in 18. None is needed: identifiers are generated in PHP, per
  [ADR-0011](0011-identifiers-are-ulids-generated-in-php.md), so one exists before its row is inserted.
- The panel will run against one database, and a deployment wanting another is not supported.

## Sources

- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19, whose revisions only reformatted its list of cards: the
  decision, the version floor and the distributions it follows, upstream support until November 2028, what the floor
  costs and why none of it is needed, committing to one dialect rather than abstracting over two, what changes in
  types, queries and connections, and each primitive with the consumer that needs it. This is `created` and
  `decided`; the decision came from a session before that date, which is not recalled.
- Issue [#54], Engine - Database - PostgreSQL test infrastructure and CI, 2026-08-19: the floor run locally and the
  newer versions covered in continuous integration, and the layer's coverage and mutation score being re-earned.
- Issue [#59], Engine - Database - Schema: tables, indexes and DDL objects, 2026-08-19: schema changes running inside
  transactions, and the two statements that cannot.
- PostgreSQL having considerably more features and functionality that the panel benefits from, and MySQL having been
  the simpler assumption for a panel built from scratch and distributed as its own binary: first written down on
  2026-09-16.

[#51]: https://github.com/thegamepanel/panel/issues/51
[#54]: https://github.com/thegamepanel/panel/issues/54
[#59]: https://github.com/thegamepanel/panel/issues/59
