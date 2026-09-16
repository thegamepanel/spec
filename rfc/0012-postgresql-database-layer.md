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

The database component moves to PostgreSQL and behind a compiler. Query and schema objects become nodes describing
what is wanted, and a compiler resolved by node class turns each into SQL and its bound values together. Connections
produce the PostgreSQL driver, transactions nest through savepoints, the type model and DDL become PostgreSQL's, and
the engine gains the primitives that later subsystems are built on.

## Motivation

The database component assumes MySQL and produces its SQL inside the query and schema objects themselves, per
[RFC-0003](0003-database-component.md). PostgreSQL is now the only supported database, per
[ADR-0016](../adr/0016-postgresql-16-is-the-only-supported-database.md), and SQL is produced by a compiler, per
[ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md).

Both changes reach the same classes, so they arrive together: the dialect changes what SQL is produced, and the
compiler changes what produces it.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Node | An object describing part of a statement, which a compiler turns into SQL. |
| Compiler | Turns a node into SQL and its bound values, recursing for the nodes it contains. |
| Compiled SQL | The SQL of one statement and the values bound to it, produced together. |
| Identifier | A name written into SQL, quoted and escaped by one value object. |
| Statement list | The ordered statements one schema node compiles into, executed together. |
| Primitive | A thin wrapper over something PostgreSQL provides that a later subsystem needs. |

### Components

| Component | Responsibility |
|---|---|
| `Node` | Marks anything the compiler understands. `Query`, `Schema`, `Column` and `Index` extend it. |
| `Expression` | Narrowed to a fragment producing a value, such as raw SQL. |
| `NodeCompiler` | Compiles one type of node, given the compiler for its children. |
| `Compiler`, `CompilerRegistry`, `CompilerCatalogue` | Dispatch to a node's compiler, collected and then sealed. |
| `CompiledSql` | SQL and bound values, composed together as fragments combine. |
| `Identifier` | Quotes and escapes a name, qualified or wildcard. |
| `Connection`, `ConnectionFactory` | The PostgreSQL connection, its options, transactions and primitives. |
| `Cursor` | Server-side iteration over a result. |

### The compiler seam

```php
interface Node {}
interface Expression extends Node {}

interface NodeCompiler
{
    public function compile(Node $node, Compiler $compiler): CompiledSql;
}
```

`Compiler::compile(Node $node): CompiledSql` owns no syntax of its own. It resolves the compiler registered for the
node's class from the catalogue and passes itself, so a compiler recurses for the nodes it contains: a table's
compiler never needs to know how a column renders.

`CompilerRegistry` collects compilers and seals into `CompilerCatalogue`, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md).

```php
final readonly class CompiledSql
{
    public string $sql;
    public array $bindings;

    public static function of(string $sql, mixed ...$bindings): self;
    public function append(string $sql, CompiledSql ...$fragments): self;
}
```

A fragment carries its own bound values, and composing fragments concatenates the SQL and the values in the same
order, so a placeholder and its value cannot be produced separately.

Every name written into SQL goes through `Identifier`:

- It is rendered in double quotes, with any embedded quote doubled.
- It is quoted unconditionally, so names are case sensitive exactly as written. The engine's own names are lower
  snake case, so the folding difference never arises in practice, though it means a table declared otherwise needs
  quoting by hand thereafter.
- Qualified names and wildcards are handled, which is what allows a schema to qualify a name later without the
  compilers changing.
- A name containing a null byte is rejected, and so is one over 63 bytes, because PostgreSQL truncates silently at
  that length and two long generated index names could otherwise collide.

A matching quoter handles comments and any other literal that cannot be a bound value.

The compiler is the only point at which a complete statement exists before it becomes a string, so it carries a hook
for refusing one.

### Connections and transactions

The factory produces the PostgreSQL driver, which gives the engine access to notifications, bulk copying and large
objects without a second extension.

| Change | Detail |
|---|---|
| Connection string | Gains the SSL mode, a connection timeout and an application name, the last making it possible to tell which part of the panel holds a lock. |
| Sockets | A socket setting now means the directory PostgreSQL's socket lives in, rather than a path to a socket file. |
| Options | Split into overridable defaults and a forced set, so nothing reachable from configuration can disable the error mode the connection depends on. Emulated prepares are dropped, since the driver always prepares natively. |
| Timeouts | Statement, lock and idle-in-transaction timeouts become connection configuration, applied when the connection is made. |
| Persistent connections | Applied, with the session reset on connect so a reused connection inherits no prepared statements, temporary tables, search path or session locks. Incompatible with a transaction-mode pooler. |
| Passwords | May be empty, which is what socket peer authentication uses. |

Transactions become savepoint-aware, which is the sharpest behavioural change in this design. PostgreSQL aborts an
entire transaction when any statement fails, and every later statement fails until it is rolled back. A nested
transaction therefore issues a savepoint, a failure inside it rolls back to that savepoint, and only the outermost
call commits. Without that, code catching a query failure and carrying on inside the same unit of work is silently
broken.

