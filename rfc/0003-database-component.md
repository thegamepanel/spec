---
id: RFC-0003
title: Database component
status: accepted
created: 2026-03-29
decided: 2026-04-17
backfilled: 2026-09-14
depends: [ADR-0002, ADR-0005, ADR-0007, ADR-0008, RFC-0001, RFC-0002]
updates: []
obsoletes: []
---

# RFC-0003: Database component

## Abstract

A database component built directly on PDO. Any number of named connections are configured, one of them is
designated the primary connection and used whenever no name is given, and a connection is injected into a parameter
with the `Database` attribute. Queries and schema changes are built as objects that produce their own SQL and
bindings, and a connection executes them, returning fetched results, a cursor or a write result.

## Motivation

The panel needs a simple database abstraction, so that queries can be built programmatically without working in SQL
directly, and so that the database can be created and then changed as the panel and its modules change.

The panel needs to be able to connect to one or more separate databases. The core and first-party components and
modules only ever use one, but support for several is included, with little extra effort, for future situations
that need it.

## Proposal

### Parts

| Part | Responsibility |
|---|---|
| Connections | Configuring connections, creating them, and injecting them. |
| Execution | Running SQL against a connection, returning its results, and transactions. |
| Query builder | Building `SELECT`, `INSERT`, `UPDATE` and `DELETE` queries as objects. |
| Schema builder | Building statements that create, alter and drop databases, tables, columns and indexes, as objects. |

The query builder and schema builder are distinct from connections. A builder produces SQL and the values bound to it,
and knows nothing of the connection it is run against. A connection accepts SQL and bound values, whether as a
string or from a builder.

### Concepts

| Term | Meaning |
|---|---|
| Connection | An open connection to one database, identified by name. |
| Connection name | The name a connection is configured and requested under. |
| Primary connection | The connection used whenever no connection name is given. |
| Driver | The type of database a connection talks to. Only MySQL, including compatible MariaDB, is exposed. |
| Persistent connection | A connection that PDO keeps open and reuses, rather than closing it. |
| Expression | An object that produces SQL, with a `?` placeholder for each bound value, and the values bound to those placeholders. |
| Query | An expression that is a complete query, such as a select or an insert, rather than a fragment such as `col = ?`. |
| Schema statement | An expression that is a complete schema statement, such as creating a table. |
| Condition | One comparison in a `WHERE`, `HAVING` or join clause, joined to the one before it with `AND` or `OR`. |
| Result | The rows returned by a query, fetched in full. |
| Cursor | The rows returned by a query, fetched one at a time. |

### Connections

#### Configuration

`DatabaseConfig` and `ConnectionConfig` are configuration objects, per [RFC-0002](0002-configuration-objects.md) and
[ADR-0005](../adr/0005-configuration-is-held-in-typed-objects.md). Both are created through a static `make()`, and
both restore themselves from an exported array by passing its entries to their constructor as named arguments.

```php
$config = DatabaseConfig::make(
    primary: 'main',
    connections: [
        'main'    => ConnectionConfig::make('127.0.0.1', 3306, null, 'panel', 'panel', 'secret'),
        'replica' => ConnectionConfig::make(null, null, '/var/run/mysqld/mysqld.sock', 'panel', 'panel', 'secret'),
    ],
    persistent: false,
);
```

| `DatabaseConfig` | Holds |
|---|---|
| `primary` | The name of the primary connection. |
| `connections` | Every connection's `ConnectionConfig`, keyed by connection name. |
| `persistent` | Whether every connection is persistent. One flag applies to all of them, and it defaults to `false`. |

A database configuration needs a primary connection name, at least one connection, and a connection configured under
the primary name.

| `ConnectionConfig` | Holds |
|---|---|
| `host` | The host to connect to, when not connecting through a socket. |
| `port` | The port to connect to on the host. Defaults to `3306` when not given. |
| `socket` | The Unix socket to connect through. Used in preference to the host when both are given. |
| `database` | The database to use. |
| `username`, `password` | The credentials to connect with. |
| `options` | PDO options for the connection, applied over the defaults. |
| `driver` | The driver, always `mysql`. It is not configurable. |

The driver is carried by every connection's configuration, so support for other database types can be added inside
the component without restructuring it, but only MySQL is exposed, per
[ADR-0007](../adr/0007-mysql-and-mariadb-are-the-default-database.md).

#### Connection factory

`ConnectionFactory` is constructed with the `DatabaseConfig`, and hands out connections through `make()`:

```php
$primary = $factory->make();
$replica = $factory->make('replica');
```

`make(?string $name = null): Connection` follows these steps:

1. With no name, the primary connection's name is used.
2. A connection already created under the name is returned.
3. Otherwise the connection's configuration is looked up. With no configuration under the name, it throws.
4. A PDO connection is created from the configuration, with its DSN, credentials and options. If it cannot connect,
   it throws.
5. The connection is kept under its name for the lifetime of the factory, and returned.

For MySQL, the DSN is built as follows:

