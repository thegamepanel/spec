---
id: ADR-0007
title: MySQL and MariaDB are the default database
status: accepted
created: 2026-03-29
decided: 2026-03-29
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0007: MySQL and MariaDB are the default database

## Context

The engine needs a database component: support for several simultaneous connections, one of them designated the
primary connection and used by default, an object-first abstraction of SQL queries, and a schema migration tool.
The component has to assume a database by default.

MySQL and MariaDB are the most common choices, and the easiest for people installing the panel to set up.

## Decision

The panel assumes MySQL, or a compatible MariaDB, by default. Other driver types, such as PostgreSQL and MSSQL, are
supported inside the database component so that one can be added later without a large refactor, but are not
exposed.

## Alternatives

No alternative to MySQL and MariaDB was weighed. PostgreSQL and MSSQL are driver types to support inside the
component later, not alternatives to them.

## Consequences

Easier:

- People installing the panel will set it up against the database they are most likely to have, or can most easily
  get.
- The query builder will use MySQL's own syntax, including `INSERT IGNORE`, `REPLACE INTO`, `ON DUPLICATE KEY UPDATE`
  and `MATCH ... AGAINST` full-text search.

Harder:

- Each supported version will need testing: the query builder against MySQL 8.0 and 8.4 and MariaDB 10.11 and 11.4,
  and connections against MySQL 8.4.
- Moving to another database will mean replacing the MySQL syntax the query and schema builders are built on, not
  only adding a driver.

Constrained:

- The connection driver will be hardcoded to MySQL, and no other driver will be reachable through configuration.

## Sources

- Issue [#26], Engine - Database, 2026-03-29: the scope of the database component, and the reason for assuming MySQL
  and MariaDB. This is `created` and `decided`.
- Issue [#27], Engine - Database - Connections, 2026-03-29: other driver types supported internally and not exposed,
  naming PostgreSQL and MSSQL.
- Issue [#29], Engine - Database - Query Builder, 2026-03-29: the MySQL-specific query features.
- PR [#30], feat(database): Add the database component, merged 2026-04-17 and squashed as [560e9ab]: the hardcoded
  driver, from "Hardcode connection driver to MySQL", and the query builder's syntax, from "Complete query builder
  with tests and CI matrix", taken from what it built rather than anticipated.
- [`.github/workflows/integration.yml` at 560e9ab](https://github.com/thegamepanel/panel/blob/560e9ab/.github/workflows/integration.yml):
  the database versions tested, taken from what was built rather than anticipated.
- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19: `INSERT IGNORE`, `REPLACE INTO` and the MySQL column types
  replaced throughout to move to another database, taken from what followed rather than anticipated.

[#26]: https://github.com/thegamepanel/panel/issues/26
[#27]: https://github.com/thegamepanel/panel/issues/27
[#29]: https://github.com/thegamepanel/panel/issues/29
[#30]: https://github.com/thegamepanel/panel/pull/30
[#51]: https://github.com/thegamepanel/panel/issues/51
[560e9ab]: https://github.com/thegamepanel/panel/commit/560e9ab
