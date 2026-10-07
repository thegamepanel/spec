---
id: ADR-0025
title: Run migrations only from commands, under maintenance
status: proposed
created: 2026-10-06
decided:
depends: [ADR-0009, ADR-0023]
updates: []
obsoletes: []
---

# ADR-0025: Run migrations only from commands, under maintenance

## Context

A migration changes the schema, so whatever runs one needs credentials able to change it.

The panel serves every request from long-running worker processes that boot once, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). New code reaches them only when they
restart, so a migration applied while they serve meets code written for the schema as it was before.

PostgreSQL runs schema changes inside transactions, as [ADR-0023](0023-postgresql-replaces-mysql-and-mariadb.md)
records, but some statements cannot run inside one, such as creating an index concurrently, which lets a table keep
taking writes while the index is built, or creating a database.

## Decision

We will run migrations only from commands, never from the process that serves requests, so that only those commands hold
credentials able to change the schema. Every command that runs migrations will put the panel into
[maintenance mode](../GLOSSARY.md#maintenance-mode), in which it refuses requests, run the migrations, restart the
workers so that they load the new code, and then leave maintenance mode. A new binary will not serve while its
migrations are pending. Because nothing is served while migrations run, a statement that cannot run inside a transaction
will not be allowed in a migration.

## Alternatives

**Running migrations when the panel starts.** The panel would run any pending migrations itself as it boots, before
serving. We rejected this because the process that serves every request would then hold credentials able to change the
schema.

**Migrating while the panel serves.** Migrations would run with the panel up, and each would have to suit both the code
before it and the code after it, such as by adding a column in one release and removing the old one in a later release.
We rejected this because it asks every [module](../GLOSSARY.md#module) author for a discipline the panel cannot check,
while the worker model already interrupts service to load new code.

## Consequences

Easier:

- The process serving requests will never hold credentials able to change the schema.
- No request will meet code and a schema that do not match.
- Every migration will run inside a transaction, with no statement excepted.
- A migration's author will not need to keep a schema working for two versions of their code.

Harder:

- Every change that runs migrations will interrupt service while it runs.
- Installing or upgrading will take a command, not only a replaced binary.

Constrained:

- [Maintenance mode](../GLOSSARY.md#maintenance-mode) will have to exist before migrations can run, and its design is
  its own.
- A statement that cannot run inside a transaction, such as creating an index concurrently, will not be possible in a
  migration.
