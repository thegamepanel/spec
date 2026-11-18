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

PostgreSQL is the panel's only database, as [ADR-0023](0023-postgresql-replaces-mysql.md) decides. A
PostgreSQL schema is a namespace for tables inside one database, and a schema can be created or dropped, with everything
in it, inside a transaction.

[Bundled module](../GLOSSARY.md#bundled-module)s ship inside the binary, per
[ADR-0009](0009-frankenphp-worker.md), and are written, reviewed and upgraded
together. [External module](../GLOSSARY.md#external-module)s are installed, updated and removed one at a time through
Composer, per [ADR-0012](0012-composer-installs.md), by authors who do not
coordinate with each other or with the panel. Two external modules, or an external module and the panel itself, can pick
the same table name, and removing an external module means finding everything it created.

## Decision

I will give each [external module](../GLOSSARY.md#external-module) a PostgreSQL schema of its own, named from its
[module identifier](../GLOSSARY.md#module-identifier), and create its tables, and the objects that belong to them, in
that schema. [Bundled module](../GLOSSARY.md#bundled-module)s will share the default schema. An external module's schema
will be created inside the transaction that first migrates the [module](../GLOSSARY.md#module).

## Alternatives

**A schema for every module, bundled modules included.** Each [bundled module](../GLOSSARY.md#bundled-module) would get
a schema of its own as well. I rejected this because bundled modules ship, upgrade and are reviewed together as one
binary, so separating them from each other buys nothing: a schema exists to separate what is installed separately.

## Consequences

Easier:

- An [external module](../GLOSSARY.md#external-module)'s table names will not conflict with another
  [module](../GLOSSARY.md#module)'s, or with the [bundled module](../GLOSSARY.md#bundled-module)s'.
- Removing an external module's own tables will mean dropping its schema. Anything it changed outside that schema, such
  as a column it added to another module's table, will have to be reversed separately.
- The bundled modules' tables will keep unqualified names in the default schema.

Harder:

- Bundled modules will have to avoid table names that conflict with each other's by convention, checked in review.
- Nothing in the database will record which bundled module owns which table in the default schema.
- A reference from one external module to another's table will have to name that module's schema.

Constrained:

- An external module's schema name will come from its [module identifier](../GLOSSARY.md#module-identifier), so the
  identifier will have to make a usable schema name: not the default schema's name, not one PostgreSQL reserves, such as
  a name beginning `pg_`, and no longer than PostgreSQL's 63-byte limit on names.