`lastInsertId` is removed from the write result rather than fixed. `RETURNING` replaces it.

The driver returns native integers, floats and booleans where MySQL returned strings, so the typed accessors in
[RFC-0003](0003-database-component.md) carry weight they did not before, and the representations are pinned by
tests rather than assumed.

### Queries

Query objects keep their builder methods and lose their rendering. `Select`, `Insert`, `Update`, `Delete` and the raw
expression become nodes, the clause objects gain their own compilers, and the traits that compose shared clauses keep
their fluent methods only.

`Connection::query()`, `execute()` and `stream()` accept a node or a string and compile through the compiler they are
given, so being executable means a compiler is registered for the node rather than the object having a method that
returns SQL.

Moving the nodes behind the compiler changes no SQL except that names are now quoted. Every dialect change is
separate from that move.

#### Writes

| Construct | Change |
|---|---|
| Insert ignoring conflicts | `ON CONFLICT DO NOTHING`, replacing `INSERT IGNORE`. |
| Replace | Removed with no replacement. Deleting the conflicting row and inserting a new one fires delete triggers and loses every column not mentioned; an upsert is what is wanted instead. |
| Upsert | `ON CONFLICT` with an explicit target, either a column list or a named constraint, then either nothing or an update with an optional condition. The excluded row is available, since referring to the value that failed to insert is the point of the construct. |
| Returning | Available on insert, update and delete. Asking for it changes the node's type so the statement is run as a query and yields rows, which keeps the write result a simple value. It is the answer to the removed `lastInsertId`, and works for multi-row inserts, which that never did. |
| Multi-table writes | An update may take a `FROM`, and a delete a `USING`, so neither needs a correlated subquery. |
| Ordering and limiting writes | Removed from update and delete, which PostgreSQL does not support. The replacement is a subquery selecting the keys to act on. |
| Unguarded writes | An update or delete compiled with no condition is refused unless the caller has said that is what they meant. This is possible only because the compiler sees the whole statement first. |

#### Reads

| Construct | Detail |
|---|---|
| Locking | `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE` and `FOR KEY SHARE`, each able to skip locked rows, refuse to wait, or name the tables it applies to. Skipping locked rows is what a job queue claims work with. |
| Select constructs | Distinct on named expressions, common table expressions including recursive and data-modifying ones, lateral joins, and window functions with partitions and frames. |
| Pattern matching | Case-insensitive matching and the regular expression operators. |
| JSON | Access and containment operators over `jsonb`. Existence is expressed through the function forms, because the operators collide with the placeholder marker and a query containing them either fails to parse or has the operator taken as a parameter. |
| Arrays | Containment and overlap, and comparison against any or all elements. |
| Full-text search | A text search vector matched against a query parsed from what people actually type, with ranking available for ordering. It replaces the MySQL full-text expression, which is removed. |
| Offset | An offset without a limit is valid, which it was not under MySQL. |

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

Timestamps are always time zone aware. A panel scheduling restarts across time zones with a naive timestamp is a bug
waiting for the clocks to change.

The per-family column classes remain, since they are what makes the shared templates work, but their membership
changes and new families arrive. An array is a modifier on any column rather than a family of its own, because
almost any type can be an array.

Modifiers gain a collation, a check constraint, an identity column, a stored generated column, and default
expressions alongside default literals. They lose column placement, unsigned, per-column character sets, and virtual
generated columns, which need a version above the floor.

An enumeration is text with a check constraint by default, because changing the permitted values is then a
constraint swap inside a transaction. Opting into a native type gives something compact and introspectable, at the
cost of a schema object with a lifecycle of its own.

#### DDL takes no bound values

PostgreSQL accepts no placeholders in schema statements, so a default, a check expression and an enumeration's
values are all literal text. Every literal goes through the quoter, and a schema node compiles to a statement with
no bound values by construction.

#### One node, several statements

A table's definition no longer compiles to one string. Only primary keys and uniqueness can be expressed inline, so
every other index is a separate statement, and a comment on a table or a column is a statement of its own.

Compiling a schema node therefore produces an ordered list of statements, executed in order inside one transaction.
That PostgreSQL wraps schema changes in transactions is the quiet win of this change: a migration failing at the
seventh statement rolls back the first six. The exceptions are creating an index concurrently and creating a
database, neither of which can run inside a transaction, so a node advertises that rather than failing when it is
executed.

#### Tables, indexes and objects

