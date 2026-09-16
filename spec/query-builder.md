---
title: Query builder
includes: [ADR-0007, RFC-0003]
---

# Query builder

Queries are built as objects that produce their own SQL and the values bound to it. A builder knows nothing of the
connection it runs against, and a connection accepts either a builder or a string, per [Database](database.md).

The SQL is MySQL's, per [ADR-0007](../adr/0007-mysql-and-mariadb-are-the-default-database.md): `REPLACE INTO`,
`INSERT IGNORE`, `ON DUPLICATE KEY UPDATE` and `MATCH ... AGAINST` all appear in what it writes.

## Expressions

`Expression` requires `toSql(): string` and `getBindings(): array`. `Query` extends it and adds nothing, marking a
complete query. `Select`, `Insert`, `Update` and `Delete` implement `Query`; `WhereClause`, `JoinClause`, `Raw` and
the built-in expressions implement `Expression`.

An expression containing others builds its SQL from theirs and gathers their bound values in the same order.

## Select

```php
Select::from('servers')
    ->columns('servers.id', Expressions::count('players.id'))
    ->leftJoin('players', 'players.server_id', '=', 'servers.id')
    ->where('servers.status', '=', 'running')
    ->groupBy('servers.id')
    ->havingRaw('COUNT(players.id) > ?', [0])
    ->orderBy('servers.name')
    ->limit(20)
    ->offset(40);
```

| Method | Effect |
|---|---|
| `from(Expression\|string $table)` | Static. An expression is written in parentheses, as a subquery. |
| `columns(Expression\|string ...$columns)` | Sets the columns, replacing any set before. With none set, `*` is selected. |
| `addColumn(Expression\|string $column)` | Adds one column. |
| `distinct()` | Writes `DISTINCT`. |

Clauses are written in a fixed order whatever order they were added:

```
SELECT [DISTINCT] {columns} FROM {table} {joins} WHERE {conditions} GROUP BY {columns} HAVING {conditions}
ORDER BY {columns} LIMIT {limit} OFFSET {offset}
```

A clause with nothing added is left out. The bound values are gathered in a different order from the clauses: the
table subquery first, then the column expressions, then joins, conditions, grouping, having and ordering.

## Insert

| Method | Effect |
|---|---|
| `into(string $table)` | Static. |
| `values(array $values)` | Adds a row, keyed by column name. Called again, it adds another row to the same query. |
| `ignore()` | Writes `INSERT IGNORE INTO`. |
| `replace()` | Writes `REPLACE INTO`, taking precedence over `ignore()`. |
| `upsert(array $values)` | Writes `ON DUPLICATE KEY UPDATE`, replacing any values set before. An expression value is written as SQL. |

The column list and the placeholder count come from the first row alone, and every row is written as a tuple of that
many placeholders. The bound values are each row's values in that row's own key order, followed by the upsert's.

A second row whose keys are in a different order therefore binds its values in that different order, against a
column list taken from the first row.

`toSql()` reads the first row directly, so an insert with no rows raises an error rather than throwing
`InvalidExpressionException`.

## Update

| Method | Effect |
|---|---|
| `table(string $table)` | Static. |
| `set(array $values)` | Sets columns to values, merged with any set before. An expression value is written as SQL. |

Written as `UPDATE {table} SET {values} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`. The bound values are
the set values, then the conditions, then the ordering.

The column names in `SET` are written between backticks. Nothing else the builder writes is quoted.

## Delete

`Delete::from(string $table)`, written as `DELETE FROM {table} WHERE {conditions} ORDER BY {columns} LIMIT {limit}`.
The bound values are the conditions, then the ordering.

## Conditions

`Select`, `Update` and `Delete` share one set of condition methods, each delegating to a `WhereClause`. Every method
has an `or` form adding the condition with `OR` instead of `AND`.

| Method | Adds |
|---|---|
| `where(Closure\|string $column, ?string $operator = null, mixed $value = null)` | A comparison, or a group when given a closure. |
| `whereNull()`, `whereNotNull()` | `IS NULL` and `IS NOT NULL`. |
| `whereIn(string $column, array\|Expression $values)` | `IN` with a placeholder for each value, or the expression's SQL as a subquery. |
| `whereNotIn()` | `NOT IN`, the same way. |
| `whereRaw(string $sql, array $bindings = [])` | The SQL as given, with its bound values. |
| `whereFullText(array $columns, string $value, string $mode = 'natural')` | A full-text search. |

The first condition has no conjunction; each later one is joined to the one before it. A group is written in
parentheses, and a closure that adds no condition throws.

When a column is given, the operator is required and is matched regardless of case:

| Operator | SQL |
|---|---|
| `=`, `!=`, `<`, `>`, `<=`, `>=` | The comparison, with a placeholder. |
| `is` | `{column} IS ?` |
| `is null`, `is not null` | `IS NULL` and `IS NOT NULL`, binding nothing. |
| `in`, `not in` | `IN` and `NOT IN`, with a placeholder per value. |

Any other operator throws `InvalidExpressionException`. An `IN` or `NOT IN` given an empty array throws.

Given an expression rather than an array, `whereIn()` and `whereNotIn()` delegate to the raw form, so the subquery
is written with the `AND` conjunction. The `or` forms delegate to the raw `or` form.

## Joins

| Method | Adds |
|---|---|
| `join(string $table, Closure\|string $first, ?string $operator = null, ?string $second = null)` | `INNER JOIN`. |
| `leftJoin()`, `rightJoin()` | `LEFT JOIN` and `RIGHT JOIN`. |
| `crossJoin(string $table)` | `CROSS JOIN`, with no conditions. |

Given two columns and an operator, the join compares them. Given a closure, it receives the join's `JoinClause`:

| Method | Adds |
|---|---|
| `on(string $left, string $operator, string $right)` | A comparison of two columns, joined with `AND`. |
| `orOn()` | The same, joined with `OR`. |
| `where(string $column, string $operator, mixed $value)` | A comparison against a bound value, joined with `AND`. |
| `orWhere()` | The same, joined with `OR`. |

A join's conditions follow `ON`. A join with no conditions has no `ON`, which is how a cross join is written.

## Grouping, having, ordering and limits

`groupBy(Expression|string ...$columns)` appends columns, so calling it again adds to those already grouped by.

`having()`, `orHaving()`, `havingRaw()` and `orHavingRaw()` add to a second `WhereClause`, so a having clause is
built from the same conditions as a where clause.

`orderBy(Expression|string $column, string $direction = 'asc')` appends a column. A direction of `desc` in any case
orders descending, and every other value, recognised or not, orders ascending.

`limit(int $limit)` and `offset(int $offset)` are written into the SQL as integers rather than bound. An offset set
without a limit is written on its own.

## Raw SQL

`Raw::from(string $sql, array $bindings = [])` holds SQL and its bound values, and can be used wherever an
expression is accepted. `Expressions::raw()` does the same through a separate class.

## Built-in expressions

| Method | SQL |
|---|---|
| `count(Expression\|string $column = '*')` | `COUNT({column})` |
| `sum()`, `min()`, `max()`, `avg()` | The matching aggregate. |
| `match(array $columns, string $value)` | A natural-language full-text search. |
| `matchBoolean(array $columns, string $value)` | A boolean-mode full-text search. |
| `whereColumn(string $operator, string $column, mixed $value)` | The comparison for the operator. |
| `raw(string $sql, array $bindings)` | The SQL as given. |

An aggregate's column may be an expression, whose SQL is written inside the function.

## Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `InvalidExpressionException` | `InvalidArgumentException` | An operator is not recognised, a grouped condition adds nothing, or an `IN` clause is given an empty array. |
