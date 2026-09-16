---
title: Database
includes: [ADR-0002, ADR-0007, ADR-0008, RFC-0003]
---

# Database

Database access is built directly on PDO, per
[ADR-0008](../adr/0008-database-access-is-built-directly-on-pdo.md). Any number of named connections are
configured, one is the primary, and a connection executes queries and manages transactions.

Only MySQL is reachable, per [ADR-0007](../adr/0007-mysql-and-mariadb-are-the-default-database.md). The driver is
not configurable: `ConnectionConfig` sets it to `mysql` in its constructor.

## Configuration

`DatabaseConfig` and `ConnectionConfig` are configuration objects, hydrated through `fromArray()`. Array shapes are
asserted there, and value rules in each constructor, so `make()` cannot bypass them.

| `DatabaseConfig` | Rule |
|---|---|
| `primary` | Required, not empty, and a connection must be configured under it. |
| `connections` | Required, a non-empty map. Each entry is a non-empty map hydrated into a `ConnectionConfig`, and a failure is rethrown naming the connection. |
| `persistent` | Optional, boolean, defaulting to `false`. |

| `ConnectionConfig` | Rule |
|---|---|
| `database`, `username`, `password` | Required, and none may be empty. |
| `socket` | Optional. When set, the host and port are ignored and left `null`. |
| `host`, `port` | Required when no socket is given. The port must be an integer. |
| `options` | Optional, and an array. |
| `driver` | Not configurable. Always `mysql`. |

`persistent` is held and validated. Nothing reads it, so no connection is created with `PDO::ATTR_PERSISTENT`.

## Connections

`ConnectionFactory` is constructed with the `DatabaseConfig` and hands out connections:

```php
$primary = $factory->make();
$replica = $factory->make('replica');
```

`make(?string $name = null)` uses the primary connection's name when none is given, returns a connection already
created under that name, and otherwise builds one and keeps it for the life of the factory. A name with no
configuration throws `ConnectionException`, and so does a PDO failure while connecting.

The DSN is built from the configuration:

| Configuration | DSN |
|---|---|
| A socket | `mysql:unix_socket={socket};dbname={database}` |
| A host | `mysql:host={host};port={port};dbname={database}` |

A driver other than `mysql` throws `DatabaseException`.

### Options

Every connection is created with these defaults:

| Option | Value |
|---|---|
| `PDO::ATTR_ERRMODE` | `PDO::ERRMODE_EXCEPTION` |
| `PDO::ATTR_DEFAULT_FETCH_MODE` | `PDO::FETCH_ASSOC` |
| `PDO::ATTR_EMULATE_PREPARES` | `false` |
| `PDO::ATTR_STRINGIFY_FETCHES` | `false` |

The configured options are combined as `$config->options + self::$defaultOptions`, so a configured option takes
precedence over the default of the same name, the error mode included.

### Injection

A connection is injected into a parameter typed exactly `Connection` and carrying the `Database` attribute, which is
a resolvable attribute paired with `DatabaseResolver`, per
[ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md).

```php
public function __construct(
    #[Database] private Connection $primary,
    #[Database('replica')] private Connection $replica,
) {}
```

The resolver returns the connection for the attribute's name, so a parameter with no name receives the primary. It
throws `DatabaseException` when the dependency's resolvable attribute is not `Database`, and when the parameter's
type is not exactly `Connection`.

## Executing

`Connection` carries its name and wraps a PDO connection.

| Method | Returns |
|---|---|
| `query(Expression\|string $query, array $bindings = [])` | A `Result`. |
| `execute(Expression\|string $query, array $bindings = [])` | A `WriteResult`. |
| `stream(Expression\|string $query, array $bindings = [])` | A `Cursor`. |

Each takes SQL as a string with its bound values, or an expression, whose SQL and bound values are read from
`toSql()` and `getBindings()`. Bound values given alongside an expression are discarded.

Every statement is prepared and then executed. Booleans among the bound values are converted to integers first. A
statement that cannot be prepared or executed throws `QueryException`, which carries the SQL and bound values
through `getSql()` and `getBindings()`, and the PDO failure as its previous exception.

`execute()` reads the last inserted identifier from the connection rather than from the statement, so the value it
returns after an update or a delete is whatever that connection last inserted.

### Results

`Result` fetches every row the first time any row is read, and keeps them.

| Method | Effect |
|---|---|
| `first()` | The first row, or `null`. |
| `all()` | Every row. |
| `each(callable $callback)` | Calls the callback with each row. |
| `count()` | The statement's row count. |
| `isEmpty()` | Whether that count is zero. |

`Cursor` fetches one row at a time as it is iterated, so a large result is not held in memory at once. Its rows can
be iterated once, through `each()` or the generator from `rows()`. It has the same `count()` and `isEmpty()`.

Both take their row count from PDO's `rowCount()`, which for a select is driver-dependent.

`WriteResult` holds `affectedRows()`, `lastInsertId()` and `wasSuccessful()`, which is whether at least one row was
affected. `Result`, `Cursor` and `WriteResult` each carry the bound values they were executed with.

### Rows

A `Row` holds one row's values, keyed by column name.

| Method | Effect |
|---|---|
| `get(string $column)` | The value, or `null` when the column is absent. |
| `has(string $column)` | Whether the column is present, including when its value is `null`. |
| `isNull(string $column)` | Whether the column is present and holds `null`. An absent column returns `false`. |
| `toArray()` | Every value, keyed by column name. |
| `string()`, `int()`, `float()`, `bool()`, `array()` | The value cast to that type, through `ValueGetter`. |

### Transactions

| Method | Effect |
|---|---|
| `transaction(callable $callback)` | Begins a transaction, calls the callback with the connection, commits, and returns the callback's result. A throwing callback rolls back and the exception is rethrown. |
| `beginTransaction()`, `commit()`, `rollback()` | Begin, commit and roll back directly. |
| `isInTransaction()` | Whether a transaction is active. |

A PDO failure beginning, committing or rolling back throws `DatabaseException`. Transactions do not nest, and there
are no savepoints.

## Migrations

`Migrator` is an empty class. Nothing runs migrations, records which have run, or rolls one back. The `Migration`
and `ReversibleMigration` contracts exist under the schema builder.

## Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `DatabaseException` | `RuntimeException` | The driver is not supported, a transaction cannot be begun, committed or rolled back, or the resolver is given a dependency it cannot resolve. |
| `ConnectionException` | `DatabaseException` | No configuration exists for a connection name, or the connection cannot be established. It carries the name. |
| `QueryException` | `DatabaseException` | A statement cannot be prepared or executed. It carries the SQL and bound values. |
