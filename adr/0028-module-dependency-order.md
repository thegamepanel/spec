---
id: ADR-0028
title: Run migrations in module dependency order
status: proposed
created: 2026-10-07
decided:
depends: [ADR-0009, ADR-0012, ADR-0026]
updates: [ADR-0023]
obsoletes: []
---

# ADR-0028: Run migrations in module dependency order

## Context

[Modules](../GLOSSARY.md#module) keep tables of their own, and one module's tables can refer to another's, such as an
invoice referring to the server it bills. A table has to exist before anything can refer to it.

[ADR-0023](0023-postgresql-replaces-mysql-and-mariadb.md) records a migration design that runs in three stages, creating
tables, then the references between them, then changing data, with each stage run across every migration before the
next begins, so that modules can refer to each other's tables without their migrations being ordered.

[Bundled modules](../GLOSSARY.md#bundled-module) ship inside the binary, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), and are not Composer packages. Only
[external modules](../GLOSSARY.md#external-module) declare Composer requirements, per
[ADR-0012](0012-modules-are-installed-with-composer-used-as-a-library.md). Every bundled module's migrations run when a
binary is replaced, as [ADR-0026](0026-transaction-per-deployment.md) decides, whether or not the module is enabled.

The module system, in [RFC-0015](../rfc/0015-module-system.md), resolves no dependency between two modules and orders
them by none. A module is enabled by its configuration being present, so it can be enabled without any command.

## Decision

We will run migrations [module](../GLOSSARY.md#module) by module, in an order where every module comes after the modules
it depends on, and drop the three stages, so that each migration runs whole. A module that refers to another module's
tables will declare a [module dependency](../GLOSSARY.md#module-dependency) on it in its
[manifest](../GLOSSARY.md#manifest). The module system will provide the order, covering every installed module, enabled
or not, and will refuse a dependency that is missing or circular, a [bundled module](../GLOSSARY.md#bundled-module)
depending on an external one, and removing a module another installed module depends on. An enabled module whose
dependency is disabled will be skipped when the panel boots, with the reason logged, and the commands that enable and
disable modules will refuse to leave a module in that state.

## Alternatives

**Running each stage across every module.** Migrations would run in the three stages, with each stage run for every
[module](../GLOSSARY.md#module) before the next, so that modules could refer to each other's tables without being
ordered. We rejected this because ordering modules by their [module dependencies](../GLOSSARY.md#module-dependency)
already lets one module's tables refer to another's, and a reference to another module's table is a dependency that
should be declared.

**Deriving the order from Composer.** The order would come from each
[external module](../GLOSSARY.md#external-module)'s Composer requirements. We rejected this because
[bundled modules](../GLOSSARY.md#bundled-module) are not Composer packages, so a dependency on one could not be
expressed, and building an order from Composer's requirements is more complex than declaring it.

**The migrations ordering modules themselves.** Whatever runs migrations would build its own order from the
[manifests](../GLOSSARY.md#manifest). We rejected this because a dependency between modules is a fact about modules,
which enabling, disabling and removing them need as well.

## Consequences

Easier:

- A migration will create a table together with its references to another [module](../GLOSSARY.md#module)'s tables,
  because those tables will already exist.
- Enabling, disabling and removing modules will be able to use the same declared
  [module dependencies](../GLOSSARY.md#module-dependency).

Harder:

- A module's author will have to declare every module whose tables it refers to.
- The module system will have to validate dependencies and provide an order before any migration runs.
- Nothing will check that a migration's references match its declared dependencies, so an undeclared reference may work
  on one installation and fail on another, depending on where the order places the two modules.

Constrained:

- Two modules will not be able to refer to each other's tables.
- A [bundled module](../GLOSSARY.md#bundled-module) will not be able to depend on an external one.
- A module whose dependency is disabled will not run, even when it is enabled.