| Area | Change |
|---|---|
| Tables | Lose the storage engine and table-level character set and collation, since encoding belongs to the database. Gain unlogged tables, for genuinely disposable data, and creation only when absent. |
| Altering | Several actions in one statement, and a type change carrying an explicit conversion where the cast is not implicit. |
| Dropping and truncating | Gain cascading and restricting, and truncation can restart identity columns. |
| Indexes | A method may be chosen, and an index may be on an expression, partial, covering, unique with nulls not distinct, or use an operator class. |
| Constraints | Primary keys, uniqueness, foreign keys with their actions and deferability, checks, and exclusion constraints, which have no MySQL equivalent and let the database guarantee that no two rows overlap. |
| Objects | Schemas, enumerated types and extensions can be created and dropped, which the native enumeration path needs and which makes qualifying a schema per module possible later. |

### Streaming

`stream()` becomes a real server-side cursor: the query is declared as a cursor, rows are fetched forward in batches
and the cursor is closed. Memory then stays flat whatever the size of the result, where before the driver retrieved
everything before the first row was handed back.

A cursor lives inside a transaction, so streaming either joins the caller's transaction or opens one for the
cursor's lifetime. Two consequences are worth stating rather than discovering: a long stream holds a transaction
open, which holds back vacuuming, and a slow consumer can be killed by the idle-in-transaction timeout that protects
the panel from wedged connections.

The cursor is closed and the transaction resolved however iteration ends, including a consumer breaking out early or
a callback throwing, because a leaked cursor holds its transaction and a leaked transaction eventually holds
everything.

### Primitives

Each exists because a designed subsystem needs it. All are thin wrappers, with no policy.

| Primitive | Shape |
|---|---|
| Notifications | Sending is emitted as a function call so the channel and payload are bound values rather than interpolated text. Delivery is transactional, so a notification inside a transaction fires only on commit, and a listener takes its own dedicated connection and blocks with a timeout rather than polling. |
| Advisory locks | Taken for the duration of a transaction, so a lock cannot outlive its unit of work or leak when a process dies. A lock taken outside a transaction opens one. Keys are namespaced per subsystem, so two subsystems cannot collide, though two names within one namespace still can. |
| Bulk copying | Reading and writing rows in bulk, in a format that handles quoting and embedded delimiters, which is an order of magnitude faster than many-row inserts. It is for moving rows, where streaming is for processing them. |
| Session variables | Set for the duration of a transaction, through a function so the value is bound, carrying the request identity that row-level policies read. Being transaction-scoped is what stops it leaking into the next request on a reused connection. |

### Testing

Every construct the compiler can produce has a fixture, and one integration test prepares each fixture's SQL against
a real server, so the test proves the database accepts the statement rather than proving the compiler produced what
the test author typed. The fixture set is shared with the unit tests, so a construct cannot be added with a string
comparison alone.

### Out of scope

- **The migration runner**, which inherits the two statements that cannot run inside a transaction.
- **Splitting database roles and row-level policies**, which the session variable makes possible but which belong
  with whoever designs installation.
- **The subsystems the primitives exist for**: delivering events durably, the job queue, the scheduler, and any
  policy.
- **Reconnection and backoff for a listener**, which belongs to whatever consumes notifications.
- **Declarative partitioning**, which has no consumer yet.
- **`MERGE`**, which is available at the floor but is the wrong tool for an upsert.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- PostgreSQL as the only supported database, in
  [ADR-0016](../adr/0016-postgresql-16-is-the-only-supported-database.md)
- SQL produced by a compiler, in
  [ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md)

**Escaping the JSON existence operators rather than using their function forms.** Doubling the marker works on this
driver, but it puts a driver quirk into generated SQL that anyone reading a log has to decode.

**Fixing the last inserted identifier rather than removing it.** Asking the connection afterwards cannot answer for a
multi-row insert and reports a stale value after an update or delete, where returning rows from the statement that
generated them answers both.

**Holding a cursor beyond its transaction.** It is possible, but it materialises the whole result when the
transaction commits, which is the cost streaming exists to avoid.

No other alternatives were weighed.

## Backwards compatibility

- MySQL and MariaDB are no longer supported, and everything written against their dialect changes: the upsert and
  replace constructs, full-text search, the type model, and ordering or limiting a write.
- Query and schema objects no longer produce their own SQL, so anything calling those methods compiles the node
  instead, and a connection needs a compiler.
- Names are quoted, so generated SQL differs from today's even where the construct is unchanged.
- The last inserted identifier is gone from the write result, replaced by returning rows.
- A socket setting now names a directory rather than a file.

## Open questions

- **The shape of a compiled result carrying several statements.** The column model and the table work agree it is
  needed and that the table work delivers it, but neither settles what it looks like.
- **How a caller opts out of the unguarded-write guard.**
- **The builder methods for an upsert**, which the issues describe by behaviour rather than by name.
- **Whether writes needing ordering get a helper**, or only a documented subquery pattern.
- **How a module registers a node and its compiler**, which the compiler seam makes possible but does not describe.
- **The namespace values for advisory locks**, one per subsystem.
- **The default batch size for a cursor.**

## Changelog

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
