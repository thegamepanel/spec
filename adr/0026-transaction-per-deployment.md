---
id: ADR-0026
title: Migrate each deployment in one transaction
status: proposed
created: 2026-10-06
decided:
depends: [ADR-0009, ADR-0012, ADR-0023, ADR-0025]
updates: []
obsoletes: []
---

# ADR-0026: Migrate each deployment in one transaction

## Context

Code reaches an installation in two units. A new binary brings every [bundled module](../GLOSSARY.md#bundled-module) at
once, per [ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), and cannot be split: going back
means going back to the previous binary. An [external module](../GLOSSARY.md#external-module) is installed, updated or
removed on its own, through Composer, per [ADR-0012](0012-modules-are-installed-with-composer-used-as-a-library.md).

Each unit can bring migrations for the code it changes. PostgreSQL runs schema changes inside transactions, as
[ADR-0023](0023-postgresql-replaces-mysql-and-mariadb.md) records, and no migration may use a statement that cannot run
inside one, as [ADR-0025](0025-migrations-run-from-commands.md) decides, so a set of migrations can apply entirely or
not at all.

## Decision

We will run the migrations of one [deployment](../GLOSSARY.md#deployment) in one transaction: every
[bundled module](../GLOSSARY.md#bundled-module)'s pending migrations when a binary is replaced, and one
[external module](../GLOSSARY.md#external-module)'s when it is installed, updated or [purged](../GLOSSARY.md#purge). The
record of which migrations have run will be written inside the same transaction. A deployment whose migrations fail will
leave the schema, and that record, as they were before it.

## Alternatives

**One transaction for each migration.** Each migration would commit on its own. We rejected this because a
[deployment](../GLOSSARY.md#deployment) failing part way would leave a schema that matches neither the code before it
nor the code after it.

**One transaction for each module.** Each [module](../GLOSSARY.md#module)'s pending migrations would commit on their
own. We rejected this because a binary brings every [bundled module](../GLOSSARY.md#bundled-module) at once, so a
failure part way would leave the earlier bundled modules on the new schema, while the previous binary, the only way
back, matches the old one.

**One transaction for every pending migration.** Bundled and external migrations would commit together. We rejected this
because one [external module](../GLOSSARY.md#external-module)'s broken migration would undo the bundled modules' upgrade
with it, coupling modules that are installed separately.

## Consequences

Easier:

- A failed [deployment](../GLOSSARY.md#deployment) will leave the schema matching the code it replaced, so putting that
  code back restores a working installation.
- The record of which migrations have run will never disagree with the schema.
- An [external module](../GLOSSARY.md#external-module)'s broken migration will never block or undo an upgrade of the
  [bundled modules](../GLOSSARY.md#bundled-module).

Harder:

- A binary's migrations will hold their locks until every bundled module's migrations have finished.
- Whatever runs migrations will have to know which deployment it is running them for.

Constrained:

- A migration will not be able to commit part of its work before the rest of its deployment.
- A value added to an enumerated type will not be usable until its deployment commits, so a later migration in the same
  deployment will not be able to use it.
