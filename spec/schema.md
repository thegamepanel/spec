---
title: Schema builder
includes: [ADR-0007, RFC-0003]
---

# Schema builder

Schema changes are built as objects that produce their own SQL, alongside the [query builder](query-builder.md) and
executed through a connection like any other statement, per [Database](database.md). Schema statements bind no
values: every literal is written into the SQL.

The DDL is MySQL's, per [ADR-0007](../adr/0007-mysql-and-mariadb-are-the-default-database.md).

## Statements

| Factory | Produces |
|---|---|
| `Create::table(string $table, array $columns)` | `CREATE TABLE` with the columns given. |
| `Create::database(string $database)` | `CREATE DATABASE`. |
| `Alter::table(string $table)` | `ALTER TABLE`. |
| `Alter::database(string $database)` | `ALTER DATABASE`. |
| `Drop::table()`, `database()`, `column()`, `index()`, `primaryKey()`, `foreignKey()` | `DROP` of that type. |
| `Rename::table(string $table, string $newName)` | `RENAME TABLE`. |
| `Truncate::table(string $table)` | `TRUNCATE TABLE`. |

`Schema` marks a complete statement and extends `Expression`, adding nothing. `Column` and `Index` mark a column and
an index definition, and both extend `Expression` too, so a definition is an expression in its own right.

Table, column and index names are written between backticks. Nothing doubles a backtick inside a name, and string
literals, comments and defaults are written between single quotes without escaping.

## Creating a table

```php
Create::table('servers', [
    Column::bigInt('id')->unsigned()->autoIncrement(),
    Column::varchar('name', 255)->notNull(),
    Column::enum('status', ServerStatus::class)->default(ServerStatus::Stopped),
    Column::timestamp('created_at')->defaultCurrentTimestamp(),
])
    ->indexes(
        Index::primary('id'),
        Index::foreign('servers_owner_foreign', 'owner_id')->on('users')->references('id')->onDeleteCascade(),
    )
    ->engine('InnoDB')
    ->charset('utf8mb4');
```

| Method | Effect |
|---|---|
| `ifNotExists()` | Writes `IF NOT EXISTS`. |
| `temporary()` | Writes `TEMPORARY`. |
| `indexes(Index ...$indexes)` | Sets the indexes, replacing any set before. |
| `engine(string $engine)` | Sets the storage engine. |
| `charset()`, `collation()` | Set the character set and collation. |
| `autoIncrement(int $value)` | Sets the value auto-increment starts from. |
| `comment(string $comment)` | Sets the table's comment. |
| `options(Expression ...$options)` | Sets further options as expressions, replacing any set before. |

The columns and then the indexes are written inside parentheses, one per line, and the table options follow, each
separated by a comma.

## Altering a table

| Method | Effect |
|---|---|
| `rename(string $newName)` | Renames the table. |
| `add(Column\|Index ...$new)` | Adds columns and indexes, sorted by what each one is. |
| `modify(Column ...$columns)` | Replaces the definitions of existing columns. |
| `move(string $column, string $newColumn)` | Renames a column. |
| `drop(Drop ...$drops)` | Drops columns, indexes, foreign keys or the primary key. |

Every change is written into one `ALTER TABLE` statement, in a fixed order whatever order they were added:

1. Renaming the table
2. Dropping the primary key
3. Dropping indexes and foreign keys
4. Dropping columns
5. Modifying columns
6. Renaming columns
7. Adding columns
8. Adding indexes

Several changes can refer to the same thing, such as a column dropped and added again rather than modified.

`drop()` sorts each `Drop` by what it targets. A drop of a table or a database throws `InvalidArgumentException`.
Dropping the primary key is recorded as a flag, so the name given to `Drop::primaryKey()` is not used.

## Dropping

`Drop` writes `DROP {type} \`{name}\``, with `TEMPORARY` before the type and `IF EXISTS` after it where each
applies.

`ifExists()` is valid only on a table or a database, and `temporary()` only on a table. Either used on another type
throws `InvalidSchemaException`.

