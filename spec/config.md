---
title: Configuration
includes: [ADR-0003, ADR-0005, ADR-0006, ADR-0010, RFC-0002, RFC-0004, RFC-0005]
---

# Configuration

Configuration is read from TOML files into a single tree, and the tree hydrates typed configuration objects held in
an immutable catalogue. Environment variables are available during bootstrap through a static class, and are
interpolated into the tree as it loads.

## Paths

`Paths` is a readonly value object holding five absolute directories: `config`, `data`, `modules`, `cache` and
`logs`. Each has a method of the same name joining a relative path onto it, trimming the root's trailing separator
and the path's leading one, so a path given with or without a leading separator produces the same result.

`Paths` holds the roots and does nothing else with them. It does not work out where they are, check that they exist,
or create them. Nothing binds it into the container, and only tests construct it.

## Loading

`TomlLoader::load(Paths $paths): array` builds the configuration tree:

1. `config.toml` is parsed from the configuration directory.
2. Its top level is checked for the reserved keys `modules` and `__enabled_modules`.
3. Every `*.toml` in `config.d` is parsed in filename order, checked for the same reserved keys, and merged over the
   tree in turn. A missing directory is treated as empty.
4. Every `*.toml` in `modules-enabled` is parsed in filename order and placed under `modules`, keyed by its filename
   without the extension. A missing directory is treated as empty.
5. Those identifiers, in the same order, are placed under `__enabled_modules`.
6. Environment variables are interpolated throughout the tree.

Only files with a `.toml` extension are read. Dots separate the segments of a section path, and a module identifier
containing one throws.

```php
[
    // keys from config.toml, with config.d/*.toml merged over them
    'modules'           => ['admin' => [/* admin.toml */]],
    '__enabled_modules' => ['admin'],
]
```

### Merging

A drop-in is merged over the tree from the top level down. Where both values are arrays and neither is a non-empty
list, they merge recursively. Otherwise the drop-in's value replaces the tree's, so lists and arrays of tables are
replaced whole rather than merged.

An empty array is not treated as a list, so an empty drop-in file merges as a no-op rather than replacing the tree.

### Interpolation

A string in the tree may contain any number of references:

| Reference | Replaced with |
|---|---|
| `${NAME}` | The variable's value. Loading throws when the variable is absent or `null`. |
| `${NAME:-default}` | The variable's value, or the default when it is absent or `null`. |

A variable name starts with an uppercase letter or underscore, followed by uppercase letters, digits or
underscores. A reference written any other way is left as it is. Only strings are interpolated, including strings
inside lists, and the result is always a string. The error names the variable and the dotted path to the value, with
list positions written as `tags[0]`.

Interpolation reads through `Env`, so `Env` is initialised before the loader runs.

## Configuration objects

A configuration object implements `ConfigObject`, which requires `fromArray(array $data): static`. Each
configuration is an instance of a class unique to it, per
[ADR-0005](../adr/0005-configuration-is-held-in-typed-objects.md), and its values are typed readonly properties.

`ModulesEnabled` is the only core configuration object.

## The registry

`ConfigRegistry` is the mutable half, sealed into `ConfigCatalogue`, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md). It is constructed with the tree
and the core mapping.

| Method | Effect |
|---|---|
| `sealCore(): void` | Hydrates every class in the core mapping from its top-level key. |
| `for(string $class): ConfigObject` | Returns a hydrated core configuration object by class. |
| `register(string $module, string $name, string $class): void` | Registers a module's configuration class for hydration. |
| `seal(): ConfigCatalogue` | Hydrates every registration and returns the catalogue. |

It passes through three phases:

| Phase | Entered by | Permits |
|---|---|---|
| Open | Construction | `sealCore()` |
| Core sealed | `sealCore()` | `for()`, `register()`, `seal()` |
| Sealed | `seal()` | Nothing |

`ConfigLifecycleException` is thrown when a method is called in a phase that does not permit it: sealing the core
twice, reading before the core seal or after the full seal, registering before the core seal or after the full seal,
sealing twice, and sealing before the core seal.

`CoreConfig::MAPPING` maps a top-level key of the tree to the class hydrated from it, and holds
`__enabled_modules => ModulesEnabled` alone. A missing key hydrates from an empty array; a key holding anything
other than an array throws. Core configuration is placed in the catalogue under the module `engine`, named by its
key.

A module registration records the module, the name, and the section path `modules.{module}.{name}`. Registering
under `engine` throws, and so does a module or name containing a dot. `seal()` walks that path into the tree, taking
a missing segment as an empty array and throwing when a segment holds something that is not an array.

Any failure while hydrating is wrapped in `InvalidConfigException`, naming the file and section it came from.

## The catalogue

`ConfigCatalogue` is constructed with the configuration objects, keyed by module and then by name, and builds a map
from each object's class to its module and name.

| Method | Effect |
|---|---|
| `get(string $module, string $config): ?ConfigObject` | The object under that module and name, or `null`. |
| `has(string $module, string $config): bool` | Whether one is held there. |
| `for(string $class): ?ConfigObject` | The object of that class, or `null`. |

Nothing binds configuration objects into the container.

## Environment variables

`Env` is a static singleton holding environment variables, per
[ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md). `destroy()` removes the instance.
Nothing else confines it: once initialised, it is readable from anywhere until it is destroyed.

| Method | Effect |
|---|---|
| `createFromSuperglobal()` | Initialises from `$_ENV`. |
| `createFromFile(string $path)` | Initialises from the `.env` file in the directory, read array-backed so nothing is written into `$_ENV`. |
| `destroy()` | Removes the instance. |
| `get(string $key, $default = null)` | The value, or the default when absent or `null`. |
| `has(string $key)` | Whether the key is present, including when its value is `null`. |

Initialising twice throws, and reading before initialising throws.

The typed accessors differ in how they handle a value they cannot cast:

| Accessor | Casts | An uncastable value |
|---|---|---|
| `string()` | A string, number or boolean | Returns the default. |
| `int()` | An integer, numeric string or boolean | Throws `InvalidEnvException`. |
| `float()` | A float, numeric string or boolean | Throws `InvalidEnvException`. |
| `bool()` | A boolean or integer, and the strings `true`, `1`, `yes`, `false`, `0`, `no` | Throws `InvalidEnvException`, including for any other string. |

Each returns the default when the value is absent or `null`.

## Errors

Every exception implements `ConfigException`.

| Exception | Thrown when |
|---|---|
| `InvalidConfigException` | A file cannot be read or parsed, a reserved key is declared, a module filename contains a dot, an environment variable is absent with no default, a section is not an array, or hydration fails. |
| `ConfigLifecycleException` | A registry method is called in a phase that does not permit it, a module registers as `engine`, or a module or name contains a dot. |
| `ConfigNotRegisteredException` | `for()` is asked for a class that is not core configuration. |
| `EnvInitialisationException` | `Env` is read before initialisation, or initialised twice. |
| `InvalidEnvException` | A typed accessor cannot cast a value. |
| `MissingEnvVariableException` | Nothing throws it. |
