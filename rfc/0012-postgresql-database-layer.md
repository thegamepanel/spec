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

We will move the database component to PostgreSQL and behind a [compiler](../GLOSSARY.md#compiler). Each query and
schema object will become a [node](../GLOSSARY.md#node) describing what is wanted, and the compiler registered for a
node's class will turn it into SQL and its bound values together. Connections will use the PostgreSQL driver,
transactions will nest through savepoints, the type model and DDL will become PostgreSQL's, and the
[engine](../GLOSSARY.md#engine) will gain the PostgreSQL features later subsystems are built on, each as a
[primitive](../GLOSSARY.md#primitive).

## Motivation

The database component assumes MySQL, and its query and schema objects produce their own SQL, as
[RFC-0003](0003-database-component.md) describes.
[ADR-0016](../adr/0016-postgresql-16-is-the-only-supported-database.md) makes PostgreSQL the only supported database,
and [ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md) has SQL produced by a
[compiler](../GLOSSARY.md#compiler).

Both changes reach the same classes, so they will arrive together: the dialect changes what SQL is produced, and the
compiler changes what produces it.

## Proposal

### The compiler seam

Every object the compiler understands will implement `Node`, and `Query`, `Schema`, `Column` and `Index` will extend it.
An expression will be narrowed to a fragment producing a value, such as raw SQL. A `NodeCompiler` will compile one type
of [node](../GLOSSARY.md#node), given the [compiler](../GLOSSARY.md#compiler) for the nodes it contains.

```php
interface Node {}
interface Expression extends Node {}

interface NodeCompiler
{
    public function compile(Node $node, Compiler $compiler): CompiledSql;
}
```

`Compiler::compile(Node $node): CompiledSql` will own no syntax of its own. It will resolve the compiler registered for
the node's class from the [catalogue](../GLOSSARY.md#catalogue) and pass itself, so a compiler recurses for the nodes it
contains: a table's compiler will never need to know how a column renders.

`CompilerRegistry` will be the [registry](../GLOSSARY.md#registry) that collects compilers, sealed into
`CompilerCatalogue`, following [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md).

```php
final readonly class CompiledSql
{
    public string $sql;
    public array $bindings;

    public static function of(string $sql, mixed ...$bindings): self;
    public function append(string $sql, CompiledSql ...$fragments): self;
}
```

A fragment of [compiled SQL](../GLOSSARY.md#compiled-sql) will carry its own bound values, and composing fragments will
concatenate the SQL and the values in the same order, so a placeholder and its value cannot be produced separately.

Every name written into SQL will go through `Identifier`, a value object that quotes and escapes it, whether it is
qualified or a wildcard:

- It will be rendered in double quotes, with any embedded quote doubled.
- It will be quoted unconditionally, so names are case sensitive exactly as written. The
  [engine](../GLOSSARY.md#engine)'s own names are lower snake case, so the folding difference never arises in practice,
  though a table declared otherwise needs quoting by hand thereafter.
- Qualified names and wildcards will be handled, which is what allows a schema to qualify a name later without the
  compilers changing.
- A name containing a null byte will be rejected, and so will one over 63 bytes, because PostgreSQL truncates silently
  at that length and two long generated index names could otherwise collide.

A matching quoter will handle comments and any other literal that cannot be a bound value.

The compiler will be the only point at which a complete statement exists before it becomes a string, so it will carry a
hook for refusing one.

### Connections and transactions

`ConnectionFactory` will produce connections that use the PostgreSQL driver, which gives the
[engine](../GLOSSARY.md#engine) access to notifications, bulk copying and large objects without a second extension. A
[connection](../GLOSSARY.md#connection) will hold its options, run its transactions and provide each
[primitive](../GLOSSARY.md#primitive).

- **Connection string.** It will gain the SSL mode, a connection timeout and an application name. The application name
  will make it possible to tell which part of the panel holds a lock.
- **Sockets.** A socket setting will mean the directory PostgreSQL's socket lives in, rather than a path to a socket
  file.
- **Options.** These will be split into overridable defaults and a forced set, so nothing reachable from configuration
  can disable the error mode the connection depends on. Emulated prepares will be dropped, since the driver always
  prepares natively.
- **Timeouts.** Statement, lock and idle-in-transaction timeouts will become connection configuration, applied when the
  connection is made.
- **Persistent connections.** These will be applied, with the session reset on connect so a reused connection inherits
  no prepared statements, temporary tables, search path or session locks. They are incompatible with a
  transaction-mode pooler.
- **Passwords.** These may be empty, which is what socket peer authentication uses.

Transactions will become savepoint-aware, which is the sharpest change in behaviour in this design. PostgreSQL aborts
an entire transaction when any statement fails, and every later statement fails until it is rolled back. A nested
transaction will therefore issue a savepoint, a failure inside it will roll back to that savepoint, and only the
outermost call will commit. Without that, code catching a query failure and carrying on inside the same unit of work
is silently broken.

`lastInsertId` will be removed from the write result rather than fixed, and `RETURNING` will replace it.

The driver returns native integers, floats and booleans where MySQL returned strings, so the typed accessors on a
[row](../GLOSSARY.md#row) from [RFC-0003](0003-database-component.md) will carry weight they did not before. Their
representations will be pinned by tests rather than assumed.

### Queries

Each query object will keep its builder methods and lose its rendering. `Select`, `Insert`,
`Update`, `Delete` and the raw expression will each become a [node](../GLOSSARY.md#node), the clause objects will gain
their own compilers, and the traits that compose shared clauses will keep only their fluent methods.

`Connection::query()`, `execute()` and `stream()` will accept a node or a string and compile it through the
[compiler](../GLOSSARY.md#compiler) they are given, so being executable will mean a compiler is registered for the node,
rather than the object having a method that returns SQL.

Moving the nodes behind the compiler will change no SQL except that names are quoted. Every dialect change is separate
from that move.

#### Writes

- **Insert ignoring conflicts.** `ON CONFLICT DO NOTHING` will replace `INSERT IGNORE`.
- **Replace.** This will be removed with no replacement. Deleting the conflicting row and inserting a new one fires
  delete triggers and loses every column not mentioned; an upsert is what is wanted instead.
- **Upsert.** It will be `ON CONFLICT` with an explicit target, either a column list or a named constraint, followed by
  either nothing or an update with an optional condition. The excluded row will be available, since referring to the
  value that failed to insert is the point of the construct.
- **Returning.** This will be available on insert, update and delete. Asking for it will change the type of the
  [node](../GLOSSARY.md#node) so the statement is run as a query and yields rows, which keeps the write result a simple
  value. It will be the answer to the removed `lastInsertId`, and it will work for multi-row inserts, which
  `lastInsertId` never did.
- **Multi-table writes.** An update may take a `FROM`, and a delete a `USING`, so neither needs a correlated subquery.
- **Ordering and limiting writes.** These will be removed from update and delete, which PostgreSQL does not support.
  The replacement will be a subquery selecting the keys to act on.
- **Unguarded writes.** An update or delete compiled with no condition will be refused unless the caller has said that
  is what they meant. This is possible only because the compiler will see the whole statement first.

#### Reads

- **Locking.** `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` and `FOR KEY SHARE` will be available, each able to skip
  locked rows, refuse to wait, or name the tables it applies to. Skipping locked rows is what a job queue claims work
  with.
- **Select constructs.** A select will gain distinct on named expressions, common table expressions including recursive
  and data-modifying ones, lateral joins, and window functions with partitions and frames.
- **Pattern matching.** Case-insensitive matching and the regular expression operators will be available.
- **JSON.** Access and containment operators over `jsonb` will be available. Existence will be expressed through the
  function forms, because the operators collide with the placeholder marker and a query containing them either fails
  to parse or has the operator taken as a parameter.
- **Arrays.** Containment and overlap will be available, along with comparison against any or all elements.
- **Full-text search.** A text search vector will be matched against a query parsed from what people actually type,
  with ranking available for ordering. It will replace the MySQL full-text expression, which will be removed.
- **Offset.** An offset without a limit will be valid, which it was not under MySQL.

### Schema

#### The type model

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

Timestamps will always be time zone aware. A panel scheduling restarts across time zones with a naive timestamp is a
bug waiting for the clocks to change.

The per-family column classes will remain, since they are what makes the shared templates work, but their membership
will change and new families will arrive. An array will be a modifier on any column rather than a family of its own,
because almost any type can be an array.

Modifiers will gain a collation, a check constraint, an identity column, a stored generated column, and default
expressions alongside default literals. They will lose column placement, unsigned, per-column character sets, and
virtual generated columns, which need a version above the minimum the panel supports.

An enumeration will be text with a check constraint by default, because changing the permitted values is then a
constraint swap inside a transaction. Opting into a native type will give something compact and introspectable, at the
cost of a schema object with a lifecycle of its own.

#### DDL takes no bound values

PostgreSQL accepts no placeholders in schema statements, so a default, a check expression and an enumeration's values
are all literal text. Every literal will go through the quoter, and a schema [node](../GLOSSARY.md#node) will compile to
a statement with no bound values by construction.

#### One node, several statements

A table's definition will no longer compile to one string. Only primary keys and uniqueness can be expressed inline,
so every other index is a separate statement, and a comment on a table or a column is a statement of its own.

Compiling a schema [node](../GLOSSARY.md#node) will therefore produce a [statement list](../GLOSSARY.md#statement-list),
executed in order inside one transaction. PostgreSQL runs schema changes inside transactions, so a migration failing at
the seventh statement rolls back the first six. The exceptions are creating an index concurrently and creating a
database, neither of which can run inside a transaction, so a node will advertise that rather than failing when it is
executed.

#### Tables, indexes and objects

- **Tables.** The storage engine and the table-level character set and collation will be removed, since encoding belongs
  to the database. Unlogged tables, for genuinely disposable data, and creation only when absent will be added.
- **Altering.** An alter will take several actions in one statement, and a type change will carry an explicit
  conversion where the cast is not implicit.
- **Dropping and truncating.** Both will gain cascading and restricting, and truncation will be able to restart
  identity columns.
- **Indexes.** A method may be chosen, and an index may be on an expression, partial, covering, unique with nulls not
  distinct, or use an operator class.
- **Constraints.** Primary keys, uniqueness, foreign keys with their actions and deferability, checks, and exclusion
  constraints will be available. Exclusion constraints have no MySQL equivalent and let the database guarantee that no
  two rows overlap.
- **Objects.** Schemas, enumerated types and extensions will be able to be created and dropped, which the native
  enumeration path needs and which makes qualifying a schema per [module](../GLOSSARY.md#module) possible later.

### Streaming

`stream()` will become a real server-side cursor: the query will be declared as a cursor, rows will be fetched forward
in batches, and the cursor will be closed. Memory will then stay flat whatever the size of the result, where before the
driver retrieved everything before the first row was handed back.

A cursor lives inside a transaction, so streaming will either join the caller's transaction or open one for as long as
the cursor is open. This will have two consequences: a long stream will hold a transaction open, which holds back
vacuuming, and a slow consumer can be killed by the idle-in-transaction timeout that protects the panel from wedged
connections.

The cursor will be closed and the transaction resolved however iteration ends, including a consumer breaking out early
or a callback throwing, because a leaked cursor holds its transaction and a leaked transaction eventually holds
everything.

### Primitives

Each [primitive](../GLOSSARY.md#primitive) will exist because a designed subsystem needs it. Each will be a thin
wrapper, with no policy.

- **Notifications.** Sending will be emitted as a function call, so the channel and payload are bound values rather than
  interpolated text. Delivery is transactional, so a notification sent inside a transaction fires only on commit. A
  listener will take its own dedicated [connection](../GLOSSARY.md#connection) and block with a timeout rather than
  polling.
- **Advisory locks.** A lock will be taken for the duration of a transaction, so it cannot outlive its unit of work or
  leak when a process dies. A lock taken outside a transaction will open one. Keys will be namespaced per subsystem, so
  two subsystems cannot collide, though two names within one namespace still can.
- **Bulk copying.** Rows will be read and written in bulk, in a format that handles quoting and embedded delimiters,
  which is an order of magnitude faster than many-row inserts. It will be for moving rows, where streaming is for
  processing them.
- **Session variables.** Each will be set for the duration of a transaction, through a function so the value is bound,
  and will carry the request identity that row-level policies read. Being transaction-scoped is what stops it leaking
  into the next request on a reused connection.

### Testing

Every construct the [compiler](../GLOSSARY.md#compiler) can produce will have a fixture, and one integration test will
prepare each fixture's SQL against a real server, so the test will prove the database accepts the statement rather than
proving the compiler produced what the test author typed. The fixture set will be shared with the unit tests, so a
construct cannot be added with a string comparison alone.

### Out of scope

- **The migration runner.** It inherits the two statements that cannot run inside a transaction.
- **Splitting database roles and row-level policies.** The session variable makes them possible, but they belong with
  whoever designs installation.
- **The subsystems the primitives exist for.** These are delivering events durably, the job queue, the scheduler, and
  any policy.
- **Reconnection and backoff for a listener.** This belongs to whatever consumes notifications.
- **Declarative partitioning.** It has no consumer yet.
- **`MERGE`.** It is available at the minimum supported version, but it is the wrong tool for an upsert.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- database access built directly on PDO, in [ADR-0008](../adr/0008-database-access-is-built-directly-on-pdo.md)
- PostgreSQL as the only supported database, in
  [ADR-0016](../adr/0016-postgresql-16-is-the-only-supported-database.md)
- SQL produced by a compiler, in
  [ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md)

**Escaping the JSON existence operators rather than using their function forms.** We rejected this because, although
doubling the marker works on this driver, it puts a driver quirk into generated SQL that anyone reading a log has to
decode.

**Fixing the last inserted identifier rather than removing it.** We rejected this because asking the connection
afterwards cannot answer for a multi-row insert and reports a stale value after an update or delete, where returning
rows from the statement that generated them answers both.

**Holding a cursor beyond its transaction.** We rejected this because, although it is possible, it materialises the
whole result when the transaction commits, which is the cost streaming exists to avoid.

No other alternatives were weighed.

## Backwards compatibility

- MySQL and MariaDB will no longer be supported, and everything written against their dialect will change: the upsert
  and replace constructs, full-text search, the type model, and ordering or limiting a write.
- A query or schema object will no longer produce its own SQL, so anything calling those methods
  will compile the [node](../GLOSSARY.md#node) instead, and a [connection](../GLOSSARY.md#connection) will need a
  [compiler](../GLOSSARY.md#compiler).
- Names will be quoted, so generated SQL will differ from today's even where the construct is unchanged.
- The last inserted identifier will be gone from the write result, replaced by returning rows.
- A socket setting will name a directory rather than a file.

## Open questions

- **The shape of a compiled result carrying several statements.** The column model and the table work agree it is needed
  and that the table work delivers it, but neither settles what it looks like.
- **How a caller opts out of the unguarded-write guard.**
- **The builder methods for an upsert**, which the record describes by behaviour rather than by name.
- **Whether writes needing ordering get a helper**, or only a documented subquery pattern.
- **How a module registers a node and its compiler**, which the compiler seam makes possible but does not describe.
- **The namespace values for advisory locks**, one per subsystem.
- **The default batch size for a cursor.**

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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