| Configuration | DSN |
|---|---|
| A socket | `mysql:unix_socket={socket};dbname={database}` |
| A host and no socket | `mysql:host={host};port={port};dbname={database}`, with the port defaulting to `3306` |
| Neither | Throws |

A driver other than MySQL throws.

#### PDO options

Every connection is created with these options:

| Option | Value | Configurable |
|---|---|---|
| `PDO::ATTR_ERRMODE` | `PDO::ERRMODE_EXCEPTION` | No, always forced |
| `PDO::ATTR_DEFAULT_FETCH_MODE` | `PDO::FETCH_ASSOC` | Yes |
| `PDO::ATTR_EMULATE_PREPARES` | `false` | Yes |
| `PDO::ATTR_STRINGIFY_FETCHES` | `false` | Yes |

A connection's configured options are applied over the defaults, so they can change any configurable default and add
options of their own. The error mode is always forced to throw exceptions, because everything built on a connection
relies on errors being thrown.

When `persistent` is set on the `DatabaseConfig`, every connection is also created with `PDO::ATTR_PERSISTENT`.

#### Injection

A connection is injected into a parameter typed `Connection` with the `Database` attribute, which takes an optional
connection name:

```php
public function __construct(
    #[Database] private Connection $primary,
    #[Database('replica')] private Connection $replica,
) {}
```

`DatabaseResolver` is the resolver paired with `Database`, the resolver tier of
[ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md), using the mechanism in
[RFC-0001](0001-dependency-injection-container.md). It is resolved through the container, receiving the
`ConnectionFactory`, and returns the connection from `make()` for the attribute's name, so a parameter with no name
receives the primary connection. The parameter must be typed exactly `Connection`.

### Execution

| Component | Responsibility |
|---|---|
| `Connection` | A named connection, wrapping its PDO connection. Executes SQL and manages transactions. |
| `Result` | The rows returned by a query, fetched in full. |
| `Cursor` | The rows returned by a query, fetched one at a time. |
| `WriteResult` | The outcome of a statement that writes. |
| `Row` | One row, with typed access to its columns. |

#### Executing SQL

| Method | Effect |
|---|---|
| `query(Expression\|string $query, array $bindings = []): Result` | Executes a query that returns rows, and returns them as a `Result`. |
| `execute(Expression\|string $query, array $bindings = []): WriteResult` | Executes a statement that writes, and returns a `WriteResult`. |
| `stream(Expression\|string $query, array $bindings = []): Cursor` | Executes a query that returns rows, and returns a `Cursor` over them. |

Each method accepts SQL as a string with the values bound to it, or an expression, whose SQL and bound values are
taken from its `toSql()` and `getBindings()`. Bound values given alongside an expression are ignored.

Every statement is prepared, then executed with its bound values. Booleans among the bound values are converted to
the integers `1` and `0` first, because MySQL treats booleans as integers. A statement that cannot be prepared or
executed throws `QueryException`, which carries its SQL and bound values.

```php
$servers = $connection->query(
    Select::from('servers')->where('status', '=', 'running')->orderBy('name'),
);

foreach ($servers->all() as $server) {
    $name = $server->string('name');
}

$write = $connection->execute(Insert::into('servers')->values(['name' => 'Survival']));
$id    = $write->lastInsertId();
```

`Result`, `Cursor` and `WriteResult` each carry the bound values they were executed with, as `bindings`.

#### Results

A `Result` fetches every row the first time any row is read, and keeps them.

| Method | Effect |
|---|---|
| `first(): ?Row` | Returns the first row, or `null` when there are none. |
| `all(): array` | Returns every row. |
| `each(callable $callback): void` | Calls the callback with each row in turn. |
| `count(): int` | Returns the number of rows. |
| `isEmpty(): bool` | Returns whether there are no rows. |

A `Cursor` fetches rows one at a time as they are iterated, so a large result is processed without every row being
held in memory at once. Its rows can be iterated once.

| Method | Effect |
|---|---|
| `each(callable $callback): void` | Fetches each row in turn and calls the callback with it. |
| `rows(): Generator` | Returns a generator that fetches and yields each row in turn. |
| `count(): int` | Returns the number of rows. |
| `isEmpty(): bool` | Returns whether there are no rows. |

A `WriteResult` holds the outcome of a write.

| Method | Effect |
|---|---|
| `affectedRows(): int` | Returns the number of rows the statement affected. |
| `lastInsertId(): ?string` | Returns the identifier generated by the statement's insert, or `null` when it generated none. |
| `wasSuccessful(): bool` | Returns whether the statement affected at least one row. |

#### Rows

A `Row` holds one row's values, keyed by column name.

| Method | Effect |
|---|---|
| `get(string $column): mixed` | Returns the column's value, or `null` when the row has no such column. |
| `has(string $column): bool` | Returns whether the row has the column, including when its value is `null`. |
| `isNull(string $column): bool` | Returns whether the row has the column and its value is `null`. |
| `toArray(): array` | Returns every value, keyed by column name. |
| `string(string $column): string` | Returns the value as a string. |
| `int(string $column): int` | Returns the value as an integer. |
| `float(string $column): float` | Returns the value as a float. |
| `bool(string $column): bool` | Returns the value as a boolean. |
| `array(string $column): array` | Returns the value as an array. |

