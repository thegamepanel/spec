---
id: RFC-0012
title: PostgreSQL database layer
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0003, ADR-0008, ADR-0016, ADR-0017]
updates: [RFC-0003]
obsoletes: []
---

# RFC-0012: PostgreSQL database layer

## Abstract

I will move the database component onto PostgreSQL and put a [compiler](../GLOSSARY.md#compiler) between its objects and
the SQL they become. Each query and schema object will be a [node](../GLOSSARY.md#node) describing what is wanted, and
the compiler registered for its class will turn it into SQL and bound values together. Connections, transactions, the
type model and schema statements will become PostgreSQL's, and the [engine](../GLOSSARY.md#engine) will gain the
PostgreSQL features its later subsystems are built on, each as a [primitive](../GLOSSARY.md#primitive).

## Motivation

The database component assumes MySQL, and every query and schema object produces its own SQL, as
[RFC-0003](0003-database-component.md) describes.
[ADR-0016](../adr/0016-postgresql-16.md) makes PostgreSQL the only supported database,
and [ADR-0017](../adr/0017-sql-compiler.md) moves SQL production into a
[compiler](../GLOSSARY.md#compiler).

Both changes reach the same classes, so they will arrive together: the dialect changes what SQL is produced, and the
compiler changes what produces it.

## Proposal

### SQL behind a compiler

Anything the [compiler](../GLOSSARY.md#compiler) understands will be a [node](../GLOSSARY.md#node). Queries, schema
statements, columns and indexes will all be nodes. An expression will narrow to what the word means, a fragment that
produces a value, such as raw SQL. A node compiler will turn one type of node into SQL, and is handed the compiler so
that it can compile the nodes it contains:

```php
interface Node {}
interface Expression extends Node {}

interface NodeCompiler
{
    public function compile(Node $node, Compiler $compiler): CompiledSql;
}
```

`Compiler::compile(Node $node): CompiledSql` will own no syntax of its own. It will look up the node compiler registered
for the node's class in a [catalogue](../GLOSSARY.md#catalogue), and pass itself along, so compiling recurses: the
compiler for a table will never need to know how a column renders. Compilers will be collected in a
[registry](../GLOSSARY.md#registry) and sealed into that catalogue, as
[ADR-0003](../adr/0003-sealed-catalogues.md) decides.

What a compiler returns will be [compiled SQL](../GLOSSARY.md#compiled-sql): a string of SQL together with the values
bound to it.

```php
final readonly class CompiledSql
{
    public string $sql;
    public array $bindings;

    public static function of(string $sql, mixed ...$bindings): self;
    public function append(string $sql, CompiledSql ...$fragments): self;
}
```

A fragment will carry its own bound values, and composing fragments will concatenate their SQL and their values in the
same order. There will be no way to produce a placeholder and its value separately.

The compiler will be the only place a complete statement exists before it becomes a string, so it will carry a hook
where a statement can be refused.

### Names and literals

Every name written into SQL will go through `Identifier`, a value object that quotes and escapes it:

- A name will be rendered in double quotes, with any double quote inside it doubled.
- Names will be quoted unconditionally, so they are case sensitive exactly as written. The
  [engine](../GLOSSARY.md#engine)'s own names are lower snake case, so the difference in case folding never arises for
  them, but a table declared with any other case has to be quoted by hand from then on.
- Qualified names and wildcards will be handled, which is what will let a name be qualified with a schema later without
  any compiler changing.
- A name containing a null byte will be rejected, and so will a name longer than 63 bytes. PostgreSQL truncates longer
  names silently, so two long generated index names could otherwise collapse into one.

A matching quoter will handle comments and any other literal that cannot be a bound value.

### Connections

`ConnectionFactory` will produce connections on the PostgreSQL driver, which gives the [engine](../GLOSSARY.md#engine)
notifications, bulk copying and large objects without a second extension. A [connection](../GLOSSARY.md#connection) will
hold its options, run its transactions and provide each [primitive](../GLOSSARY.md#primitive).

- **The connection string** will gain the SSL mode, a connection timeout and an application name. The application name
  is what will make it possible to tell which part of the panel holds a lock.
- **A socket setting** will name the directory PostgreSQL's socket lives in, rather than the socket file.
- **Options** will be split into overridable defaults and a forced set, so that nothing reachable from configuration can
  turn off the error mode the connection depends on. Emulated prepares will be dropped, since the driver always
  prepares natively.
- **Statement, lock and idle-in-transaction timeouts** will be connection configuration, applied when the connection is
  made.
- **Persistent connections** will be applied, and the session reset on connect, so that a reused connection inherits no
  prepared statements, temporary tables, search path or session locks. They are incompatible with a pooler running in
  transaction mode.
- **A password** may be empty, since socket peer authentication uses none.

The last inserted identifier will be removed from the write result rather than fixed. Returning rows from the statement
itself will replace it.

The driver returns native integers, floats and booleans where MySQL's returned strings, so the typed accessors on a
[row](../GLOSSARY.md#row) from [RFC-0003](0003-database-component.md) will carry weight they have not carried before.
What the driver returns for each type will be pinned by tests rather than assumed.

### Transactions nest through savepoints

This is the sharpest change in behaviour in the design. When any statement fails, PostgreSQL aborts the whole
transaction, and every statement after it fails until the transaction is rolled back. A nested transaction will
therefore issue a savepoint, a failure inside it will roll back to that savepoint, and only the outermost transaction
will commit. Without this, code that catches a failed query and carries on within the same unit of work would be
silently broken.

### Queries

Each query object will keep its builder methods and lose its rendering. Selects, inserts, updates, deletes and raw
expressions will become [node](../GLOSSARY.md#node)s, the clauses they share will gain compilers of their own, and the
traits that compose those clauses will keep only their fluent methods.

`Connection::query()`, `execute()` and `stream()` will accept a node or a string, and compile a node through the
[compiler](../GLOSSARY.md#compiler) the connection is given. Being executable will then mean that a compiler is
registered for the node, rather than that the object has a method returning SQL.

Moving the query nodes behind the compiler will change no SQL except that names are quoted. Every change of dialect will
be made separately from that move.

#### Writes

- **Inserting while ignoring conflicts** will be `ON CONFLICT DO NOTHING`, in place of `INSERT IGNORE`.
- **Replacing** will be removed with nothing in its place. It deletes the conflicting row and inserts a new one, which
  fires delete triggers and loses every column not mentioned. An upsert is what is wanted instead.
- **An upsert** will be `ON CONFLICT` with an explicit target, either a list of columns or a named constraint, followed
  by doing nothing or by an update with an optional condition. The excluded row will be available to the update, since
  referring to the value that failed to insert is the point of the construct.
- **Returning rows** will be available on inserts, updates and deletes. Asking for them will change the type of the
  [node](../GLOSSARY.md#node), so that the statement runs as a query and yields rows, and the write result stays a
  simple value. This will be the answer to the removed last inserted identifier, and unlike it, it will work for an
  insert of several rows.
- **Writes across tables** will be possible without a correlated subquery: an update may take a `FROM`, and a delete a
  `USING`.
- **Ordering and limiting** will be removed from updates and deletes, since PostgreSQL supports neither on a write. A
  subquery selecting the keys to act on will take their place.
- **An unguarded write**, meaning an update or delete compiled with no condition, will be refused unless the caller has
  said that is what they meant. This is possible only because the compiler sees the whole statement first.

#### Reads

- **Locking** will offer `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` and `FOR KEY SHARE`, each able to skip locked
  rows, refuse to wait, or name the tables it applies to. Skipping locked rows is how a job queue will claim work.
- **A select** will gain distinct on named expressions, common table expressions, including recursive and
  data-modifying ones, lateral joins, and window functions with partitions and frames.
- **Pattern matching** will gain case-insensitive matching and the regular expression operators.
- **JSON** will gain the access and containment operators over `jsonb`. Existence will be tested through the function
  forms rather than the operators, because the operators collide with the placeholder marker: a query containing one
  either fails to parse or has the operator taken as a parameter.
- **Arrays** will gain containment and overlap, and comparison against any or all of their elements.
- **Full-text search** will match a text search vector against a query parsed from what people actually type, with
  ranking available for ordering. It will replace MySQL's full-text expression, which will be removed.
- **An offset** without a limit will be valid, which it was not under MySQL.

### Types

| Today | Becomes |
|---|---|
| The four integer sizes | `smallint`, `integer`, `bigint` |
| Unsigned integers | Removed. Widen the type, or add a check constraint |
| `DECIMAL`, `FLOAT`, `DOUBLE` | `numeric`, `real`, `double precision` |
| The six character types | `char`, `varchar`, `text` |
| The six binary types | `bytea` |
| Enumerations | Text with a check constraint, or a native type by opting in |
| Sets and years | Removed |
| The four temporal types | `date`, `time`, `timestamptz` |
| JSON | `jsonb` |
| New | `uuid`, `interval`, the network address types, and a text search vector |

Timestamps will always carry a time zone. A naive timestamp in a panel that schedules restarts across time zones is a
bug waiting for the clocks to change.

Column classes will stay grouped by family, since that is what makes the shared templates work, but which types belong
to each family will change and new families will arrive. An array will be a modifier on any column rather than a family
of its own, because almost any type can be an array.

Columns will gain a collation, a check constraint, identity, stored generated values, and default expressions beside
default literals. They will lose column placement, unsigned, a character set per column, and virtual generated values,
which need a version above the panel's minimum.

An enumeration will be text with a check constraint by default, because changing the permitted values is then a swap of
one constraint for another inside a transaction. Opting into a native type will give something compact that can be
introspected, at the cost of a schema object with a lifecycle of its own.

### Schema statements

#### Literals, never bound values

PostgreSQL accepts no placeholders in a schema statement, so a default, a check expression and an enumeration's values
all have to be literal text. Every literal will go through the quoter, and a schema [node](../GLOSSARY.md#node) will
compile to a statement with no bound values by construction.

#### One node, several statements

A table's definition will no longer compile to one string. Only primary keys and uniqueness can be declared inline, so
every other index is a statement of its own, and so is a comment on a table or a column.

Compiling a schema [node](../GLOSSARY.md#node) will therefore produce a [statement list](../GLOSSARY.md#statement-list),
executed in order inside one transaction. PostgreSQL runs schema changes inside transactions, so a migration that fails
at its seventh statement rolls back the first six. The exceptions are creating an index concurrently and creating a
database, neither of which can run inside a transaction, so a node will say so in advance rather than failing when it is
executed.

#### Tables, indexes and other objects

- **Tables** will lose the storage engine and the table-level character set and collation, since encoding belongs to the
  database. They will gain unlogged tables, for data that is genuinely disposable, and creation only when the table is
  absent.
- **Altering a table** will take several actions in one statement, and a change of type will carry an explicit
  conversion where the cast is not implicit.
- **Dropping and truncating** will both gain cascading and restricting, and truncating will be able to restart identity
  columns.
- **An index** may choose its method, and may be on an expression, partial, covering, unique with nulls not distinct, or
  use an operator class.
- **Constraints** will cover primary keys, uniqueness, foreign keys with their actions and deferability, checks, and
  exclusion constraints. An exclusion constraint has no MySQL equivalent, and lets the database guarantee that no two
  rows overlap.
- **Schemas, enumerated types and extensions** will be able to be created and dropped. The native enumeration path needs
  this, and it is what will later make it possible to qualify names with a schema for each
  [module](../GLOSSARY.md#module).

### Streaming through a cursor

`stream()` will use a real server-side cursor: the query will be declared as a cursor, its rows fetched forward in
batches, and the cursor closed. Memory will stay flat whatever the size of the result, where today the driver retrieves
every row before the first is handed back.

A cursor lives inside a transaction, so streaming will join the caller's transaction or open one for as long as the
cursor is open. That has two consequences. A long stream holds a transaction open, which holds back vacuuming. And a
slow consumer can be killed by the idle-in-transaction timeout that protects the panel from wedged connections.

However iteration ends, including a consumer breaking out early or a callback throwing, the cursor will be closed and
the transaction resolved. A leaked cursor holds its transaction, and a leaked transaction eventually holds everything.

### Primitives

Each [primitive](../GLOSSARY.md#primitive) will exist because a subsystem already designed needs it, and each will be a
thin wrapper with no policy of its own.

- **Notifications** will be sent through a function call, so the channel and the payload are bound values rather than
  interpolated text. Delivery is transactional, so a notification sent inside a transaction fires only when it commits.
  A listener will hold a [connection](../GLOSSARY.md#connection) of its own and block with a timeout rather than
  polling.
- **Advisory locks** will last for a transaction, so a lock cannot outlive its unit of work or leak when a process dies.
  Taking a lock outside a transaction will open one. Keys will be namespaced by subsystem, so two subsystems cannot
  collide, though two names within one namespace still can.
- **Bulk copying** will read and write rows in bulk, in a format that handles quoting and embedded delimiters, an order
  of magnitude faster than inserting many rows. It is for moving rows, where streaming is for processing them.
- **Session variables** will be set for the length of a transaction, through a function so that the value is bound, and
  will carry the request identity that row-level policies read. Being scoped to the transaction is what stops one
  request's value leaking into the next on a reused connection.

### Proving statements against a server

Every construct the [compiler](../GLOSSARY.md#compiler) can produce will have a fixture, and one integration test will
prepare each fixture's SQL against a real PostgreSQL server. The test will then prove that the database accepts the
statement, rather than that the compiler produced what the test's author typed. The fixtures will be shared with the
unit tests, so a construct cannot be added with a string comparison alone.

### Out of scope

- **The migration runner.** It inherits the two statements that cannot run inside a transaction.
- **The subsystems the primitives exist for**: delivering events durably, the job queue, the scheduler, and any
  policy.
- **Reconnection and backoff for a listener**, which belong to whatever consumes notifications.
- **Declarative partitioning**, which has no consumer yet.
- **`MERGE`.** It is available at the minimum supported version, but it is the wrong tool for an upsert.

## Alternatives considered

This design rests on four decisions recorded separately:
[ADR-0003](../adr/0003-sealed-catalogues.md), which seals mutable registries into
immutable catalogues; [ADR-0008](../adr/0008-direct-pdo.md), which builds database access
directly on PDO; [ADR-0016](../adr/0016-postgresql-16.md), which makes PostgreSQL 16 and
above the only supported database; and
[ADR-0017](../adr/0017-sql-compiler.md), which produces SQL with a compiler.

**Escaping the JSON existence operators** by doubling the placeholder marker, rather than using the function forms. It
works on this driver. I rejected this because it puts a driver quirk into generated SQL that anyone reading a log has to
decode.

**Fixing the last inserted identifier** rather than removing it. I rejected this because asking the connection after
the statement cannot answer for an insert of several rows, and reports a stale value after an update or delete, where
returning rows from the statement that generated them answers both.

**Holding a cursor beyond its transaction.** It is possible. I rejected this because it materialises the whole result
when the transaction commits, which is the cost streaming exists to avoid.

## Backwards compatibility

- MySQL and MariaDB will no longer be supported, and everything written against their dialect will change: the upsert
  and replace constructs, full-text search, the type model, and ordering or limiting a write.
- A query or schema object will no longer produce its own SQL, so anything calling those methods will compile the
  [node](../GLOSSARY.md#node) instead, and a [connection](../GLOSSARY.md#connection) will need a
  [compiler](../GLOSSARY.md#compiler).
- Names will be quoted, so generated SQL will differ from today's even where the construct is unchanged.
- The last inserted identifier will be gone from the write result, replaced by returning rows.
- A socket setting will name a directory rather than a file.

## Open questions

### The shape of a compiled result carrying several statements

The column model and the table work agree it is needed and that the table work delivers it, but neither settles what it
looks like.

### How a caller opts out of the unguarded-write guard

### The builder methods for an upsert

The record describes them by behaviour rather than by name.

### Whether writes needing ordering get a helper

Or only a documented subquery pattern.

### How a module registers a node and its compiler

The compiler seam makes it possible but does not describe it.

### The namespace values for advisory locks

One per subsystem.

### The default batch size for a cursor

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.
- 2026-11-18: Removed splitting database roles and row-level policies from Out of scope, since neither belongs to
  this design.

## Sources

- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19, whose revisions only reformatted its list of cards: the
  scope, the architecture and what it buys, what changes across types, queries and connections, each primitive with
  the consumer that needs it, and the issues it absorbs.
- Issue [#52], Engine - Database - Compiler seam, 2026-08-19: the contracts, the compiled result, the identifier
  value object and its rules, the literal quoter, and the hook for refusing a statement.
- Issue [#53], Engine - Database - PostgreSQL connections and transactions, 2026-08-19: the driver, the connection
  string, socket semantics, the option split, session timeouts, savepoint-aware transactions, persistent connections
  with a session reset, empty passwords, and the fetch representations to pin.
- Issue [#54], Engine - Database - PostgreSQL test infrastructure and CI, 2026-08-19: the fixture registry and the
  harness that prepares every construct against a real server.
- Issue [#55], Engine - Database - Query compilers at parity, 2026-08-19: moving the query nodes behind the compiler
  with quoting as the only intended difference, the connection gaining a compiler, and what being executable then
  means.
- Issue [#56], Engine - Database - Query: PostgreSQL write features, 2026-08-19: the removals, the upsert, returning
  rows, multi-table writes, and the unguarded-write guard.
- Issue [#57], Engine - Database - Query: PostgreSQL read features, 2026-08-19: locking, the select constructs,
  pattern matching, the JSON operators and the placeholder collision, arrays, full-text search, and the bare offset.
- Issue [#58], Engine - Database - Schema: column and type model, 2026-08-19: the type catalogue, time zone aware
  timestamps, the class shape and arrays as a modifier, the modifiers gained and lost, DDL taking no bound values,
  comments being separate statements, and both enumeration paths.
- Issue [#59], Engine - Database - Schema: tables, indexes and DDL objects, 2026-08-19: the ordered statement list,
  the table, index, constraint and object changes, and the two statements that cannot run inside a transaction.
- Issue [#60], Engine - Database - Server-side cursor streaming, 2026-08-19: the cursor, its transaction, the
  vacuuming and timeout trade-offs, and cleanup on early termination.
- Issue [#61], Engine - Database - PostgreSQL primitives, 2026-08-19: each primitive's shape, why the function forms
  are used, transaction-scoped locks and their key derivation, bulk copying, and transaction-scoped session
  variables.

[#51]: https://github.com/thegamepanel/panel/issues/51
[#52]: https://github.com/thegamepanel/panel/issues/52
[#53]: https://github.com/thegamepanel/panel/issues/53
[#54]: https://github.com/thegamepanel/panel/issues/54
[#55]: https://github.com/thegamepanel/panel/issues/55
[#56]: https://github.com/thegamepanel/panel/issues/56
[#57]: https://github.com/thegamepanel/panel/issues/57
[#58]: https://github.com/thegamepanel/panel/issues/58
[#59]: https://github.com/thegamepanel/panel/issues/59
[#60]: https://github.com/thegamepanel/panel/issues/60
[#61]: https://github.com/thegamepanel/panel/issues/61
