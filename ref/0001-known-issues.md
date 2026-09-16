---
id: REF-0001
title: Known issues
verified: 2026-09-16
---

# REF-0001: Known issues

Defects known to be present in the built code, as at commit
[3372f20](https://github.com/thegamepanel/panel/commit/3372f20).

This is informational and never normative. Where an entry contradicts a document in this repository, that document
is named. Some entries contradict nothing, because the behaviour was never specified; they appear anyway, since
what a reader needs is the defect and its consequence, not its documentation status.

An issue link means the defect is tracked there, not that it is fixed. [#41], [#42] and [#48] are closed, and the
behaviour each describes is still present: each was closed in favour of the PostgreSQL work rather than by a change
to the code. An entry leaves this document when the behaviour changes, struck through with what changed it, so the
record of its having existed survives.

## Security

### Identifiers reach SQL unescaped

Values passed to the query builder are bound and are safe. Identifiers are not. A table, column or index name is
interpolated into the SQL as given, so a name taken from untrusted input can close the statement and append
another.

The surface is every place a name is written rather than bound:

| Where | Written as |
|---|---|
| The table of an insert, update or delete | Unquoted. |
| A selected column, and the table of a select | Unquoted. |
| Both sides of a join's `ON` condition, and the joined table | Unquoted. |
| A column in a `WHERE`, `HAVING`, `GROUP BY` or `ORDER BY` clause | Unquoted. |
| A column inside an aggregate | Unquoted. |
| A column in an `UPDATE ... SET` clause | Between backticks, with no backtick inside it doubled. |
| Every identifier in a schema statement | Between backticks, with no backtick inside it doubled. |

Quoting without escaping is not protection: a name containing a backtick closes the quoting and everything after it
is SQL.

**The operator in a join condition is interpolated too.** `JoinClause::on()` writes `{$left} {$operator} {$right}`
and validates none of the three. A `WHERE` clause is better off, because its operator is matched against a known
set and an unrecognised one throws, but a join's is not.

Tracked by [#41], which describes the quoting and escaping but not the operator. The fix is the compiler in
[ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md), where every name becomes SQL
through one identifier object.

**This is latent rather than live.** Nothing in the engine currently takes an identifier from a request: there is
no HTTP layer, no actions, and no caller that builds a query from input. The exposure arrives with the first
consumer that does, which is why it is recorded here rather than left to be rediscovered then.

### Schema literals are quoted without escaping

A column's string default and a column's or table's comment are written between single quotes with nothing escaped,
so a value containing a quote closes it. Schema statements bind no values, so every literal in them is written this
way. Tracked by [#41].

## Container

| What happens | Against | Issue |
|---|---|---|
| A shared binding resolved twice through an alias returns a new instance each time. Instances are stored under the binding's abstract and read back under the class the resolution names, with no alias normalisation on the read. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | [#44] |
| An auto-wired class is never cached. The builder defaults `shared` to true, while resolution reads it as false when there is no binding, so a class with no binding is constructed again on every resolution. Under a worker that is one instance per resolution where one per process is intended. The fallback that would key such an instance by the requested class is therefore unreachable. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | [#63] |
| `Resolution::with()` has no effect. A constructor is invoked as `Invocation::constructor($class)` with nothing passed, and the resolution's arguments are read nowhere, so they are silently discarded. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | none |
| A binding does not record the module scope it was registered under. The builder carries one and does not copy it into the binding; the catalogue keeps a separate map of scope to classes, which nothing reads. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | none |
| `Lazy`, `Liminal` and `NoResolution` on a parameter are handled as resolvable attributes, not by the container. Each implements `Resolvable`, so a parameter carrying one is routed to the resolver registered against that attribute's class, and throws `InvalidResolverException` when none is. Only `Ghost` has a resolver. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | none |
| A qualified instance is cached per qualifier value rather than per qualifier class. The cache compares the qualifier's class and then calls `equals()`. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | none |
| `BindingNotFoundException` is never thrown. It is not referenced anywhere outside its own definition. | [RFC-0001](../rfc/0001-dependency-injection-container.md) | none |
| Neither `BindingRegistry` nor `ResolverRegistry` produces a catalogue. Catalogues are constructed from arrays, so the registries and the catalogues have no seam between them. | [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md) | none |

## Configuration

| What happens | Against | Issue |
|---|---|---|
| `Env::string()` returns the default for a value it cannot cast, where `int()`, `float()` and `bool()` throw. | [RFC-0004](../rfc/0004-toml-configuration-loading.md), [ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md) | none |
| Nothing binds configuration objects into the container, so a configuration class cannot be resolved by type. | [RFC-0002](../rfc/0002-configuration-objects.md) | none |
| Nothing confines `Env` to bootstrap. `destroy()` exists, and once initialised it is readable from anywhere until it is called. | [ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md) | none |
| `MissingEnvVariableException` is never thrown. It is not referenced anywhere outside its own definition. | [RFC-0002](../rfc/0002-configuration-objects.md) | none |

## Database

| What happens | Against | Issue |
|---|---|---|
| A connection's configured options take precedence over the defaults, so a connection configuring `PDO::ATTR_ERRMODE` overrides the error mode everything else relies on. The options are merged with the configured set on the left. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `persistent` is validated and never applied. `PDO::ATTR_PERSISTENT` appears nowhere in the source. | [RFC-0003](../rfc/0003-database-component.md) | none |
| A password may not be empty, which rules out socket peer authentication. | [RFC-0004](../rfc/0004-toml-configuration-loading.md) | [#48] |
| `WriteResult::lastInsertId()` reports the connection's last insert, not the statement's, so it returns a value after an update or a delete. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `Result::count()` and `Cursor::count()` return PDO's `rowCount()`, which is driver-dependent for a select, and `isEmpty()` inherits that. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `Row::isNull()` returns false for a column that is absent, the same answer it gives for a column holding a value. | [RFC-0003](../rfc/0003-database-component.md) | none |
| The `catch` in `Connection::execute()` cannot be reached. The statement helper it wraps already converts a PDO failure into a `QueryException`. | [RFC-0003](../rfc/0003-database-component.md) | none |

## Query builder

| What happens | Against | Issue |
|---|---|---|
| An insert binds each row's values in that row's own key order, against a column list taken from the first row alone, so a later row with the same columns in a different order binds them to the wrong columns. | [RFC-0003](../rfc/0003-database-component.md) | [#42] |
| An insert with no rows raises an undefined array key error rather than throwing `InvalidExpressionException`. | [RFC-0003](../rfc/0003-database-component.md) | [#42] |
| An insert does not check that later rows carry the same columns as the first. Nothing validates the key set, so a row with different keys is written against the first row's column list. | [RFC-0003](../rfc/0003-database-component.md) | [#42] |
| A select gathers its bound values in a different order from its placeholders. The table subquery's values are collected before the columns', while the table is written after the columns, so a select with both a bound subquery table and a bound column expression binds them the wrong way round. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `whereIn()` and `whereNotIn()` given an expression write the subquery with the `AND` conjunction, because they delegate to the raw form rather than the conjunction they were called for. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `orderBy()` accepts any direction. Anything that is not `desc`, in any case, orders ascending, including a misspelling. | [RFC-0003](../rfc/0003-database-component.md) | none |
| An offset set without a limit is written on its own, producing `OFFSET` with no `LIMIT`, which MySQL rejects. | [RFC-0003](../rfc/0003-database-component.md) | none |

## Schema builder

| What happens | Against | Issue |
|---|---|---|
| `after()` and `first()` are recorded on a column and never written into the SQL. | [RFC-0003](../rfc/0003-database-component.md) | none |
| The name given to `Drop::primaryKey()` is discarded. Altering a table records dropping the primary key as a flag. | [RFC-0003](../rfc/0003-database-component.md) | none |
| `Drop` is not a schema statement. It implements the expression contract rather than the schema one. | [RFC-0003](../rfc/0003-database-component.md) | none |

## Decided but not built

These are not defects. Each is a decision this repository records that the code has not yet caught up with, listed
so the gap is visible in one place.

| Decision | State of the code |
|---|---|
| [ADR-0016](../adr/0016-postgresql-16-is-the-only-supported-database.md), PostgreSQL 16 only | The driver is fixed to MySQL and the DSN builder handles no other. |
| [ADR-0017](../adr/0017-sql-is-produced-by-a-compiler-not-by-the-query-objects.md), SQL from a compiler | Every query and schema object still produces its own SQL. |
| [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md), memoised class attributes | Each resolution reflects for the attributes again. |
| [ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md), file access through Flysystem | Configuration loading reads through PHP's own functions. |

## Sources

- [`src/Database/Query` at 3372f20](https://github.com/thegamepanel/panel/tree/3372f20/src/Database/Query): every
  identifier written into SQL by interpolation, the join operator written unvalidated beside them, and the column
  expressions binding their value while interpolating their column, which is what makes values safe and names not.
- [`src/Database/Schema` at 3372f20](https://github.com/thegamepanel/panel/tree/3372f20/src/Database/Schema): names
  written between backticks with no backtick doubled, and string defaults and comments written between single
  quotes with nothing escaped.
- [`src/Container` at 3372f20](https://github.com/thegamepanel/panel/tree/3372f20/src/Container): the alias
  asymmetry between storing and reading a resolved instance, the shared flag read as false without a binding, the
  four marker attributes implementing `Resolvable`, the qualified cache comparing with `equals()`, both registries
  having no method that produces a catalogue, a resolution's arguments being read nowhere, and a binding carrying no
  scope.
- [`src/Config` at 3372f20](https://github.com/thegamepanel/panel/tree/3372f20/src/Config): `Env::string()`
  returning its default where the other typed accessors throw, and nothing binding configuration objects or
  confining `Env`.
- [`src/Database` at 3372f20](https://github.com/thegamepanel/panel/tree/3372f20/src/Database): the options merged
  with the configured set taking precedence, `persistent` never reaching PDO, the last insert identifier read from
  the connection, row counts taken from `rowCount()`, `isNull()` on an absent column, the unreachable catch, the
  insert's column list and binding order, the absence of any check on a later row's columns, the select gathering
  its table subquery's values before its columns' while writing the table after them, the conjunction lost by the
  raw delegation, the order direction, the offset written alone, `after()` and `first()` never written, and the
  discarded primary key name.
- A search of `src/` for `MissingEnvVariableException`, `BindingNotFoundException` and `ATTR_PERSISTENT`, none of
  which is referenced outside its own definition. A search for a caller that builds a query from request input,
  which finds none, since the engine has no HTTP layer.
- Issues [#41], [#42], [#44], [#48] and [#63], which track the entries marked with them. Every other entry has no
  issue yet. [#41], [#42] and [#48] were closed on 2026-08-19 and their entries remain live, which is why a closed
  issue is not read here as a fix.

[#41]: https://github.com/thegamepanel/panel/issues/41
[#42]: https://github.com/thegamepanel/panel/issues/42
[#44]: https://github.com/thegamepanel/panel/issues/44
[#48]: https://github.com/thegamepanel/panel/issues/48
[#63]: https://github.com/thegamepanel/panel/issues/63