The typed accessors cast the value, and throw `InvalidValueCastException` when it cannot be cast. A missing column, or
a `null` value, cannot be cast to any type.

| Accessor | Accepts |
|---|---|
| `string` | A string, or a number cast to a string. |
| `int` | An integer, or a numeric string cast to an integer. |
| `float` | A float, or an integer or numeric string cast to a float. |
| `bool` | A boolean, or an integer cast to a boolean. The strings `true`, `1` and `yes` are `true`, and `false`, `0` and `no` are `false`. |
| `array` | An array, or a string holding valid JSON, decoded. |

#### Transactions

| Method | Effect |
|---|---|
| `transaction(callable $callback): mixed` | Begins a transaction, calls the callback with the connection, commits, and returns the callback's result. If the callback throws, the transaction is rolled back and the exception rethrown. |
| `beginTransaction(): void` | Begins a transaction. |
| `commit(): void` | Commits the active transaction. |
| `rollback(): void` | Rolls back the active transaction. |
| `isInTransaction(): bool` | Returns whether a transaction is active. |

```php
$connection->transaction(function (Connection $connection) {
    $connection->execute(Update::table('servers')->set(['owner_id' => 2])->where('id', '=', 1));
    $connection->execute(Insert::into('audit')->values(['server_id' => 1, 'action' => 'transfer']));
});
```

Transactions do not nest, and there are no savepoints. Beginning a transaction while one is active throws
`DatabaseException`, and so does a commit or rollback the database refuses.

### Expressions

Every object in the query builder and schema builder implements `Expression`:

| Method | Returns |
|---|---|
| `toSql(): string` | The object's SQL, with a `?` placeholder for each bound value. |
| `getBindings(): array` | The values bound to those placeholders, in the order the placeholders appear. |

Each object is responsible for converting itself to SQL. An expression that contains other expressions builds its
SQL from theirs, and gathers their bound values in the same order, so an expression can be used anywhere another is
accepted and its bound values follow it.

These contracts extend `Expression` and add nothing to it. Each exists to mark what type of expression is required:

| Contract | Marks |
|---|---|
| `Query` | A complete query: `Select`, `Insert`, `Update` or `Delete`. |
| `Schema` | A complete schema statement. |
| `Column` | A column definition. |
| `Index` | An index definition. |

Values supplied to a query are bound, never written into its SQL. The exceptions are a query's limit and offset,
which are written as integers, and the literal values in a schema statement, such as a column's default or comment,
since schema statements have no bound values.

A name written into SQL as an identifier, such as a table, column or index name, is quoted with backticks, with any
backtick inside it doubled. A string written into SQL as a literal is quoted with single quotes, with any single
quote inside it escaped.

### Query builder

| Component | Responsibility |
|---|---|
| `Select` | Builds a `SELECT` query. |
| `Insert` | Builds an `INSERT` or `REPLACE` query. |
| `Update` | Builds an `UPDATE` query. |
| `Delete` | Builds a `DELETE` query. |
| `WhereClause` | A list of conditions, used for `WHERE` and `HAVING`. |
| `JoinClause` | The conditions of one join. |
| `Raw` | Raw SQL and the values bound to it. |
| `Expressions` | Static factories for the built-in expressions. |

Each query is created through a static factory naming its table, and built with fluent methods that return the
query. Clauses shared between queries, such as conditions or ordering, are written once and composed into every
query that supports them, so they behave the same everywhere.

#### Select

```php
$query = Select::from('servers')
    ->columns('servers.id', 'servers.name', Expressions::count('players.id'))
    ->leftJoin('players', 'players.server_id', '=', 'servers.id')
    ->where('servers.status', '=', 'running')
    ->groupBy('servers.id', 'servers.name')
    ->havingRaw('COUNT(players.id) > ?', [0])
    ->orderBy('servers.name')
    ->limit(20)
    ->offset(40);
```