A drop of a table or a database is a statement executed on its own. The others are used within `Alter::table()`.

## Columns

`Column` is a factory, and the method used fixes the column's type.

| Family | Factories | Sizing |
|---|---|---|
| Integer | `tinyInt()`, `smallInt()`, `mediumInt()`, `int()`, `bigInt()` | An optional display length. |
| Decimal | `decimal()`, `float()`, `double()` | An optional length and number of decimal places. |
| String | `char()`, `varchar()`, `text()`, `tinytext()`, `mediumtext()`, `longtext()` | A length on `char`, `varchar` and `text`. |
| Binary | `binary()`, `varbinary()`, `tinyblob()`, `blob()`, `mediumblob()`, `longblob()` | A length on `binary`, `varbinary` and `blob`. |
| Enumerated | `enum()`, `set()` | The permitted values, as strings or the class of a backed enum. |
| Temporal | `date()`, `datetime()`, `timestamp()`, `time()`, `year()` | A fractional seconds precision. |
| Boolean | `boolean()` | None. |
| JSON | `json()` | None. |

Every column has these modifiers:

| Modifier | Effect |
|---|---|
| `nullable()` | Writes `NULL`, cancelling `notNull()`. |
| `notNull()` | Writes `NOT NULL`, cancelling `nullable()`. |
| `default(mixed $value)` | Sets the default. |
| `unique()` | Writes `UNIQUE`. |
| `comment(string $comment)` | Sets the column's comment. |
| `after(string $column)`, `first()` | Recorded, and not written into the SQL. |
| `generated(Query $query)` | Writes `GENERATED ALWAYS AS ({sql})`. |
| `virtual()`, `stored()` | Mark a generated column's storage. On a column that is not generated, either throws. |

Some families have further modifiers, each throwing `InvalidSchemaException` when used on a column type it does not
apply to:

| Modifier | Families |
|---|---|
| `unsigned()` | Integer, decimal |
| `autoIncrement()` | Integer |
| `length(int $length)` | String and binary, on the types that take one |
| `charset()`, `collation()` | String, enumerated |
| `precision(int $precision)` | Temporal, on the types that take one |
| `defaultCurrentTimestamp()`, `onUpdateCurrentTimestamp()` | Temporal, on the types that take one |

A column is written as its backtick-quoted name, then its type definition. A generated column then has its
expression written, and no null modifier or default. Every other column has its null modifier and default written.
`UNIQUE` and the comment follow.

A default is written according to its value:

| Value | Written as |
|---|---|
| `null` | `NULL` |
| A string | A single-quoted literal. |
| An integer or float | The number. |
| A boolean | `1` or `0`. |
| A backed enum case | Its backing value, as a literal or a number. |
| An array or `JsonSerializable` | Its JSON encoding. |
| An expression | Its SQL. |

## Indexes

| Factory | Writes |
|---|---|
| `Index::primary(string ...$columns)` | A primary key over the columns. |
| `Index::unique(string $name, string ...$columns)` | A unique index. |
| `Index::index(string $name, string ...$columns)` | A plain index. |
| `Index::fulltext(string $name, string ...$columns)` | A full-text index. |
| `Index::foreign(string $name, string ...$columns)` | A foreign key constraint. |

A foreign key is completed with `on(string $table)` and `references(string ...$columns)`, both required, and
optionally one `ON DELETE` and one `ON UPDATE` action from `onDeleteCascade()`, `onDeleteSetNull()`,
`onDeleteRestrict()`, `onDeleteSetDefault()`, `onDeleteNoAction()` and their `onUpdate` counterparts.

Producing the SQL of a foreign key with no referenced table, or no referenced columns, throws
`InvalidSchemaException`.

## Migrations

`Migration` and `ReversibleMigration` are contracts. Nothing runs them, per [Database](database.md).

## Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `InvalidSchemaException` | `LogicException` | A modifier is used on a type it does not apply to, `virtual()` or `stored()` is used on a column that is not generated, or a foreign key has no referenced table or columns. |
| `InvalidArgumentException` | | `Alter::table()` is given a drop of a table or a database. |
