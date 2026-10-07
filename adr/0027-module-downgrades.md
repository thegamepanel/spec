---
id: ADR-0027
title: Downgrade external modules through a migration map
status: proposed
created: 2026-10-07
decided:
depends: [ADR-0012, ADR-0026]
updates: []
obsoletes: []
---

# ADR-0027: Downgrade external modules through a migration map

## Context

An [external module](../GLOSSARY.md#external-module) is installed and updated through Composer, per
[ADR-0012](0012-modules-are-installed-with-composer-used-as-a-library.md), and its migrations run when it is installed
or updated. A newer version can bring migrations an older version does not know, such as a new table or column. Going
back to the older version without reversing them leaves a schema its code does not match.

Which migrations have run says nothing about which version introduced each. A [module](../GLOSSARY.md#module) installed
directly at 1.3 runs every one of its migrations at once, including those 1.2 already had.

## Decision

We will let an [external module](../GLOSSARY.md#external-module) be downgraded by reversing the migrations introduced
after the version being installed. Each released version of a [module](../GLOSSARY.md#module) will carry a
[migration map](../GLOSSARY.md#migration-map), generated when it is released, recording the version that introduced each
of its migrations: an existing migration keeps its version, a new one takes the version being released, and one not yet
released carries a placeholder. A downgrade will read the map of the version currently installed, reverse the migrations
it records as introduced after the target version, newest first, using that version's code, and then install the target,
as one [deployment](../GLOSSARY.md#deployment) under [ADR-0026](0026-transaction-per-deployment.md). A downgrade will be
refused if the installed version has no map, if any of its migrations is missing from the map, or if a migration it
would reverse cannot be reversed.

## Alternatives

**Refusing downgrades.** A [module](../GLOSSARY.md#module) could not go back a version once a newer version's migrations
had run. We rejected this because undoing one module's upgrade would then mean restoring the whole database.

**Recording the version that applied each migration.** The record of which migrations have run would note the module's
version when each ran, and a downgrade would reverse those applied by later versions. We rejected this because a module
installed directly at 1.3 records every migration under 1.3, so downgrading it to 1.2 would reverse migrations 1.2
already had, losing their data.

**Declaring the version on each migration.** Each migration would state the version that introduced it. We rejected
this because no migration should have to be written knowing the version it will ship in.

**Guessing about a migration missing from the map.** A migration the map does not record would be treated as newer than
any version, or as older than every version. We rejected this because the first reverses migrations the target version
needs, losing their data, and the second leaves a schema the target version does not know.

## Consequences

Easier:

- An operator will be able to take one [external module](../GLOSSARY.md#external-module) back a version without touching
  the rest of the database.
- A migration will not need to know the version it ships in.

Harder:

- An author who wants a [module](../GLOSSARY.md#module) to be downgradable across a migration will have to give that
  migration a way to be reversed.
- Every release will have to regenerate the [migration map](../GLOSSARY.md#migration-map), and a version released
  without one will not be downgradable.
- Release tooling to generate the map will have to exist.

Constrained:

- A downgrade will reverse migrations with the code of the version being replaced, so it will have to run before that
  code is removed.
- A migration's identity, once released, will have to stay the same, because the map records versions by identity.
