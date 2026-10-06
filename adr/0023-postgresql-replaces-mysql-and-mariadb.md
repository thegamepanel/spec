---
id: ADR-0023
title: Replace MySQL and MariaDB with PostgreSQL
status: accepted
created: 2026-08-07
decided: 2026-08-07
backfilled: 2026-10-06
depends: [ADR-0009]
updates: []
obsoletes: [ADR-0007]
---

# ADR-0023: Replace MySQL and MariaDB with PostgreSQL

## Context

The panel assumes MySQL, or a compatible MariaDB, as [ADR-0007](0007-mysql-and-mariadb-are-the-default-database.md)
records. MySQL is the most common database and the easiest for people installing the panel to set up, and people moving
from Pterodactyl already have a MySQL database. The panel writes SQL for the database it targets rather than a subset
portable across several: it is an application with a known infrastructure target, not a framework, and has no need to
abstract over databases.

The database component exists, with connections, a query builder and a schema builder written against MySQL. Nothing
consumes it yet: no table has been defined, and no feature stores anything. Changing database now means rewriting that
component alone. Changing later would also mean rewriting everything built on it.

[Modules](../GLOSSARY.md#module) are a headline feature of the panel, and an
[external module](../GLOSSARY.md#external-module)'s migrations are third-party code. MySQL and MariaDB commit each
schema change implicitly, so a migration that fails part way leaves the schema partly applied, with no clean way back.

Without a namespace of their own, modules' tables would share one, so two modules could choose the same table name, and
removing a module would mean finding everything it created. MySQL and MariaDB can give a module a database of its own,
but cannot create or drop one inside a transaction. PostgreSQL's schemas give each module a namespace inside one
database, and a schema can be created or dropped, with everything in it, inside a transaction.

Pterodactyl has had authorisation defects that exposed one user's servers to another, such as an authenticated user
acting against servers they had no access to ([GHSA-g8gw-6j32-8w7g]), and a user who was not an administrator listing
every server through the client API ([GHSA-6888-7f3w-92jx]). Neither MySQL nor MariaDB has row-level security.

## Decision

We will use PostgreSQL as the panel's only database, and remove support for MySQL and MariaDB entirely rather than keep
it alongside. We are choosing it for what MySQL and MariaDB lack: schema changes that run inside transactions, so that a
[module](../GLOSSARY.md#module)'s migration that fails can be rolled back rather than leave the schema partly applied,
and so that a module's schema can be created and dropped with everything in it; and row-level security, which the panel
could use as a floor beneath its own authorisation against defects of the sort Pterodactyl has had. Its schemas also
give each module a namespace of its own, free of conflicting names and removable as a whole. One database also means one
test matrix and SQL written for it, and PostgreSQL's features are there for designs to build on. The move is cheap now,
because nothing yet consumes the database component. The minimum PostgreSQL version, and how connections are held under
the worker process, are left open.

## Alternatives

**Keeping MySQL and MariaDB.** They are familiar to the game hosting community, match existing Pterodactyl
installations, and are more widely offered on budget VPS and shared hosting. We rejected this because neither provides
transactional schema changes or row-level security, and their advantages are familiarity and availability rather than
anything the panel needs from a database. Moving from Pterodactyl is a one-time export of data either way, because the
panel does not share Pterodactyl's schema.

**Supporting both.** The panel would abstract over MySQL, MariaDB and PostgreSQL, and leave the choice to whoever
installs it. We rejected this because it doubles the test matrix and limits the SQL to what the databases share, for an
audience that would mostly not use the choice, which makes it the worst of the options. Ghost supports only MySQL 8 in
production for a similar reason, keeping a narrow set of supported environments so that maintenance stays manageable
([Ghost's supported databases]).

## Consequences

Easier:

- Schema changes can run inside transactions, so a [module](../GLOSSARY.md#module)'s migration that fails can be
  rolled back rather than leave the schema partly applied. The benefit will grow with the number of modules.
- Each module could be given a schema of its own, so that its tables and data never conflict with another module's
  names, and the module can be removed by dropping its schema inside a transaction. Whether and how modules use
  schemas is not decided here.
- Row-level security will be available to the panel as a possible floor beneath its own checks, enforced by the
  database, so that an error such as a missed `WHERE` clause or an insecure direct object reference would not reach
  another user's rows. Whether and how the panel uses it is not decided here.
- One database will mean one test matrix, one set of migration behaviours, and SQL written for it rather than for what
  two databases share.
- Designs will build on PostgreSQL's features, such as `jsonb` with GIN indexes, advisory locks, `LISTEN`
  and `NOTIFY`, `RETURNING`, `ON CONFLICT`, `FOR UPDATE SKIP LOCKED` and `NOWAIT`, partial and expression indexes,
  generated columns, check constraints, and native full-text search.
- Schema conventions will follow, each to be decided by the design that owns it, such as `timestamptz`
  rather than naive datetimes, `jsonb` for schemaless options and metadata, `inet` and `cidr` for allocation addresses,
  `RETURNING` in place of a second query after an insert, an indexed `server_id` on every table scoped to a server,
  which row-level policies would need, check constraints and strict `NOT NULL` by default, and migrations run inside
  transactions.

Harder:

- Transactions will have to fit a migration design that runs in three stages, schema, then relations, then data, with
  each stage run across every migration before the next begins, so that modules can reference each other's tables
  without the migrations being ordered. Where transactions begin and end in such a run is not decided here.
- Installing PostgreSQL will be extra work for some people self-hosting, since it is less common than MySQL in game
  hosting and on budget hosts. Documentation and the installer will need to treat it as one more package beside the
  panel's single binary, per [ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md).
- Importing from Pterodactyl will be a move of data from MySQL to PostgreSQL, and a feature in its own right.
- The database component will have to be rewritten, since its syntax, types and schema building are MySQL's.

Constrained:

- The panel will run against one database, and an installation wanting MySQL or MariaDB will not be supported.

## Sources

- Planning session, 2026-08-07, not publicly available and first written down on 2026-10-06: the decision, which is
  `created` and `decided`; transactional schema changes for module migrations, row-level security as a possible floor
  against defects of the sort Pterodactyl has had, schemas as a namespace for each module, one database, and the
  features it makes usable; nothing consuming the database component yet; the alternatives and why each was rejected,
  with Ghost as prior art; the cost to people self-hosting and to importing from Pterodactyl; the schema conventions as
  possibilities; and the minimum version and connection handling left open.
- Planning session, March 2026, not publicly available and first written down on 2026-10-06: MySQL and MariaDB chosen
  to ease moving from Pterodactyl, and SQL written for one database rather than a portable subset.
- Issue [#26], Engine - Database, 2026-03-29: MySQL and MariaDB assumed as the most common database and the easiest to
  set up.
- Issue [#28], Engine - Database - Schema, 2026-03-29: migrations run in three stages, each across every migration
  before the next, so that modules can reference each other's tables without an order.
- Issue [#48], Engine - Database - Connection configuration fixes, 2026-08-08, with no edits: the first public record of
  the decision, scoping its work to changes "that survive the PostgreSQL replacement".
- [The panel at efac787], the last commit before 2026-08-07: the database component written against MySQL, with
  nothing outside it using it and no migration defined.
- Pterodactyl advisories [GHSA-g8gw-6j32-8w7g], published 2021-01-20, and [GHSA-6888-7f3w-92jx], published
  2020-07-26: the authorisation defects named in Context.
- [Ghost's supported databases], verified 2026-10-06: MySQL 8 as the only database supported in production, to keep
  maintenance manageable.

[#26]: https://github.com/thegamepanel/panel/issues/26
[#28]: https://github.com/thegamepanel/panel/issues/28
[#48]: https://github.com/thegamepanel/panel/issues/48
[The panel at efac787]: https://github.com/thegamepanel/panel/tree/efac787
[GHSA-g8gw-6j32-8w7g]: https://github.com/pterodactyl/panel/security/advisories/GHSA-g8gw-6j32-8w7g
[GHSA-6888-7f3w-92jx]: https://github.com/pterodactyl/panel/security/advisories/GHSA-6888-7f3w-92jx
[Ghost's supported databases]: https://docs.ghost.org/faq/supported-databases
