---
id: ADR-0024
title: Give each external module its own schema
status: proposed
created: 2026-10-06
decided:
depends: [ADR-0009, ADR-0012, ADR-0023]
updates: []
obsoletes: []
---

# ADR-0024: Give each external module its own schema

## Context

PostgreSQL is the panel's only database, as [ADR-0023](0023-postgresql-replaces-mysql-and-mariadb.md) decides. Its
schemas give a set of tables a namespace of its own inside one database, and a schema can be created or dropped, with
everything in it, inside a transaction.

[Bundled modules](../GLOSSARY.md#bundled-module) ship inside the binary, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), and are written, reviewed and upgraded
together. [External modules](../GLOSSARY.md#external-module) are installed, updated and removed one at a time, through
Composer per [ADR-0012](0012-modules-are-installed-with-composer-used-as-a-library.md), by authors who do not coordinate
with each other or with the panel. Two external modules, or an external module and the panel's own, can choose the same
table name, and removing an external module means finding everything it created.

## Decision

We will give each [external module](../GLOSSARY.md#external-module) a PostgreSQL schema of its own, named from its
[module identifier](../GLOSSARY.md#module-identifier), and create its tables, and the objects that belong to them, in
that schema. [Bundled modules](../GLOSSARY.md#bundled-module) will share the default schema. An external module's schema
will be created inside the transaction that first migrates the [module](../GLOSSARY.md#module).

## Alternatives

**A schema for every module, bundled modules included.** Each [bundled module](../GLOSSARY.md#bundled-module) would have
a schema of its own as well. We rejected this because bundled modules ship, upgrade and are reviewed together as one
binary, so separating them from each other buys nothing: a schema exists to separate what is installed separately.

## Consequences

Easier:

- The names of an [external module](../GLOSSARY.md#external-module)'s own tables will not conflict with another
  [module](../GLOSSARY.md#module)'s, or with those of the [bundled modules](../GLOSSARY.md#bundled-module).
- Removing an external module's own tables will be a matter of dropping its schema. Anything it changed outside its
  schema, such as a column added to another module's table, will have to be reversed separately.
- The bundled modules' tables will keep unqualified names in the default schema.

Harder:

- Bundled modules will have to avoid conflicting table names with each other by convention, checked in review.
- Nothing in the database will record which bundled module owns which table in the default schema.
- A reference from one external module to another's table will have to name that module's schema.

Constrained:

- An external module's schema name will be derived from its [module identifier](../GLOSSARY.md#module-identifier), so
  the identifier will have to make a usable schema name: not the default schema's, not one PostgreSQL reserves, such as
  a name beginning `pg_`, and no longer than its 63-byte limit on names.
