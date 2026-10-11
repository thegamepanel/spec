---
id: RFC-0017
title: Migration runner
status: proposed
created: 2026-10-11
decided:
depends: [ADR-0009, ADR-0012, ADR-0013, ADR-0023, ADR-0024, ADR-0025, ADR-0026, ADR-0027, ADR-0028, RFC-0001, RFC-0003, RFC-0012, RFC-0015]
updates: []
obsoletes: []
---

# RFC-0017: Migration runner

## Abstract

We will run each [module](../GLOSSARY.md#module)'s migrations in [module dependency](../GLOSSARY.md#module-dependency)
order, one transaction for each [deployment](../GLOSSARY.md#deployment), from management commands that hold a lock while
they run. A [ledger](../GLOSSARY.md#ledger) will record which migrations have run, a migration class will be able to
hold several migrations so that old ones can be squashed, and an [external module](../GLOSSARY.md#external-module) will
be removable with its schema and downgradable through its [migration map](../GLOSSARY.md#migration-map).

## Motivation

Nothing runs migrations. The `Migration` and `ReversibleMigration` contracts exist, but `Migrator` is an empty class,
nothing records which migrations have run, and nothing reverses one.

[Modules](../GLOSSARY.md#module) need migrations before they can store anything, and an
[external module](../GLOSSARY.md#external-module)'s migrations are written by authors who do not coordinate with each
other. Each module's migrations have to run in an order that lets one module's tables refer to another's, a failure has
to leave the schema matching the code, and removing or downgrading a module has to leave a schema the remaining code
recognises.

## Proposal

### Migrations and migration classes

A migration will be the code that applies, and where it can, reverses, one
[migration identity](../GLOSSARY.md#migration-identity). A migration class will implement `Migration`, and
`ReversibleMigration` where its migrations can be reversed:

```php
interface Migration
{
    public function up(Migrator $migrator): void;
}

interface ReversibleMigration extends Migration
{
    public function down(Migrator $migrator): void;
}
```

A migration class will declare the identities of its migrations with the `MigrationId` attribute, which takes one or
more:

```php
#[MigrationId('create_invoices')]
final class CreateInvoicesTable implements ReversibleMigration
{
    public function up(Migrator $migrator): void { /* ... */ }

    public function down(Migrator $migrator): void { /* ... */ }
}
```

An identity will stay the same across releases, so renaming or moving the class will not change it. The
[ledger](../GLOSSARY.md#ledger) will know a migration by its [module identifier](../GLOSSARY.md#module-identifier) and
its identity together, such as `billing` and `create_invoices`. Once a version of a [module](../GLOSSARY.md#module) has
been released, none of its identities will be removed from what the module collects, though one migration class can take
over several, as described below.

`Migrator` will give a migration what it needs from its run:

| Method | Effect |
|---|---|
| `connection(): Connection` | Returns the run's [connection](../GLOSSARY.md#connection), for executing statements, such as a compiled schema [node](../GLOSSARY.md#node) from [RFC-0012](0012-postgresql-database-layer.md), and for reading and changing data. |
| `module(): string` | Returns the identifier of the module the migration belongs to. |
| `schema(): string` | Returns the name of the schema the module's tables live in. |
| `as(string $identity, Closure $migration): void` | Registers the code of one migration, in a migration class holding several. |

The run will open the outermost transaction, and a migration's own transactions will only ever create and release
savepoints within it, as [RFC-0012](0012-postgresql-database-layer.md) nests them. A node advertising that it cannot run
inside a transaction will be refused before it runs, as [ADR-0025](../adr/0025-migrations-run-from-commands.md)
requires, through the compiler's way of refusing a node from RFC-0012, which the run will hold in place on its
connection, so a service executing such a node through that connection is refused too.

A statement given as a string cannot be inspected. PostgreSQL will refuse one that cannot run inside a transaction, such
as creating an index concurrently, as a failed query. It will accept transaction control, such as `COMMIT` or
`ROLLBACK`, which would end the run's transaction or break its savepoints, and a session setting, such as
`search_path`, which would carry into later migrations, and neither will be guarded. After each migration, the runner
will ask the driver whether the run's transaction is still open, and if it is not, will stop with
`InvalidMigrationException` naming the migration. What was committed by then will stay committed.

A migration class will be constructed by the dependency injection container from
[RFC-0001](0001-dependency-injection-container.md), and can take anything the container can resolve, with nothing
restricted. The run will use the container's [primary connection](../GLOSSARY.md#primary-connection), so a service using
that connection runs its queries inside the run's transaction. Which [bindings](../GLOSSARY.md#binding) exist will
depend on what was registered when the process running the migration started:

- the [engine](../GLOSSARY.md#engine)'s own bindings will always exist;
- a module that was enabled and loaded will have its bindings, but a module skipped for a disabled dependency, or for
  having no manifest, will not;
- a module that is disabled will have none, so a migration of a disabled
  [bundled module](../GLOSSARY.md#bundled-module) that needs its own module's services will fail, and with it the
  binary's [deployment](../GLOSSARY.md#deployment);
- a module being installed for the first time is not yet enabled, and will have none either;
- no cycle from [RFC-0007](0007-binding-lifetimes.md) will be open, so nothing bound for the length of a cycle can be
  resolved.

A migration needing something that does not exist will fail, its error wrapped in `MigrationException`, and its
deployment will roll back.

### Squashed migrations

A migration class will be able to hold several migrations, so that a [module](../GLOSSARY.md#module) can replace many
old migration classes with one:

```php
#[MigrationId(
    'create_invoices',
    'add_invoice_currency',
    'add_invoice_notes',
)]
final class Version1Migrations implements ReversibleMigration
{
    public function up(Migrator $migrator): void
    {
        $migrator->as('create_invoices', fn () => /* ... */);
        $migrator->as('add_invoice_currency', fn () => /* ... */);
        $migrator->as('add_invoice_notes', fn () => /* ... */);
    }

    public function down(Migrator $migrator): void
    {
        $migrator->as('create_invoices', fn () => /* ... */);
        $migrator->as('add_invoice_currency', fn () => /* ... */);
        $migrator->as('add_invoice_notes', fn () => /* ... */);
    }
}
```

`as()` will register a migration's code rather than run it. A migration class holding one migration will do its work
directly in `up()` and `down()`. A migration class holding several will register, in `up()`, exactly the identities it
declares, and in `down()`, none it does not declare. An identity with no code registered in `down()` cannot be reversed.
The migrations a class holds will have to be ones that ran one after another in the module's order, so that holding them
together does not change the order they apply in.

### Collecting migrations

A [module](../GLOSSARY.md#module) will contribute its migration classes through
[collection](../GLOSSARY.md#collection), from [RFC-0015](0015-module-system.md), with a `MigrationCollector`:

```php
#[Collect]
public function migrations(MigrationCollector $migrations): void
{
    $migrations->add(Version1Migrations::class);
    $migrations->add(BackfillInvoiceTotals::class);
}
```

| Method | Effect |
|---|---|
| `add(string $class): void` | Adds a migration class. Migrations will apply in the order their classes are added, and a class holding several in the order it declares them. |

Migrations will be collected from a module whether or not it is enabled, which the module system in
[RFC-0015](0015-module-system.md) will have to allow, since its collection walks only enabled modules. The build will
compile every [bundled module](../GLOSSARY.md#bundled-module)'s migrations, in order, into the binary. Where modules are
not compiled, such as when RFC-0015 reflects them at boot for debugging, they will be collected from the code as it is.

### Order

Migrations will apply [module](../GLOSSARY.md#module) by module, in the order the module system provides from each
module's declared [module dependencies](../GLOSSARY.md#module-dependency), and within a module in the order they are
collected, as [ADR-0028](../adr/0028-module-dependency-order.md) decides. Declaring, validating and ordering module
dependencies will belong to the module system, which [RFC-0015](0015-module-system.md) will have to take on.

A purge will reverse a module's migrations in the reverse of the order they applied in: by
[deployment](../GLOSSARY.md#deployment) number from the highest, and within a deployment in the reverse of the order
they are collected. A downgrade will reverse them in the order its migration map gives instead, as described below.
Either way, each migration class will have its `down()` called at its turn in the order: a class holding one migration
reverses it there, and a class holding several has its `down()` called once, at the first of its migrations to come up,
with the code it registers run at each migration's own place in the order.

### Running

Migrations will run only from management commands, under [maintenance mode](../GLOSSARY.md#maintenance-mode), as
[ADR-0025](../adr/0025-migrations-run-from-commands.md) decides, and each [deployment](../GLOSSARY.md#deployment) will
run in one transaction, as [ADR-0026](../adr/0026-transaction-per-deployment.md) decides. A management command is one of
the commands that maintain an installation, such as running migrations or installing a module, rather than the command
that serves the panel. Which commands exist will belong to the command line's design.

Every management command will hold the runner's lock from before it enters maintenance mode until after it leaves. The
lock will be a PostgreSQL advisory lock held for the session, in the runner's own namespace, on a
[connection](../GLOSSARY.md#connection) the command opens for nothing else. A command that finds the lock held will fail
at once, saying another management command is running. A process a management command starts to do part of its work
will run under that command's lock, and will neither take the lock nor share the connection holding it. The lock's
connection will not be persistent, will not pass through a pooler, and will have no idle session timeout, which would
end the session and release the lock. PostgreSQL will release the lock when the command's process ends, or once it
notices that the connection has gone. [RFC-0012](0012-postgresql-database-layer.md) offers locks held for a transaction,
and will have to offer this one as well.

Within the lock, a deployment will run in one transaction:

1. **Ledger.** The [ledger](../GLOSSARY.md#ledger) will be created if it does not exist.
2. **Number.** The deployment's number will be one more than the highest in the ledger, or one for an empty ledger.
3. **Pending.** The identities the deployment collects will be compared with the ledger, and those not recorded will be
   pending.
4. **Schema.** For an [external module](../GLOSSARY.md#external-module), its schema will be created if it does not
   exist, as [ADR-0024](../adr/0024-external-module-schemas.md) decides.
5. **Migrate.** Each migration class with a pending identity will be constructed and its `up()` called, and its pending
   migrations applied in order.
6. **Record.** A ledger row will be written for each identity as its migration is applied.
7. **Commit.** The transaction will commit once every pending migration has been applied.

A failure will roll the whole deployment back, and nothing any of its migrations did to the schema or its data will
remain. The runner's own exceptions will be thrown as they are, and anything else thrown while a migration is applied
or reversed will be wrapped in `MigrationException`, with the module, the identity and the original error. The one
exception is a purge, described below.

Every [bundled module](../GLOSSARY.md#bundled-module)'s migrations will run when a new binary is deployed, whether or
not the module is enabled. The command that serves the panel will refuse to start while any migration the binary
collects is pending. The runner will provide the check it makes, comparing the binary's compiled migrations with the
ledger once when it starts, and a missing ledger will mean every migration is pending. How the refusal is reported will
belong to the command line's design.

An external module's migrations will run as part of installing or updating it, whether or not it is enabled:

1. Composer will resolve, download and install the version.
2. A second process, which boots with the module's new code, will check the ledger. If the ledger records an identity
   for the module that the version does not collect, the database holds migrations from a newer version, and the second
   process will refuse with a `DowngradeException`, since going back to an older version is a downgrade, described
   below. Otherwise it will run the module's deployment.
3. The first process will keep the lock and maintenance mode while the second runs, and will then read the ledger to
   learn whether the deployment committed, rather than relying on how the second process ended. If it did not, the
   first process will restore the version the module replaced, or remove the module if it was not installed before.

The second process is needed because the first loaded the module's previous code when it started, and could neither see
the new version's migrations nor a module installed after it started. How the second process is started will belong to
the command line's design.

### The ledger

The [ledger](../GLOSSARY.md#ledger) will be the `migrations` table in the default schema, which the runner creates
itself rather than through a migration, and always names with its schema:

| Column | Holds |
|---|---|
| `module` | The [module identifier](../GLOSSARY.md#module-identifier). |
| `migration` | The [migration identity](../GLOSSARY.md#migration-identity). |
| `deployment` | The number of the [deployment](../GLOSSARY.md#deployment) that applied it, as an `integer`, shared by every row that deployment wrote. |
| `applied_at` | When it was applied, as `timestamptz`, from `clock_timestamp()`. |

`module` and `migration` together will be its primary key. Among the rows present, a higher deployment number will be a
later deployment. The name will be unavailable to [bundled modules](../GLOSSARY.md#bundled-module), which share the
default schema, and the panel's own credentials will need to read the table for the check made when it starts.

### Uninstalling and purging

Uninstalling an [external module](../GLOSSARY.md#external-module) will remove its code and leave its schema and its
[ledger](../GLOSSARY.md#ledger) rows in place. Installing it again will find them, and only migrations it has not run
before will run.

[Purging](../GLOSSARY.md#purge) a [module](../GLOSSARY.md#module) will uninstall it and remove what it created, while
its code is still installed. It will be refused while another installed module depends on it, enabled or not, which the
module system will enforce. It will also be refused while a module that was uninstalled without being purged keeps
anything in its schema that depends on the purged module's schema. Such a module is one the ledger records but the
module system does not know, its schema named from its identifier as ADR-0024 decides, and the default schema is never
counted as one. An uninstalled module with no ledger rows cannot be seen this way.

The purge will run as one [deployment](../GLOSSARY.md#deployment):

1. Each migration the module has applied that can be reversed will be reversed, in the reverse of the order they
   applied in, each inside a savepoint. If a reversal fails because something outside the module's schema depends on
   what it drops, the runner will roll back to the savepoint, find what depends on it from PostgreSQL's record of
   dependencies, and refuse the purge, naming each.
2. Anything outside the module's schema still depending on the schema or on something inside it, such as a foreign key
   on another module's table or an extension created in the schema, will be found the same way, and the purge refused,
   naming each.
3. The module's schema will be dropped, with everything in it.
4. The module's ledger rows will be deleted.
5. Composer will remove the module's code.
6. The transaction will commit if Composer succeeds, and roll back if it does not, leaving the module installed.

A refused purge will throw `PurgeException`, rather than having a failed reversal wrapped in `MigrationException`. The
purge's connection will have no idle transaction timeout and no transaction timeout, overriding any default the server
or its role sets, so the transaction survives Composer removing the code.

A purge can be forced past every refusal except an installed module depending on it. Forced, what depends on the
module's schema in an uninstalled module's schema will be dropped first, a reversal that fails on something depending
on it will have that dropped and be retried, and what remains in the second step will be dropped with the schema. Each
thing dropped will be reported.

A migration that cannot be reversed will be taken to have changed nothing outside its module's schema. PostgreSQL's
record of dependencies does not cover everything: a change that depends on nothing in the schema, such as a row added to
another module's table, and a function whose body refers to the schema, such as one written in PL/pgSQL, cannot be
detected, and will be left behind. A module that is already uninstalled will be purged by installing it again first.

### Downgrading an external module

An [external module](../GLOSSARY.md#external-module) will be downgraded through its
[migration map](../GLOSSARY.md#migration-map), as [ADR-0027](../adr/0027-module-downgrades.md) decides. The map will
list, for each version, the identities that version introduced, in the order they were added:

```json
{
    "1.0": ["create_invoices"],
    "1.2": ["add_invoice_currency"],
    "1.3": ["add_invoice_notes"],
    "current": ["backfill_invoice_totals"]
}
```

`current` will mean the version the map ships with. The [module](../GLOSSARY.md#module)'s author will rename it to that
version when work on the next version begins, and start a new `current`, with tooling to help. The runner will compare
versions as versions, through `composer/semver`, rather than as text. A map listing one identity under two versions will
be invalid.

Downgrading `billing` from 1.4 to 1.3 will run as one [deployment](../GLOSSARY.md#deployment), in the process that has
1.4 installed:

1. Composer will resolve and download 1.3 before the transaction opens.
2. The installed version's map will be read, and the identities it records as introduced after 1.3 found. 1.3's own
   map, from the downloaded package, will be read too, and any identity it lists will be kept rather than reversed, so a
   placeholder renamed too late cannot cause a migration 1.3 needs to be reversed.
3. Each identity left will be reversed, the newest version first and within a version the last added first, using 1.4's
   code, and its [ledger](../GLOSSARY.md#ledger) row deleted.
4. Composer will install 1.3.
5. The transaction will commit if Composer succeeds. If Composer fails, or the transaction does not commit, the
   transaction will roll back and 1.4's code will be restored.

The downgrade's [connection](../GLOSSARY.md#connection) will have no idle transaction timeout and no transaction
timeout, overriding any default the server or its role sets, so the transaction survives Composer's install. The
downgrade will be refused if either version has no map, if the installed version's map is invalid or leaves out an
identity it collects, or if an identity it would reverse cannot be reversed.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `MigrationException` | `DatabaseException` | Anything other than the runner's own exceptions is thrown while a migration is applied or reversed, outside a purge. It carries the [module](../GLOSSARY.md#module), the identity and the original error. |
| `InvalidMigrationException` | `LogicException` | A collected class does not implement `Migration`, has no `MigrationId`, or repeats an identity already collected for its module; a migration class holding several registers in `up()` other than exactly the identities it declares, or in `down()` one it does not declare; a migration executes a [node](../GLOSSARY.md#node) that cannot run inside a transaction; or a migration leaves the run's transaction closed. |
| `DowngradeException` | `RuntimeException` | A downgrade is refused, because either version's map is missing, the installed version's map is invalid or incomplete, or an identity it would reverse cannot be reversed; or installing or updating a module is refused, because the [ledger](../GLOSSARY.md#ledger) records identities the version does not collect. It names the identities it concerns. |
| `PurgeException` | `RuntimeException` | A [purge](../GLOSSARY.md#purge) is refused because something outside the module's schema depends on it, found before reversing, by a reversal that failed, or after reversing. It names each dependent. |

Each will be final. `DatabaseException` is the database component's own, from [RFC-0003](0003-database-component.md).

### Out of scope

- **Declaring [module dependencies](../GLOSSARY.md#module-dependency)**, and validating and ordering them, which belong
  to the module system, and which [RFC-0015](0015-module-system.md) will have to take on.
- **[Maintenance mode](../GLOSSARY.md#maintenance-mode)**: entering and leaving it, refusing requests and restarting
  workers, which belong to its own design.
- **The commands**: which management commands exist, their names and arguments, how a command starts the second process
  that migrates an installed or updated module, and how a management command comes to hold credentials able to change
  the schema, all of which belong to the command line's design.
- **Generating the [migration map](../GLOSSARY.md#migration-map)**, which belongs to the tooling module authors release
  with.
- **Retiring identities**, so that a module could drop a released migration altogether. Squashing keeps every identity,
  so nothing needs it yet.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- the panel running as a single binary under a worker, in
  [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)
- installing external modules with Composer, in
  [ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md)
- the panel's own features being modules, in [ADR-0013](../adr/0013-core-features-are-modules.md)
- PostgreSQL as the only database, in [ADR-0023](../adr/0023-postgresql-replaces-mysql-and-mariadb.md)
- giving each external module its own schema, in [ADR-0024](../adr/0024-external-module-schemas.md)
- running migrations only from commands, under maintenance, in
  [ADR-0025](../adr/0025-migrations-run-from-commands.md)
- migrating each deployment in one transaction, in [ADR-0026](../adr/0026-transaction-per-deployment.md)
- downgrading external modules through a migration map, in [ADR-0027](../adr/0027-module-downgrades.md)
- running migrations in module dependency order, in [ADR-0028](../adr/0028-module-dependency-order.md)

**A directory of migration files.** A [module](../GLOSSARY.md#module) would declare a directory, and files named by the
time they were created would apply in name order. We rejected this because it adds a second way of discovering what a
module contributes, and leaves the order to the developer naming every file correctly. Tooling could name the files, but
changing an order would then mean renaming them, which is more complex than it needs to be and an unnecessary
requirement on the developer.

**The class name as identity.** We rejected this because renaming or moving a class would make the runner treat its
migration as new and apply it again.

**Identities listed in the collector.** A module would add each migration class under its identity, such as
`add('create_invoices', CreateInvoicesTable::class)`. We rejected this because the class should itself say which
[ledger](../GLOSSARY.md#ledger) rows its migrations are recorded under.

**One `MigrationId` for each identity.** A migration class holding several migrations would repeat the attribute. We
rejected this because one attribute holding them all, in order, keeps the whole set in one place, and requiring its
first identity makes an empty one impossible to write.

**One method registering both directions.** Each migration would be given its applying and reversing code in one call,
in a method replacing `up()` and `down()`. We rejected this because it changes the contracts that exist, where
registering in `up()` and `down()` keeps them.

**A `Migrator` method for executing statements.** We rejected this because the [connection](../GLOSSARY.md#connection)
already executes [nodes](../GLOSSARY.md#node) and strings.

**Migrating when a module is enabled.** We rejected this because a disabled module's code would not match the schema,
and enabling a module would then become a [deployment](../GLOSSARY.md#deployment), with
[maintenance mode](../GLOSSARY.md#maintenance-mode) and credentials able to change the schema, rather than the change of
configuration it is.

**Installing and migrating as separate commands.** We rejected this because the code and the schema it needs would then
change in two steps, and only the step that changed the code can put it back.

**Migrating an updated module in the process that installed it.** We rejected this because that process loaded the
module's previous code when it started, and would apply the previous version's migrations, or a mixture of both.

**Migration commands without modules registered.** A command running migrations would register no module, so that a
migration could only be given the [engine](../GLOSSARY.md#engine)'s own services. We rejected this because modules
could then add no commands and take no part in the command line.

**A lock held for each transaction.** We rejected this because a command does work outside its transaction, such as
running Composer and entering and leaving maintenance mode, and another command could start in between.

**Waiting for the lock.** A command finding the lock held would wait for it. We rejected this because two management
commands at once is almost always a mistake, and a command that waited would run against a panel changed beneath it.

**A lock on a file, or a row recording the lock.** We rejected a file because it only works when every instance shares
a filesystem, and a row because a crash would leave it behind.

**A ledger in each schema.** We rejected this because finding what has run would mean visiting every schema, where one
table answers it.

**An identifier generated for each deployment.** The ledger would identify a deployment with a
ULID, or number its rows by auto-increment. We rejected these because a deployment needs nothing more than a number, and
auto-increment numbers each row, where a deployment's rows share one.

**Dropping the schema when a module is uninstalled.** We rejected this because its data could not be recovered.

**Purging as a separate step after uninstalling.** We rejected this because the module's code, which reverses its
migrations, is gone once it is uninstalled.

**Checking for dependents only after reversing.** We rejected this because a dependent known in advance, such as a
module uninstalled without being purged, should refuse the purge before anything is reversed, not part-way through.

**Dropping whatever depends on the schema without asking.** We rejected this as the default because it would change
tables the module did not own without anyone knowing, and kept it as the forced [purge](../GLOSSARY.md#purge).

**Removing the modules that depend on one with it.** We rejected this because it would remove modules nobody asked to
remove.

**Allowing an older version over a newer schema.** We rejected this because its code would run against a schema it does
not know.

**Requiring a purge before installing an older version.** We rejected this because it throws away the data that
uninstalling keeps.

## Backwards compatibility

Nothing breaks. Nothing runs migrations today and no migration exists. The `Migration` and `ReversibleMigration`
contracts keep their signatures. `Migrator`, an empty readonly class, gains its methods and holds the migrations a class
registers, so it will no longer be readonly.

## Open questions

- **Downgrading the binary.** Whether a binary is downgraded by reversing a
  [migration map](../GLOSSARY.md#migration-map) compiled into it, as an
  [external module](../GLOSSARY.md#external-module) is, or by a mechanism of its own.
- **The search path.** Whether an external module's migrations run with its schema on the search path, or name it
  through `Migrator::schema()`.
- **Where the migration map lives**, in a file of its own or in the module's metadata.
- **An older binary over a newer schema.** The check made when the panel starts finds migrations that are pending, but
  not migrations the ledger records for a bundled module that the binary does not collect.

## Changelog
