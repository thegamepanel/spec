---
id: ADR-0016
title: Support only PostgreSQL 16 and above
status: accepted
created: 2026-08-19
decided: 2026-08-19
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: [ADR-0007]
---

# ADR-0016: Support only PostgreSQL 16 and above

## Context

The panel assumes MySQL, or a compatible MariaDB, and supports other drivers inside the database component without
exposing them, as [ADR-0007](0007-mysql-and-mariadb.md) records. MySQL was chosen as the
simplest assumption for people setting the panel up.

The panel is built from scratch and ships as a binary it controls, as
[ADR-0009](0009-frankenphp-worker.md) records, so the database it runs against
is the panel's choice rather than whatever a host happens to provide. Nothing requires it to run on more than one
database.

Several subsystems are designed but not built, and each needs something a database either provides or does not:
delivering events between processes, a queue in which two workers never claim the same job, a scheduler that elects one
instance to run a due task, and policies that restrict which rows a request can see.

## Decision

I will make PostgreSQL the panel's only supported database, at version 16 or above, and drop MySQL and MariaDB, so that
the [engine](../GLOSSARY.md#engine) commits to one dialect rather than abstracting over two. The floor is 16 because it
is the lowest default across every currently supported mainstream distribution, so nobody self-hosting has to add a
third-party package repository, and because it is supported upstream until November 2028.

## Alternatives

**Staying on MySQL or MariaDB**, as before. I rejected this because PostgreSQL offers more of what the panel needs, and
nothing still forces the simpler assumption MySQL was chosen for, now that the panel is built from scratch and ships as
its own binary. The comparison itself is not recorded.

**Supporting both**, by abstracting over the two dialects. I rejected this because nothing requires the panel to run
anywhere but where it is installed, so the abstraction would cost more than it gives, and neither dialect would be used
properly.

## Consequences

Easier:

- The subsystems already designed will be built on what the database provides: notifications to deliver events,
  skip-locked reads to claim queued jobs, advisory locks to elect a scheduler, and session variables carrying the
  identity that row-level policies read.
- Schema changes will run inside transactions, so a migration that fails part way will roll back entirely, which will
  make the migration runner simpler.
- Nobody self-hosting will have to add a package repository to install a supported version.

Harder:

- Everything built on MySQL's dialect will have to be replaced, not merely extended: its types, its upsert and replace
  statements, and its full-text search.
- The database layer's tests will have to be re-earned rather than preserved, since the layer will be rewritten.

Constrained:

- Features newer than the floor will not be available to rely on. `JSON_TABLE` arrives in 17, and native `uuidv7()`
  and virtual generated columns in 18. None is needed: identifiers are generated in PHP, as
  [ADR-0011](0011-ulid-identifiers.md) records, so one exists before its row is inserted.
- The panel will run against one database, and an installation wanting another will not be supported.

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
