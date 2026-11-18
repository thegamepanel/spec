---
id: RFC-0003
title: Database component
status: accepted
created: 2026-03-29
decided: 2026-04-17
backfilled: 2026-11-18
depends: [ADR-0002, ADR-0005, ADR-0007, ADR-0008, RFC-0001, RFC-0002]
updates: []
obsoletes: []
---

# RFC-0003: Database component

## Abstract

I will give the [engine](../GLOSSARY.md#engine) a database component built directly on PDO. Any number of named
[connection](../GLOSSARY.md#connection)s will be configured, one of them the
[primary connection](../GLOSSARY.md#primary-connection) used whenever no name is given, and a connection will be
injected into a parameter with the `Database` attribute. Queries and schema changes will be built as objects that
produce their own SQL and bound values, and a connection will run them, returning the rows in full, a
[cursor](../GLOSSARY.md#cursor) over them, or the outcome of a write.

## Motivation

The panel needs a simple database abstraction, so that queries can be built in code without writing SQL by hand, and
so that the database can be created and then changed as the panel and its [module](../GLOSSARY.md#module)s change.

The panel also needs to be able to connect to more than one database. The core and the first-party components and
modules only ever use one, but supporting several takes little extra effort and leaves room for a future case that
needs it.

## Proposal

### How the parts fit together

The component will have four parts. [Connection](../GLOSSARY.md#connection)s will be configured, created and injected. A
connection will execute SQL, return its results and manage transactions. The query builder will build `SELECT`,
`INSERT`, `UPDATE` and `DELETE` [queries](../GLOSSARY.md#query) as objects, and the schema builder will build, as
objects, the statements that create, alter and drop databases, tables, columns and indexes.

The two builders will be separate from connections. A builder will produce SQL and the values bound to it, and know
nothing of the connection it will run against. A connection will accept SQL and bound values, whether written as a
string or produced by a builder. The point where the two meet will be the [expression](../GLOSSARY.md#expression),
described below: every builder object will be one, and a connection will accept any of them.

### Configuring connections

The component's configuration will be two [configuration object](../GLOSSARY.md#configuration-object)s, as
[RFC-0002](0002-configuration-objects.md) and [ADR-0005](../adr/0005-typed-configuration.md)
describe: `DatabaseConfig` for the component and a `ConnectionConfig` for each [connection](../GLOSSARY.md#connection).
Both will be created through a static `make()`, and both will restore themselves from an exported array by passing its
entries to their constructor as named arguments.

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

`DatabaseConfig` will hold the name of the [primary connection](../GLOSSARY.md#primary-connection), every connection's
configuration keyed by its connection name, and a single `persistent` flag. A database configuration must name a primary
connection, configure at least one connection, and configure one under the primary name. The `persistent` flag will
apply to every connection at once, never to one alone, and will default to `false`.

A `ConnectionConfig` will hold where to connect, which database to use, the username and password to connect with,
and PDO options of its own. A connection will be reached either through a Unix socket or through a host and port, with
the port defaulting to `3306` when none is given. When both a socket and a host are given, the socket will be used.

Every connection's configuration will also carry its driver, the type of database it talks to. The driver will
always be `mysql` and will not be configurable. It will be carried so that support for other database types can be
added inside the component later without restructuring it, while only MySQL is exposed, as
[ADR-0007](../adr/0007-mysql-and-mariadb.md) decides.

### Opening a connection

A `ConnectionFactory` will be constructed with the `DatabaseConfig`, and will hand out
[connection](../GLOSSARY.md#connection)s through `make(?string $name = null): Connection`:

```php
$primary = $factory->make();
$replica = $factory->make('replica');
```

With no name, `make()` will use the name of the [primary connection](../GLOSSARY.md#primary-connection). If a connection
has already been created under the name, that connection will be returned. Otherwise the factory will look up the
configuration for the name, throwing `ConnectionException` when there is none, and create a PDO connection from it with
its DSN, credentials and options, throwing `ConnectionException` when it cannot connect. The new connection will be kept
under its name for as long as the factory lives, and returned. `ConnectionException` will carry the connection name, and
the underlying error as its previous exception.

For MySQL, the DSN will be `mysql:unix_socket={socket};dbname={database}` when a socket is configured, and
`mysql:host={host};port={port};dbname={database}` when only a host is, with the port defaulting to `3306`. A
connection configured with neither a socket nor a host will throw `DatabaseException`, and so will a driver other
than MySQL.

Every connection will be created with these PDO options:

| Option | Value | Configurable |
|---|---|---|
| `PDO::ATTR_ERRMODE` | `PDO::ERRMODE_EXCEPTION` | No, always forced |
| `PDO::ATTR_DEFAULT_FETCH_MODE` | `PDO::FETCH_ASSOC` | Yes |
| `PDO::ATTR_EMULATE_PREPARES` | `false` | Yes |
| `PDO::ATTR_STRINGIFY_FETCHES` | `false` | Yes |

A connection's configured options will be applied over these defaults, so they can change any default that is
configurable and add options of their own. The error mode will always be forced to throw exceptions, because
everything built on a connection relies on errors being thrown. When `persistent` is set, every connection will also
be created with `PDO::ATTR_PERSISTENT`, so that PDO keeps it open and reuses it rather than closing it.

### Injecting a connection

A [connection](../GLOSSARY.md#connection) will be injected into a parameter typed `Connection` by placing the `Database`
attribute on it, with an optional connection name:

```php
public function __construct(
    #[Database] private Connection $primary,
    #[Database('replica')] private Connection $replica,
) {}
```

`Database` will be a [resolvable attribute](../GLOSSARY.md#resolvable-attribute), paired with `DatabaseResolver` in the
[resolver](../GLOSSARY.md#resolver) tier of
[ADR-0002](../adr/0002-parameter-attributes.md), using the mechanism of
[RFC-0001](0001-dependency-injection-container.md). The resolver will itself be resolved through the container, which
will give it the `ConnectionFactory`, and it will return whatever `make()` returns for the attribute's name, so a
parameter with no name will receive the [primary connection](../GLOSSARY.md#primary-connection). The parameter must be
typed exactly `Connection`. The resolver will throw `DatabaseException` when it is given a dependency that does not
carry the `Database` attribute, or one that is not typed `Connection`.

### Running SQL

A `Connection` will be a named connection wrapping its PDO connection. It will run SQL in three ways, which differ in
what they return:

- `query(Expression|string $query, array $bindings = []): Result` will run a [query](../GLOSSARY.md#query) that returns
  rows, and return them as a [result](../GLOSSARY.md#result).
- `execute(Expression|string $query, array $bindings = []): WriteResult` will run a statement that writes, and return
  its outcome.
- `stream(Expression|string $query, array $bindings = []): Cursor` will run a query that returns rows, and return a
  [cursor](../GLOSSARY.md#cursor) over them.

Each will accept SQL as a string together with the values bound to it, or an [expression](../GLOSSARY.md#expression),
whose SQL and bound values will be taken from its `toSql()` and `getBindings()`. Bound values passed alongside an
expression will be ignored.

Every statement will be prepared, then executed with its bound values. Booleans among the bound values will first be
converted to the integers `1` and `0`, because MySQL treats booleans as integers. A statement that cannot be prepared
or executed will throw `QueryException`, which will carry its SQL and bound values, through `getSql()` and
`getBindings()`, and the underlying error as its previous exception.

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

Whatever is returned, a `Result`, a `Cursor` or a `WriteResult`, will carry the bound values it was executed with, as
`bindings`.

### Reading results

A [result](../GLOSSARY.md#result) will fetch every row the first time any row is read, and keep them. It will give the
first row, or `null` when there are none, every row at once, each row in turn to a callback, the number of rows, and
whether there are none.

A [cursor](../GLOSSARY.md#cursor) will fetch rows one at a time as they are iterated, so that a large result can be
processed without every row being held in memory at once. Its rows will be iterable only once, either by passing each
row to a callback or through a generator from `rows()`. It will also give the number of rows and whether there are none.

A `WriteResult` will hold the outcome of a write: the number of rows the statement affected, the identifier generated
by the statement's own insert, and whether it succeeded. `lastInsertId()` will report only an identifier the statement
itself generated, and return `null` when it generated none. `wasSuccessful()` will be true when the statement affected
at least one row.

A [row](../GLOSSARY.md#row) will hold one row's values keyed by column name. `get()` will return a column's value, or
`null` when the row has no such column. `has()` will say whether the row has the column, including when its value is
`null`, and `isNull()` whether it has the column and the value is `null`. `toArray()` will return every value keyed by
column name.

The row's typed accessors, `string()`, `int()`, `float()`, `bool()` and `array()`, will cast a column's value, and
throw `InvalidValueCastException` when it cannot be cast. A missing column, or a `null` value, will not be castable to
any type. Each will accept:

| Accessor | Accepts |
|---|---|
| `string` | A string, or a number cast to a string. |
| `int` | An integer, or a numeric string cast to an integer. |
| `float` | A float, or an integer or numeric string cast to a float. |
| `bool` | A boolean, or an integer cast to a boolean. The strings `true`, `1` and `yes` are `true`, and `false`, `0` and `no` are `false`. |
| `array` | An array, or a string holding valid JSON, decoded. |

### Transactions

`transaction(callable $callback): mixed` will begin a transaction, call the callback with the
[connection](../GLOSSARY.md#connection), commit, and return what the callback returned. If the callback throws, the
transaction will be rolled back and the exception rethrown.

```php
$connection->transaction(function (Connection $connection) {
    $connection->execute(Update::table('servers')->set(['owner_id' => 2])->where('id', '=', 1));
    $connection->execute(Insert::into('audit')->values(['server_id' => 1, 'action' => 'transfer']));
});
```

A transaction will also be manageable by hand, with `beginTransaction()`, `commit()` and `rollback()`, and
`isInTransaction()` will say whether one is active. Transactions will not nest, and there will be no savepoints.
Beginning a transaction while one is active will throw `DatabaseException`, and so will a commit or a rollback that
the database refuses.

### Expressions

Every object in the query builder and the schema builder will implement `Expression`, which will have two methods.
`toSql(): string` will return the object's SQL, with a `?` placeholder for each bound value, and
`getBindings(): array` will return the values bound to those placeholders, in the order the placeholders appear.

Each object will be responsible for turning itself into SQL. An [expression](../GLOSSARY.md#expression) that contains
others will build its SQL from theirs and gather their bound values in the same order, so an expression can be used
anywhere another is accepted and its bound values will follow it.

Four contracts will extend `Expression` without adding anything to it, to mark what type of expression a method
requires: `Query` for a complete [query](../GLOSSARY.md#query) (`Select`, `Insert`, `Update` or `Delete`), `Schema` for
a complete [schema statement](../GLOSSARY.md#schema-statement), `Column` for a column definition and `Index` for an
index definition.

Values supplied to a query will always be bound, never written into its SQL. There will be two exceptions: a query's
limit and offset, which will be written as integers, and the literal values in a schema statement, such as a column's
default or comment, since schema statements have no bound values.

A name written into SQL as an identifier, such as a table, column or index name, will be quoted with backticks, with
any backtick inside it doubled. A string written into SQL as a literal will be quoted with single quotes, with any
single quote inside it escaped.

### Building queries

Each [query](../GLOSSARY.md#query) will be created through a static factory that names its table, and built up with
fluent methods that return the query. A query will accumulate its clauses in whatever order they are added, and render
them in a fixed order of its own, with the bound values following that same order. A clause with nothing added to it
will be left out. A clause that several queries share, such as their [condition](../GLOSSARY.md#condition)s or their
ordering, will be written once and composed into every query that supports it, so it behaves the same everywhere.

#### Selecting

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

A select will render its clauses in this order:

```
SELECT [DISTINCT] {columns} FROM {table} {joins} WHERE {conditions} GROUP BY {columns} HAVING {conditions}
ORDER BY {columns} LIMIT {limit} OFFSET {offset}
```

It will select from a table, or from a subquery given as an [expression](../GLOSSARY.md#expression) and written in
parentheses. It can be made to select only distinct rows, and with no columns set it will select `*`. A column can be a
name or an expression, and setting the columns will replace any set before, where adding one will keep them. Grouping
columns will accumulate. The `HAVING` clause will take the same [condition](../GLOSSARY.md#condition)s as `WHERE`,
through `having()`, `orHaving()`, `havingRaw()` and `orHavingRaw()`. An ordering will be ascending by default: a
direction of `desc`, in any case, will order descending, and any other direction ascending. The limit and offset will be
written as integers, as [Expressions](#expressions) describes.

#### Conditions

`Select`, `Update` and `Delete` will each have a `WHERE` clause, built from [condition](../GLOSSARY.md#condition)s. The
first condition in a clause will have no conjunction, and each later one will be joined to the one before it with `AND`,
or with `OR` when it is added through the method's `or` variant, such as `orWhere()` or `orWhereNull()`. Every method
that adds a condition will have one.

`where(Closure|string $column, ?string $operator = null, mixed $value = null)` will compare a column with a bound
value using an operator. When a column is given, the operator will be required: there will be no shorthand that
assumes `=`. The operators will be matched regardless of case, and any other operator will throw
`InvalidExpressionException`:

| Operator | SQL |
|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=` | `{column} = ?`, and so on |
| `is` | `{column} IS ?` |
| `is null` | `{column} IS NULL`, with no value bound |
| `is not null` | `{column} IS NOT NULL`, with no value bound |
| `in` | `{column} IN (?, ...)`, with a placeholder for each value in an array |
| `not in` | `{column} NOT IN (?, ...)`, in the same way |

The other methods will be shorthands for particular conditions. `whereNull()` and `whereNotNull()` will add `IS NULL`
and `IS NOT NULL`. `whereIn()` and `whereNotIn()` will add `IN (...)` and `NOT IN (...)`, given either an array of
values, with a placeholder for each, or an [expression](../GLOSSARY.md#expression) whose SQL is written as a subquery.
An `IN` or `NOT IN` condition given an empty array of values will throw `InvalidExpressionException`, however it is
added. `whereRaw(string $sql, array $bindings = [])` will add SQL as given, with its bound values.
`whereFullText(array $columns, string $value, string $mode = 'natural')` will add a full-text search of the columns for
a bound value, as [Built-in expressions](#built-in-expressions) describes.

Given a closure in place of a column, `where()` and `orWhere()` will pass it a new `WhereClause`, to which it adds
conditions with the same methods. The group will be written in parentheses and joined to the condition before it. A
closure that adds no condition will throw `InvalidExpressionException`.

```php
Select::from('servers')
    ->where('game', '=', 'minecraft')
    ->where(function (WhereClause $clause) {
        $clause->where('status', '=', 'running')->orWhere('status', '=', 'starting');
    });
```

#### Joins

A select will be joined to another table with `join()` for an `INNER JOIN`, `leftJoin()` for a `LEFT JOIN` and
`rightJoin()` for a `RIGHT JOIN`. Each will take the table, then either two columns and the operator comparing them, or
a closure. Given a closure, the join will pass it the join's `JoinClause`, to which it adds
[condition](../GLOSSARY.md#condition)s. `on()` and `orOn()` will compare two columns, and `where()` and `orWhere()` will
compare a column with a bound value, joined with `AND` and `OR` respectively. `crossJoin()` will take only the table,
and add a `CROSS JOIN` with no conditions.

A join's conditions will be written after `ON`, and a join with no conditions will have no `ON`.

```php
Select::from('servers')->join('players', function (JoinClause $join) {
    $join->on('players.server_id', '=', 'servers.id')->where('players.online', '=', true);
});
```

#### Inserting

`Insert::into()` will create an insert into a table, and `values()` will add a row to it, as values keyed by column
name. Called more than once, it will insert every row in one [query](../GLOSSARY.md#query).

```php
Insert::into('player_visits')
    ->values(['server_id' => 1, 'player' => 'Ada', 'visits' => 1])
    ->values(['server_id' => 1, 'player' => 'Bob', 'visits' => 1])
    ->upsert(['visits' => Raw::from('visits + 1')]);
```

The first row's columns will be the insert's columns. Every later row must have the same columns, in any order, and
each row's values will be bound in the order of the insert's columns. A row whose columns differ will throw
`InvalidExpressionException`, and so will producing the SQL of an insert with no rows.

Three methods will change what happens to a row that would break a unique key. `ignore()` will write `INSERT IGNORE`, so
the row is skipped. `replace()` will write `REPLACE INTO`, so the row replaces the one it clashes with; it will take
precedence over `ignore()`. `upsert()` will add `ON DUPLICATE KEY UPDATE`, setting each column given to its value, and
replace any values given to it before. A value that is an [expression](../GLOSSARY.md#expression) will be written as
SQL, with its bound values, so a column can be set from itself or from another column. The bound values will be every
row's values, then the upsert's.

#### Updating and deleting

`Update::table()` will create an update of a table. `set()` will set columns to values, keyed by column name and merged
with any set before, and a value that is an [expression](../GLOSSARY.md#expression) will be written as SQL, with its
bound values. An update will take [condition](../GLOSSARY.md#condition)s as a select does, and can be ordered as a
select is, with `limit()` setting the maximum number of rows updated. It will render as
`UPDATE {table} SET {values} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`.

```php
Update::table('servers')
    ->set(['status' => 'stopped', 'restarts' => Raw::from('restarts + 1')])
    ->where('id', '=', 1);
```

`Delete::from()` will create a delete from a table, with conditions, ordering and a limit on the rows deleted in the
same way. It will render as `DELETE FROM {table} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`.

#### Raw SQL

`Raw::from(string $sql, array $bindings = [])` will create an [expression](../GLOSSARY.md#expression) holding SQL as
given and the values bound to it. It can be run through a [connection](../GLOSSARY.md#connection), or used wherever an
expression is accepted, such as a selected column, a table, a value in `set()` or `upsert()`, or a column default. A
connection will also accept SQL directly as a string, with its bound values.

#### Built-in expressions

The built-in [expression](../GLOSSARY.md#expression)s will be the SQL functions a query most often needs, created
through `Expressions`: aggregates, full-text searches, comparisons and raw SQL.

An aggregate will wrap a column in `COUNT`, `SUM`, `MIN`, `MAX` or `AVG`, with a count's column defaulting to `*`.
The column can itself be an expression, whose SQL will be written inside the function and whose bound values will
follow it.

A full-text search will write `MATCH({columns}) AGAINST(? IN NATURAL LANGUAGE MODE)`, or `IN BOOLEAN MODE` for a
boolean search, with the searched-for value bound. Its mode will be `natural` or `boolean`, and any other mode will
throw `InvalidExpressionException`.

A comparison will write the SQL for an operator, as [Conditions](#conditions) describes, and raw SQL will be written as
given, with its bound values.

### Building the schema

The schema builder will be object-first, and fall back to raw SQL where it has no object for what is needed: an
[expression](../GLOSSARY.md#expression) as a column default or a table option, a generated column's expression, or a
whole statement run through a [connection](../GLOSSARY.md#connection). A
[schema statement](../GLOSSARY.md#schema-statement) will be run like any other SQL, and will have no bound values.

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

#### Creating and changing tables

A `CREATE TABLE` statement, from `Create::table(string $table, array $columns)`, will hold the table's columns, its
indexes and its options, and render the columns and then the indexes in parentheses, followed by the options. It can
be made to create the table only if it does not already exist, or to create a temporary table. Setting the indexes
will replace any set before. The options will be the storage engine, character set, collation, the value
auto-increment starts from and the table's comment, and further options can be given as expressions, which will also
replace any given before.

An `ALTER TABLE` statement, from `Alter::table(string $table)`, will collect changes to one table in any order and
render them as a single statement in a fixed order:

1. Renaming the table.
2. Dropping the primary key.
3. Dropping indexes and foreign keys.
4. Dropping columns.
5. Modifying columns.
6. Renaming columns.
7. Adding columns.
8. Adding indexes.

The order will be fixed because several changes can refer to the same thing, such as a column dropped and added again
rather than modified, and a fixed order limits the problems that causes. Modifying a column will replace its
definition with the one given, and a column will be renamed with `move(string $column, string $newColumn)`. An alter
will drop only what lives inside a table, so given a drop of a table or a database, it will throw
`InvalidArgumentException`.

`Rename::table(string $table, string $newName)` will create a `RENAME TABLE` statement, and
`Truncate::table(string $table)` a `TRUNCATE TABLE` statement.

#### Databases

`Create::database(string $database)` will create a `CREATE DATABASE` statement, which can be made conditional on the
database not existing and given a character set and collation. `Alter::database(string $database)` will create an
`ALTER DATABASE` statement that changes the character set and collation.

#### Dropping

A drop, from `Drop`, will be a [schema statement](../GLOSSARY.md#schema-statement) of its own when it drops a table or a
database, run on its own like any other. A drop of a column, an index, a foreign key or the primary key will instead
live inside an alter of the table, and be used nowhere else. Each drop will take the name of what it drops, apart from
the primary key, which will be dropped without a name.

Only a drop of a table or a database can be made conditional on it existing, and only a drop of a table can be made to
drop a temporary table. Either asked of any other drop will throw `InvalidSchemaException`.

#### Columns

A column's factory, such as `Column::bigInt()` or `Column::varchar()`, will fix its type. Each factory will take the
column's name first, followed by its sizing where the family has any. The type's family will decide that sizing, and
which modifiers beyond the common ones apply:

| Family | Types | Sizing | Further modifiers |
|---|---|---|---|
| Integer | `tinyInt`, `smallInt`, `mediumInt`, `int`, `bigInt` | An optional display length. | `unsigned()`, `autoIncrement()`, `length()` |
| Decimal | `decimal`, `float`, `double` | An optional length and number of decimal places. | `unsigned()`, `length()` |
| String | `char`, `varchar`, `text`, `tinytext`, `mediumtext`, `longtext` | A length, on `char`, `varchar` and `text` only. | `length()`, `charset()`, `collation()` |
| Binary | `binary`, `varbinary`, `tinyblob`, `blob`, `mediumblob`, `longblob` | A length, on `binary`, `varbinary` and `blob` only. | `length()` |
| Enumerated | `enum`, `set` | The permitted values, as an array of strings or the class of a backed enum, whose cases' values are used. | `charset()`, `collation()` |
| Temporal | `date`, `datetime`, `timestamp`, `time`, `year` | A fractional seconds precision, on `datetime`, `timestamp` and `time` only. | `precision()`, `defaultCurrentTimestamp()`, `onUpdateCurrentTimestamp()` |
| Boolean | `boolean` | None. | None. |
| JSON | `json` | None. | None. |

Some families of column will have further modifiers, as the table shows. `unsigned()` will write `UNSIGNED`,
`autoIncrement()` will write `AUTO_INCREMENT`, `length()` and `precision()` will set the sizing, and
`defaultCurrentTimestamp()` and `onUpdateCurrentTimestamp()` will set the default to `CURRENT_TIMESTAMP` and write
`ON UPDATE CURRENT_TIMESTAMP`, on `timestamp` and `datetime` columns only. A modifier used on a column type it does
not apply to, such as `length()` on a `tinytext` column, will throw `InvalidSchemaException`.

An enumerated column needs at least one value, and a class given for its values must be a backed enum. Both will be
checked with assertions, as mistakes in the code that defines a schema.

Every column will share the common modifiers. It can be made nullable, writing `NULL`, or not null, writing
`NOT NULL`, and each will cancel the other. It can be given a default and a comment, and made unique, writing
`UNIQUE`. When it is added to an existing table, `after(string $column)` will place it after another column and
`first()` will place it first.

A column can also be made generated with `generated(Query $query)`, writing `GENERATED ALWAYS AS ({sql})`, and a
generated column will have no null modifier or default written. A generated column can be made virtual or stored, and
asking either of a column that is not generated will throw `InvalidSchemaException`.

A default will be written according to its value:

| Value | Written as |
|---|---|
| `null` | `NULL` |
| A string | A string literal. |
| An integer or float | The number. |
| A boolean | `1` or `0`. |
| A backed enum case | Its value, as a string literal or a number. |
| An array or `JsonSerializable` | Its JSON encoding. |
| An [expression](../GLOSSARY.md#expression) | Its SQL, such as `CURRENT_TIMESTAMP`. |

#### Indexes

`Index` will create index definitions. A primary key, from `Index::primary(string ...$columns)`, will take only its
columns and write `PRIMARY KEY ({columns})`. A unique, plain or full-text index will take a name and then its columns,
and write `UNIQUE INDEX`, `INDEX` or `FULLTEXT INDEX` with that name.

A foreign key, from `Index::foreign(string $name, string ...$columns)`, will write
`CONSTRAINT {name} FOREIGN KEY ({columns}) REFERENCES {table} ({references})`. The referenced table, set with `on()`,
and the referenced columns, set with `references()`, will both be required, and producing the SQL of a foreign key
that lacks either will throw `InvalidSchemaException`. A foreign key can take an action for `ON DELETE` and for
`ON UPDATE`: cascade, set null, restrict, set default or no action, each through a method named for the clause and
the action, such as `onDeleteCascade()` or `onUpdateSetNull()`.

### Errors

Failures that come from the database or its configuration will throw `DatabaseException`, a `RuntimeException`, or one
of the two exceptions that extend it: `ConnectionException` when a [connection](../GLOSSARY.md#connection) has no
configuration or cannot be established, and `QueryException` when a statement cannot be prepared or executed. Mistakes
in how a query is built will throw `InvalidExpressionException`, and in how a row's value is read
`InvalidValueCastException`, both `InvalidArgumentException`s. Mistakes in how a schema is defined will throw
`InvalidSchemaException`, a `LogicException`. The sections above name the condition each is thrown for.

### Out of scope

- **Migrations.** Running a series of migrations against the database, rolling them back, and recording which have
  run are designed separately.
- **Further built-in expressions.** Pattern matching (`LIKE`, `REGEXP`), ranges (`BETWEEN`), `GROUP_CONCAT`, and JSON,
  date and time, string and numeric functions will be added as they are needed. Until then, raw SQL covers them.
- **Drivers other than MySQL.** They can be supported inside the component, but none is exposed.
- **Nested transactions and savepoints.**

## Alternatives considered

This design rests on four decisions recorded separately, with the alternatives each one rejected:
[ADR-0008](../adr/0008-direct-pdo.md), building on PDO rather than a database
abstraction layer; [ADR-0007](../adr/0007-mysql-and-mariadb.md), assuming MySQL or MariaDB;
[ADR-0002](../adr/0002-parameter-attributes.md), injecting
[connection](../GLOSSARY.md#connection)s through a [resolvable attribute](../GLOSSARY.md#resolvable-attribute)
and its [resolver](../GLOSSARY.md#resolver); and [ADR-0005](../adr/0005-typed-configuration.md),
holding the component's configuration in typed objects.

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
- PR [#30], feat(database): Add the database component, merged 2026-04-17 and squashed as [560e9ab]: the implementation,
  whose merge is `decided`. The configuration, the factory, execution, results, transactions, the attribute and
  resolver, the builders' methods and the SQL they write, and the exceptions are described from
  [`src/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/src/Database),
  [`src/Values` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/src/Values),
  [`tests/Unit/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/tests/Unit/Database) and
  [`tests/Integration/Database` at 560e9ab](https://github.com/thegamepanel/panel/tree/560e9ab/tests/Integration/Database).
  None of #26 to #29 designs execution beyond queries being passed to a connection.
- Commit [44f735a], "Add a base abstract config object", 2026-03-29, on the branch of PR [#30]: configuration objects
  restoring themselves by passing an exported array to their constructor as named arguments.
- Commit [745603d], "Require operator in query builder where methods", 2026-03-30, on the branch of PR [#30]: the
  operator being required, with the `=` shorthand removed from `where()` and `orWhere()` on `WhereClause`,
  `JoinClause` and the queries. No reason is given.
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