| Method | Effect |
|---|---|
| `from(Expression\|string $table)` | Static. Creates a select from a table, or from a subquery written in parentheses. |
| `columns(Expression\|string ...$columns)` | Sets the columns selected, replacing any set before. With no columns set, `*` is selected. |
| `addColumn(Expression\|string $column)` | Adds a column to those selected. |
| `distinct()` | Selects only distinct rows. |
| `join()`, `leftJoin()`, `rightJoin()`, `crossJoin()` | Add joins, as described in [Joins](#joins). |
| `where()` and its variants | Add conditions to the `WHERE` clause, as described in [Conditions](#conditions). |
| `groupBy(Expression\|string ...$columns)` | Adds columns to group by. |
| `having(Closure\|string $column, ?string $operator = null, mixed $value = null)` | Adds a condition to the `HAVING` clause, joined with `AND`, as `where()` does. |
| `orHaving()`, `havingRaw()`, `orHavingRaw()` | Add conditions to the `HAVING` clause, as `orWhere()`, `whereRaw()` and `orWhereRaw()` do. |
| `orderBy(Expression\|string $column, string $direction = 'asc')` | Adds a column to order by. A direction of `desc`, in any case, orders descending, and any other direction ascending. |
| `limit(int $limit)` | Sets the maximum number of rows. |
| `offset(int $offset)` | Sets the number of rows to skip. |

The clauses are written in this order, whatever order they were added in, and the bound values follow the same order:

```
SELECT [DISTINCT] {columns} FROM {table} {joins} WHERE {conditions} GROUP BY {columns} HAVING {conditions}
ORDER BY {columns} LIMIT {limit} OFFSET {offset}
```

A clause with nothing added to it is left out.

#### Conditions

`Select`, `Update` and `Delete` each have a `WHERE` clause, built with these methods. Each method has a variant
prefixed `or`, which joins the condition with `OR` instead of `AND`.

| Method | Adds |
|---|---|
| `where(Closure\|string $column, ?string $operator = null, mixed $value = null)` | A comparison between the column and a bound value, using the operator. Given a closure instead of a column, a group. |
| `whereNull(string $column)` | `{column} IS NULL`. |
| `whereNotNull(string $column)` | `{column} IS NOT NULL`. |
| `whereIn(string $column, array\|Expression $values)` | `{column} IN (...)`, with a placeholder for each value, or with the expression's SQL as a subquery. |
| `whereNotIn(string $column, array\|Expression $values)` | `{column} NOT IN (...)`, in the same way. |
| `whereRaw(string $sql, array $bindings = [])` | The SQL as given, with its bound values. |
| `whereFullText(array $columns, string $value, string $mode = 'natural')` | A full-text search of the columns for the bound value, as the `match` expression describes. |

The first condition in a clause has no conjunction, and each later one is joined to the one before it.

When a column is given, the operator is required. There is no shorthand that assumes `=`.

| Operator | SQL |
|---|---|
| `=` | `{column} = ?` |
| `!=` | `{column} != ?` |
| `<`, `>`, `<=`, `>=` | `{column} < ?`, and so on |
| `is` | `{column} IS ?` |
| `is null` | `{column} IS NULL`, with no value bound |
| `is not null` | `{column} IS NOT NULL`, with no value bound |
| `in` | `{column} IN (?, ...)`, with a placeholder for each value in an array |
| `not in` | `{column} NOT IN (?, ...)`, in the same way |

Operators are matched regardless of case. Any other operator throws `InvalidExpressionException`. An `IN` or
`NOT IN` condition given an empty array of values throws `InvalidExpressionException`, however it is added.

Given a closure, `where()` and `orWhere()` pass it a new `WhereClause`, to which it adds conditions with the same
methods. The group is written in parentheses and joined to the condition before it. A closure that adds no condition
throws `InvalidExpressionException`.

```php
Select::from('servers')
    ->where('game', '=', 'minecraft')
    ->where(function (WhereClause $clause) {
        $clause->where('status', '=', 'running')->orWhere('status', '=', 'starting');
    });
```

#### Joins

| Method | Adds |
|---|---|
| `join(string $table, Closure\|string $first, ?string $operator = null, ?string $second = null)` | An `INNER JOIN`. |
| `leftJoin(string $table, Closure\|string $first, ?string $operator = null, ?string $second = null)` | A `LEFT JOIN`. |
| `rightJoin(string $table, Closure\|string $first, ?string $operator = null, ?string $second = null)` | A `RIGHT JOIN`. |
| `crossJoin(string $table)` | A `CROSS JOIN`, with no conditions. |

Given two columns and an operator, a join compares the columns. Given a closure, the join passes it the join's
`JoinClause`, to which it adds conditions:

| Method | Adds |
|---|---|
| `on(string $left, string $operator, string $right)` | A comparison between two columns, joined with `AND`. |
| `orOn(string $left, string $operator, string $right)` | A comparison between two columns, joined with `OR`. |
| `where(string $column, string $operator, mixed $value)` | A comparison between a column and a bound value, joined with `AND`. |
| `orWhere(string $column, string $operator, mixed $value)` | A comparison between a column and a bound value, joined with `OR`. |

A join's conditions are written after `ON`. A join with no conditions has no `ON`.

```php
Select::from('servers')->join('players', function (JoinClause $join) {
    $join->on('players.server_id', '=', 'servers.id')->where('players.online', '=', true);
});
```

#### Insert

| Method | Effect |
|---|---|
| `into(string $table)` | Static. Creates an insert into the table. |
| `values(array $values)` | Adds a row, as values keyed by column name. Called more than once, it inserts every row in one query. |
| `ignore()` | Writes `INSERT IGNORE`, so rows that would break a unique key are skipped. |
| `replace()` | Writes `REPLACE INTO`, so rows that would break a unique key replace the rows they clash with. It takes precedence over `ignore()`. |
| `upsert(array $values)` | Adds `ON DUPLICATE KEY UPDATE`, setting each column to its value. A value that is an expression is written as SQL, with its bound values, so a column can be set from itself or another column. It replaces any values set before. |

```php
Insert::into('player_visits')
    ->values(['server_id' => 1, 'player' => 'Ada', 'visits' => 1])
    ->values(['server_id' => 1, 'player' => 'Bob', 'visits' => 1])
    ->upsert(['visits' => Raw::from('visits + 1')]);
```

The first row's columns are the columns of the insert. Every later row must have the same columns, in any order, and
each row's values are bound in the order of the insert's columns. A row with different columns throws
`InvalidExpressionException`, and so does producing the SQL of an insert with no rows.

The bound values are every row's values, then the upsert's.

#### Update

| Method | Effect |
|---|---|
| `table(string $table)` | Static. Creates an update of the table. |
| `set(array $values)` | Sets columns to values, keyed by column name, merged with any set before. A value that is an expression is written as SQL, with its bound values. |
| `where()` and its variants | Add conditions, as described in [Conditions](#conditions). |
| `orderBy(Expression\|string $column, string $direction = 'asc')` | Adds a column to order by, as for a select. |
| `limit(int $limit)` | Sets the maximum number of rows updated. |

```php
Update::table('servers')
    ->set(['status' => 'stopped', 'restarts' => Raw::from('restarts + 1')])
    ->where('id', '=', 1);
```

The clauses are written as `UPDATE {table} SET {values} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`, and the
bound values follow the same order.

#### Delete

| Method | Effect |
|---|---|
| `from(string $table)` | Static. Creates a delete from the table. |
| `where()` and its variants | Add conditions, as described in [Conditions](#conditions). |
| `orderBy(Expression\|string $column, string $direction = 'asc')` | Adds a column to order by, as for a select. |
| `limit(int $limit)` | Sets the maximum number of rows deleted. |

The clauses are written as `DELETE FROM {table} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`, and the bound
values follow the same order.

#### Raw SQL

`Raw::from(string $sql, array $bindings = [])` creates an expression holding the SQL as given and the values bound to
it. It can be executed through a connection, or used wherever an expression is accepted: a selected column, a table,
a value in `set()` or `upsert()`, or a column default. A connection also accepts SQL directly as a string, with its
bound values.

#### Built-in expressions

`Expressions` creates the built-in expressions:

| Method | SQL |
|---|---|
| `count(Expression\|string $column = '*')` | `COUNT({column})` |
| `sum(Expression\|string $column)` | `SUM({column})` |
| `min(Expression\|string $column)` | `MIN({column})` |
| `max(Expression\|string $column)` | `MAX({column})` |
| `avg(Expression\|string $column)` | `AVG({column})` |
| `match(array $columns, string $value)` | `MATCH({columns}) AGAINST(? IN NATURAL LANGUAGE MODE)` |
| `matchBoolean(array $columns, string $value)` | `MATCH({columns}) AGAINST(? IN BOOLEAN MODE)` |
| `whereColumn(string $operator, string $column, mixed $value)` | The comparison for the operator, as described in [Conditions](#conditions). |
| `raw(string $sql, array $bindings)` | The SQL as given, with its bound values. |

An aggregate's column can be an expression, whose SQL is written inside the function and whose bound values follow
it. A full-text search's mode is `natural` or `boolean`; any other mode throws `InvalidExpressionException`.

### Schema builder

| Component | Responsibility |
|---|---|
| `Create` | Static factories for statements that create a table or a database. |
| `Alter` | Static factories for statements that alter a table or a database. |
| `Drop` | Drops a table, database, column, index, foreign key or primary key. |
| `Rename` | Renames a table. |
| `Truncate` | Truncates a table. |
| `Column` | Static factories for column definitions. |
| `Index` | Static factories for index definitions. |

The schema builder is object-first, and falls back to raw SQL where it has no object for what is needed: an
expression as a column default or table option, a generated column's expression, or a whole statement executed
through a connection. Schema statements are executed like any other SQL, and have no bound values.

```php
$connection->execute(
    Create::table('servers', [
        Column::bigInt('id')->unsigned()->autoIncrement(),
        Column::char('ulid', 26)->notNull()->unique(),
        Column::bigInt('owner_id')->unsigned()->notNull(),
        Column::varchar('name', 255)->notNull(),
        Column::enum('status', ServerStatus::class)->default(ServerStatus::Stopped),
        Column::json('settings')->nullable(),
        Column::timestamp('created_at')->defaultCurrentTimestamp(),
    ])
        ->indexes(
            Index::primary('id'),
            Index::foreign('servers_owner_foreign', 'owner_id')->on('users')->references('id')->onDeleteCascade(),
        )
        ->engine('InnoDB')
        ->charset('utf8mb4'),
);
```

#### Tables

`Create::table(string $table, array $columns)` creates a `CREATE TABLE` statement with the columns given.

| Method | Effect |
|---|---|
| `ifNotExists()` | Creates the table only if it does not already exist. |
| `temporary()` | Creates a temporary table. |
| `indexes(Index ...$indexes)` | Sets the table's indexes, replacing any set before. |
| `engine(string $engine)` | Sets the storage engine. |
| `charset(string $charset)` | Sets the character set. |
| `collation(string $collation)` | Sets the collation. |
| `autoIncrement(int $value)` | Sets the value auto-increment starts from. |
| `comment(string $comment)` | Sets the table's comment. |
| `options(Expression ...$options)` | Sets further table options as expressions, replacing any set before. |

The columns and then the indexes are written in parentheses, followed by the table options.

`Alter::table(string $table)` creates an `ALTER TABLE` statement.

| Method | Effect |
|---|---|
| `rename(string $newName)` | Renames the table. |
| `add(Column\|Index ...$new)` | Adds columns and indexes. |
| `modify(Column ...$columns)` | Replaces the definitions of existing columns with those given. |
| `move(string $column, string $newColumn)` | Renames a column. |
| `drop(Drop ...$drops)` | Drops columns, indexes, foreign keys or the primary key. A drop of a table or database throws `InvalidArgumentException`. |

Every change is written into one `ALTER TABLE` statement, in this order whatever order they were added in:

1. Renaming the table.
2. Dropping the primary key.
3. Dropping indexes and foreign keys.
4. Dropping columns.
5. Modifying columns.
6. Renaming columns.
7. Adding columns.
8. Adding indexes.

The order is fixed because several changes can refer to the same thing, such as a column dropped and added again
rather than modified, and a fixed order limits the problems that causes.

`Rename::table(string $table, string $newName)` creates a `RENAME TABLE` statement, and
`Truncate::table(string $table)` a `TRUNCATE TABLE` statement.

#### Databases

`Create::database(string $database)` creates a `CREATE DATABASE` statement, with `ifNotExists()`, `charset()` and
`collation()`. `Alter::database(string $database)` creates an `ALTER DATABASE` statement, with `charset()` and
`collation()`.

#### Drops

| Factory | Drops |
|---|---|
| `Drop::table(string $name)` | A table. |
| `Drop::database(string $name)` | A database. |
| `Drop::column(string $name)` | A column, within `Alter::table()`. |
| `Drop::index(string $name)` | An index, within `Alter::table()`. |
| `Drop::foreignKey(string $name)` | A foreign key, within `Alter::table()`. |
| `Drop::primaryKey()` | The primary key, within `Alter::table()`. |

A drop of a table or a database is a schema statement, executed on its own. The others are only used within
`Alter::table()`.

| Method | Effect |
|---|---|
| `ifExists()` | Drops the table or database only if it exists. On any other drop, it throws `InvalidSchemaException`. |
| `temporary()` | Drops a temporary table. On any other drop, it throws `InvalidSchemaException`. |

#### Columns

`Column` creates column definitions. The factory used fixes the column's type.

| Family | Factories | Sizing |
|---|---|---|
| Integer | `tinyInt()`, `smallInt()`, `mediumInt()`, `int()`, `bigInt()` | An optional display length. |
| Decimal | `decimal()`, `float()`, `double()` | An optional length and number of decimal places. |
| String | `char()`, `varchar()`, `text()`, `tinytext()`, `mediumtext()`, `longtext()` | A length, on `char`, `varchar` and `text` only. |
| Binary | `binary()`, `varbinary()`, `tinyblob()`, `blob()`, `mediumblob()`, `longblob()` | A length, on `binary`, `varbinary` and `blob` only. |
| Enumerated | `enum()`, `set()` | The permitted values, as an array of strings or the class of a backed enum, whose cases' values are used. |
| Temporal | `date()`, `datetime()`, `timestamp()`, `time()`, `year()` | A fractional seconds precision, on `datetime`, `timestamp` and `time` only. |
| Boolean | `boolean()` | None. |
| JSON | `json()` | None. |

Each factory takes the column's name first, followed by its sizing where the family has any.

Every column has these modifiers:

| Modifier | Effect |
|---|---|
| `nullable()` | Writes `NULL`, cancelling `notNull()`. |
| `notNull()` | Writes `NOT NULL`, cancelling `nullable()`. |
| `default(mixed $value)` | Sets the column's default value. |
| `unique()` | Writes `UNIQUE`. |
| `comment(string $comment)` | Sets the column's comment. |
| `after(string $column)` | Places the column after another column, when it is added to an existing table. |
| `first()` | Places the column first, when it is added to an existing table. |
| `generated(Query $query)` | Makes the column generated, writing `GENERATED ALWAYS AS ({sql})`. A generated column has no null modifier or default written. |
| `virtual()` | Makes a generated column virtual. On a column that is not generated, it throws `InvalidSchemaException`. |
| `stored()` | Makes a generated column stored. On a column that is not generated, it throws `InvalidSchemaException`. |

Some families of column have further modifiers:

| Modifier | Columns | Effect |
|---|---|---|
| `unsigned()` | Integer, decimal | Writes `UNSIGNED`. |
| `autoIncrement()` | Integer | Writes `AUTO_INCREMENT`. |
| `length(int $length)` | Integer, decimal, string, binary | Sets the length. |
| `charset(string $charset)`, `collation(string $collation)` | String, enumerated | Set the character set and collation. |
| `precision(int $precision)` | Temporal | Sets the fractional seconds precision. |
| `defaultCurrentTimestamp()` | Temporal | Sets the default to `CURRENT_TIMESTAMP`. |
| `onUpdateCurrentTimestamp()` | Temporal | Writes `ON UPDATE CURRENT_TIMESTAMP`. |

A modifier used on a column type it does not apply to throws `InvalidSchemaException`, such as
`length()` on a `tinytext` column. `defaultCurrentTimestamp()` and `onUpdateCurrentTimestamp()` apply to `timestamp`
and `datetime` only.

A default is written according to its value:

| Value | Written as |
|---|---|
| `null` | `NULL` |
| A string | A string literal. |
| An integer or float | The number. |
| A boolean | `1` or `0`. |
| A backed enum case | Its value, as a string literal or a number. |
| An array or `JsonSerializable` | Its JSON encoding. |
| An expression | Its SQL, such as `CURRENT_TIMESTAMP`. |

An enumerated column needs at least one value, and a class given for its values must be a backed enum. These are
checked with assertions, as mistakes in the code that defines a schema.

#### Indexes

`Index` creates index definitions.

| Factory | Writes |
|---|---|
| `Index::primary(string ...$columns)` | `PRIMARY KEY ({columns})` |
| `Index::unique(string $name, string ...$columns)` | `UNIQUE INDEX {name} ({columns})` |
| `Index::index(string $name, string ...$columns)` | `INDEX {name} ({columns})` |
| `Index::fulltext(string $name, string ...$columns)` | `FULLTEXT INDEX {name} ({columns})` |
| `Index::foreign(string $name, string ...$columns)` | `CONSTRAINT {name} FOREIGN KEY ({columns}) REFERENCES {table} ({references})` |

A foreign key is completed with these methods:

| Method | Effect |
|---|---|
| `on(string $table)` | Sets the referenced table. Required. |
| `references(string ...$columns)` | Sets the referenced columns. Required. |
| `onDeleteCascade()`, `onDeleteSetNull()`, `onDeleteRestrict()`, `onDeleteSetDefault()`, `onDeleteNoAction()` | Set the `ON DELETE` action. |
| `onUpdateCascade()`, `onUpdateSetNull()`, `onUpdateRestrict()`, `onUpdateSetDefault()`, `onUpdateNoAction()` | Set the `ON UPDATE` action. |

Producing the SQL of a foreign key with no referenced table or no referenced columns throws `InvalidSchemaException`.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `DatabaseException` | `RuntimeException` | The configured driver is not supported, a MySQL connection has neither a host nor a socket, a transaction cannot be begun, committed or rolled back, or `DatabaseResolver` is given a dependency without the `Database` attribute or not typed as `Connection`. |
| `ConnectionException` | `DatabaseException` | No configuration exists for the requested connection name, or the connection cannot be established. It carries the connection name, and the underlying error as its previous exception. |
| `QueryException` | `DatabaseException` | A statement cannot be prepared or executed. It carries the SQL and bound values, through `getSql()` and `getBindings()`, and the underlying error as its previous exception. |
| `InvalidExpressionException` | `InvalidArgumentException` | An operator or full-text mode is not recognised, a group adds no condition, an `IN` condition has no values, or an insert has no rows or rows with different columns. |
| `InvalidSchemaException` | `LogicException` | A modifier is used on a type it does not apply to, `virtual()` or `stored()` is used on a column that is not generated, or a foreign key has no referenced table or columns. |
| `InvalidValueCastException` | `InvalidArgumentException` | A row's typed accessor cannot cast a value. |

### Out of scope

- **Migrations.** Running a series of migrations against the database, rolling them back, and recording which have
  run are designed separately.
- **Further built-in expressions.** Pattern matching (`LIKE`, `REGEXP`), ranges (`BETWEEN`), `GROUP_CONCAT`, and JSON,
  date and time, string and numeric functions are added as they are needed. Until then, raw SQL covers them.
- **Drivers other than MySQL.** They can be supported inside the component, but none is exposed.
- **Nested transactions and savepoints.**

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- building on PDO rather than a database abstraction layer, in
  [ADR-0008](../adr/0008-database-access-is-built-directly-on-pdo.md)
- assuming MySQL or MariaDB, in [ADR-0007](../adr/0007-mysql-and-mariadb-are-the-default-database.md)
- injecting connections through a resolvable attribute and resolver, in
  [ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md)
- holding the component's configuration in typed objects, in
  [ADR-0005](../adr/0005-configuration-is-held-in-typed-objects.md)

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. No database component exists before this change.

## Open questions

## Changelog

## Sources

- Issue [#26], Engine - Database, 2026-03-29 and not edited since: the motivation, multiple simultaneous connections,
  the primary connection, an object-first abstraction of SQL queries and a schema migration tool. This is `created`.
- Issue [#27], Engine - Database - Connections, 2026-03-29 and not edited since: multiple connections, hidden driver
  support and the primary connection.
- Issue [#28], Engine - Database - Schema, 2026-03-29 and not edited since: an object-first schema abstraction that
  falls back to raw SQL, for creating and then updating the database. Its migration design is not part of this
  document.
- Issue [#29], Engine - Database - Query Builder, 2026-03-29: expressions that convert themselves to SQL and bound
  values, `Query` as a marker, the builders being distinct from connections, the queries and clauses each supports,
  raw queries, and the built-in expressions. It was edited on 2026-03-30, a formatting fix, and on 2026-04-17, when
  the implemented expressions were ticked. The ticked expressions are the ones this document includes.
- PR [#30], feat(database): Add the database component, merged 2026-04-17 and squashed as [560e9ab]: the
  implementation, whose merge is `decided`. The configuration, the factory, execution, results, transactions, the
  attribute and resolver, the builders' methods and the SQL they write, and the exceptions are described from
  [`src/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/src/Database),
  [`src/Values` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/src/Values),
  [`tests/Unit/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/tests/Unit/Database) and
  [`tests/Integration/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/tests/Integration/Database).
  None of #26 to #29 designs execution beyond queries being passed to a connection.
- Commit [44f735a], "Add a base abstract config object", 2026-03-29, on the branch of PR [#30]: configuration objects
  restoring themselves by passing an exported array to their constructor as named arguments.
- Commit [745603d], "Require operator in query builder where methods", 2026-03-30, on the branch of PR [#30]: the
  operator being required, with the `=` shorthand removed. No reason is given.
- Commits [1f8656e], "Add schema abstraction for DDL generation", 2026-03-31, [7551b24], "Remove all schema and
  migration functionality", 2026-04-01, and [017ace2], "Add schema DDL builder and kill escaped mutants", 2026-04-17,
  on the branch of PR [#30]: the schema builder first written with a `Blueprint` for alter operations and a `Table`
  for creating, altering and dropping, removed with the initial migration work, and rewritten as the statement
  objects described here. No reason is given for the change of shape.
- The fixed order of changes in an `ALTER TABLE` statement and its reason, and `after()` and `first()` placing a
  column: the docblocks of `AlterTable` and the `Column` contract at [560e9ab].
- The error mode being forced, one `persistent` flag applying to every connection, a cursor fetching rows one at a
  time, `lastInsertId()` reporting only the statement's own insert, transactions not nesting, every row of an insert
  having the same columns with at least one row required, empty `IN` conditions and unrecognised full-text modes
  throwing, a drop of a table or database being a schema statement, the primary key being dropped without a name,
  and this document covering the whole component: first written down on 2026-09-14.
- Identifiers and literals being quoted and escaped consistently: issue [#41], Engine - Query Builder - Identifier
  quoting and escaping, 2026-08-07, taken from what followed rather than anticipated.
- The reason for forcing the error mode, that overriding it would silently disable the exceptions a connection relies
  on: issue [#48], Engine - Database - Connection configuration fixes, 2026-08-08, taken from what followed rather
  than anticipated.
- The code at [560e9ab] differs from this design:
  - configured options can override the error mode, and the `persistent` flag is never applied
  - the MySQL driver buffers the whole result before a cursor fetches its first row
  - `lastInsertId()` reports the connection's last insert after an `UPDATE` or `DELETE`
  - only `Update`'s columns and the schema builder's names are quoted, no quoting is escaped, and comments, string
    defaults and JSON defaults are written unescaped or unquoted, as issue [#41] records
  - an insert binds each row's values in that row's own order, and fails with an error when it has no rows, as issue
    [#42], Engine - Query Builder - Binding and clause correctness, 2026-08-07, records
  - `where()` with the `in` operator accepts an empty array, and an unrecognised full-text mode becomes natural
    language
  - `Drop` is not a schema statement, and `Drop::primaryKey()` takes a name
  - `after()` and `first()` are never written
  - nothing checks that a later row's columns match the first row's, so no `InvalidExpressionException` is thrown
    for a row whose columns differ
  - a select gathers its table subquery's bound values before its columns', while writing the table after them, so
    the placeholders and the values can disagree
  - the temporal column factories take no precision argument, and a precision is reachable only through
    `precision()`

[#26]: https://github.com/thegamepanel/panel/issues/26
[#27]: https://github.com/thegamepanel/panel/issues/27
[#28]: https://github.com/thegamepanel/panel/issues/28
[#29]: https://github.com/thegamepanel/panel/issues/29
[#30]: https://github.com/thegamepanel/panel/pull/30
[#41]: https://github.com/thegamepanel/panel/issues/41
[#42]: https://github.com/thegamepanel/panel/issues/42
[#48]: https://github.com/thegamepanel/panel/issues/48
[560e9ab]: https://github.com/thegamepanel/panel/commit/560e9ab
[44f735a]: https://github.com/thegamepanel/panel/commit/44f735a
[745603d]: https://github.com/thegamepanel/panel/commit/745603d
[1f8656e]: https://github.com/thegamepanel/panel/commit/1f8656e
[7551b24]: https://github.com/thegamepanel/panel/commit/7551b24
[017ace2]: https://github.com/thegamepanel/panel/commit/017ace2
