---
title: Values
includes: []
---

# Values

Type-coerced access to values held in an array, keyed by name. `ValueGetter` provides it as static methods, and
`GetsAsType` is the contract an object implements to expose the same five casts over its own data. `Row` is the one
implementation, per [Database](database.md).

## Casting

Each method takes the name and the array, reads `$values[$name] ?? null`, and casts it. A value the method cannot
cast throws `InvalidValueCastException`, naming the value and the type it could not become.

| Method | Returns the value when it is | Otherwise |
|---|---|---|
| `string(string $name, array $values)` | A string, or any numeric value cast to a string. | Throws. |
| `int(string $name, array $values)` | An integer, or any numeric value cast to an integer. | Throws. |
| `float(string $name, array $values)` | A float, or any numeric value cast to a float. | Throws. |
| `bool(string $name, array $values)` | A boolean, an integer cast to a boolean, or one of the strings `true`, `1`, `yes`, `false`, `0`, `no`. | Throws, including for any other string. |
| `array(string $name, array $values)` | An array, or a string holding valid JSON, decoded. | Throws. |

A name that is absent from the array is read as `null`, and `null` satisfies none of the casts, so a missing value
and a `null` value fail the same way and are not distinguishable from the cast alone.

`string()` does not accept a boolean. `bool()` accepts an integer, and `int()` accepts a numeric string, so the
casts are not symmetrical with one another.

`array()` validates a string with `json_validate()` before decoding, and decodes with `JSON_THROW_ON_ERROR` to a
depth of 512.

## The contract

```php
interface GetsAsType
{
    public function string(string $name): string;

    public function int(string $name): int;

    public function float(string $name): float;

    public function bool(string $name): bool;

    public function array(string $name): array;
}
```

An implementation supplies the array itself, so a caller names only the value. Each method throws
`InvalidValueCastException` on a value it cannot cast.

## Errors

| Exception | Thrown when |
|---|---|
| `InvalidValueCastException` | A value cannot be cast to the type asked for. |
