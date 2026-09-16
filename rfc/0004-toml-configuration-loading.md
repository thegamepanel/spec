---
id: RFC-0004
title: TOML configuration loading
status: accepted
created: 2026-03-30
decided: 2026-06-01
backfilled: 2026-09-14
depends: [ADR-0003, ADR-0006, ADR-0010]
updates: [RFC-0002, RFC-0003]
obsoletes: []
---

# RFC-0004: TOML configuration loading

## Abstract

Configuration is read from TOML files: a main file, drop-in overrides applied in order, and one file for each enabled
module, merged into a single tree with environment variables interpolated. A registry hydrates configuration objects
from that tree in two seals, core configuration first so that the module system can read which modules are enabled,
then each module's configuration, and produces the immutable configuration catalogue.

## Motivation

The panel's configuration is written in TOML files that sysadmins edit, per
[ADR-0010](../adr/0010-configuration-files-are-toml.md). The configuration component holds configuration objects in
a catalogue, per [RFC-0002](0002-configuration-objects.md), but has no way to read them from files.

Modules provide configuration of their own. To register it, the module system needs to know which modules are
enabled, and which modules are enabled is itself configuration, so it has to be readable before any module registers
anything.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Main configuration file | `config.toml`, the base of the configuration. |
| Drop-in | A TOML file in `config.d`, applied over the main configuration file. |
| Module configuration file | A TOML file in `modules-enabled`, holding one module's configuration. Its presence enables the module. |
| Module identifier | A module's name, taken from its module configuration file's name without the `.toml` extension. |
| Configuration tree | The array produced by reading and merging every file. |
| Section | The part of the configuration tree a configuration object is hydrated from. |
| Core configuration | Configuration hydrated before the module system runs, because the module system needs it. |
| Module configuration | Configuration a module registers, hydrated once every module has registered. |

### Components

| Component | Responsibility |
|---|---|
| `ConfigPaths` | Value object holding the paths the loader reads from. |
| `TomlLoader` | Reads, merges and interpolates the TOML files into a configuration tree. |
| `ConfigRegistry` | Mutable, and exists only during bootstrap. Hydrates configuration objects from the tree in two seals, and produces the `ConfigCatalogue`. |
| `CoreConfig` | Maps top-level keys of the tree to core configuration classes. |
| `ModulesEnabled` | Core configuration object holding the enabled module identifiers. |
| `ConfigObject` | Contract every configuration object implements, hydrated through `fromArray()`. |

### Paths

The loader reads from three locations:

```
config.toml
config.d/
    10-database.toml
    20-secrets.toml
modules-enabled/
    admin.toml
    billing.toml
```

`ConfigPaths` holds the absolute path of each: the main configuration file as `configFile`, the drop-in directory as
`configDir`, and the module configuration directory as `modulesEnabledDir`. Bootstrap constructs it, from the
environment, command-line flags or compiled defaults. The configuration component never works out paths itself.

Modules that are installed but not enabled are a concern of the command line that enables and disables them. Only
`modules-enabled` matters to loading.

### Loading

`TomlLoader::load(ConfigPaths $paths): array` builds the configuration tree in these steps:

1. The main configuration file is parsed. If it cannot be read or parsed, loading throws.
2. The drop-ins are parsed in order of filename, and each is merged over the tree in turn. A missing drop-in
   directory is treated as empty.
3. The module configuration files are parsed in order of filename, and each is placed in the tree under `modules`,
   keyed by its module identifier. A missing module configuration directory is treated as empty.
4. The module identifiers, in the same order, are placed in the tree under `__enabled_modules`.
5. Environment variables are interpolated throughout the tree, module configuration included.

Only files with the `.toml` extension are read. The loader knows nothing of configuration objects: it translates
files into an array.

```php
[
    // keys from config.toml, with config.d/*.toml merged over them
    'modules'           => ['admin' => [/* admin.toml */], 'billing' => [/* billing.toml */]],
    '__enabled_modules' => ['admin', 'billing'],
]
```

`modules` and `__enabled_modules` are reserved for the loader. The main configuration file or a drop-in declaring
either at its top level throws, naming the file that declared it.

A module identifier cannot contain a dot, because dots separate the parts of a section's path. A module configuration
file whose name contains a dot throws.

#### Merging

A drop-in is merged over the tree by these rules, applied from the top level down:

- A key only in the tree is kept, and a key only in the drop-in is added.
- Where both have a key and either value is not an array, the drop-in's value replaces the tree's.
- Where both values are arrays and either is a non-empty list, the drop-in's value replaces the tree's. Lists,
  including arrays of tables, are never merged.
- Otherwise both values are tables, and they are merged by these same rules.

An empty array counts as a table, so an empty drop-in file changes nothing, and an empty array replaces a list but
leaves a table as it was.

```toml
# config.toml
[database]
primary = "main"
hosts   = ["a", "b"]

[database.connections.main]
host = "localhost"
port = 3306
```

```toml
# config.d/10-database.toml
[database]
hosts = ["c"]

[database.connections.main]
port = 3307
```

After merging, `database.primary` is `main`, `database.hosts` is `["c"]`, and the `main` connection has the host
`localhost` and the port `3307`.

### Interpolation

A string in the tree may contain any number of references to environment variables, each replaced by the variable's
value read from `Env`, per [ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md):

| Reference | Replaced with |
|---|---|
| `${NAME}` | The variable's value. If the variable is not set, or is `null`, loading throws. |
| `${NAME:-default}` | The variable's value, or the default when the variable is not set or is `null`. |

```toml
title = "${APP_NAME}"
dsn   = "${DB_HOST}:${DB_PORT}"

[database]
password = "${DB_PASSWORD}"
region   = "${REGION:-eu-west}"
tags     = ["${TAG_ONE}", "literal"]
```

A variable name is made of uppercase letters, digits and underscores, and does not start with a digit. A reference
using any other name, such as `${name}`, is left as it was written. Only strings are interpolated, including strings
inside lists, and the result is always a string. Numbers, booleans and other values are left as they are.

A variable that is not set and has no default throws, naming the variable and the path to the value, such as
`database.password` or `database.tags[0]`.

Interpolation keeps secrets, such as database passwords, out of configuration files, which are plain text. Because
it happens while the tree is loaded, `Env` must be initialised before the loader runs.

### Configuration objects

`ConfigObject` requires `fromArray(array $data): static`, which creates the configuration object from its section of
the tree. It replaces the `__set_state()` of [RFC-0002](0002-configuration-objects.md).

A configuration object validates what it is given, and validation happens as it is hydrated. Its typed readonly
properties enforce the shape and types of its values by construction. Assertions in its constructor enforce ranges,
formats and the relationships between values, so an object created any other way, such as through a `make()`
factory, is validated the same way. `fromArray()` asserts only that the keys it needs exist and have the right
shape. A failed assertion throws.

```php
final readonly class MailConfig implements ConfigObject
{
    public static function fromArray(array $data): static
    {
        Assert::keyExists($data, 'host', 'Host is not defined.');
        Assert::keyExists($data, 'port', 'Port is not defined.');

        return new self($data['host'], $data['port']);
    }

    private function __construct(
        public string $host,
        public int $port,
    ) {
        Assert::stringNotEmpty($host, 'Host is not defined.');
        Assert::range($port, 1, 65535, 'Port is out of range.');
    }
}
```

### Registry

`ConfigRegistry` is the mutable half of the configuration component, sealed into the immutable `ConfigCatalogue`,
per [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md). It is constructed with the
configuration tree and the core mapping, and is discarded once it has been sealed.

```php
$tree     = new TomlLoader()->load($paths);
$registry = new ConfigRegistry($tree, CoreConfig::MAPPING);

$registry->sealCore();

$enabled = $registry->for(ModulesEnabled::class);

// Each enabled module registers its configuration.
$registry->register('admin', 'main', AdminConfig::class);

$catalogue = $registry->seal();
```

| Method | Effect |
|---|---|
| `sealCore(): void` | Hydrates every core configuration object. |
| `for(string $class): ConfigObject` | Returns a hydrated core configuration object by its class. |
| `register(string $module, string $name, string $class): void` | Registers a module's configuration class under the module and a name, to be hydrated by `seal()`. |
| `seal(): ConfigCatalogue` | Hydrates every registered module configuration object, and returns the catalogue holding them and the core configuration. |

The registry passes through three phases, and each method can only be called in some of them:

| Phase | Entered by | Permits |
|---|---|---|
| Open | Construction | `sealCore()` |
| Core sealed | `sealCore()` | `for()`, `register()` and `seal()` |
| Sealed | `seal()` | Nothing. The catalogue is used instead. |

A method called in a phase that does not permit it throws `ConfigLifecycleException`.

#### Core configuration

Core configuration is what must be hydrated and available before the module system runs. `CoreConfig::MAPPING` maps
a top-level key of the tree to the class hydrated from it:

```php
public const array MAPPING = [
    '__enabled_modules' => ModulesEnabled::class,
];
```

`sealCore()` hydrates each class from its key. A missing key hydrates the class from an empty array, and a key holding
anything other than an array throws. In the catalogue, core configuration is placed under the module `engine`, named
by its key.

`ModulesEnabled` holds the enabled module identifiers as a list of strings, and nothing else. Its
`has(string $module): bool` returns whether a module is enabled. It is the only core configuration.

The module system reads `ModulesEnabled` through `for()`, between the two seals. It does not scan the filesystem, or
repeat the logic that decides which modules are enabled: the configuration component is the single source of which
modules are enabled.

#### Module configuration

A module registers each of its configuration classes under its module identifier and a name. `seal()` hydrates the
class from `modules.{module}.{name}` in the tree, which is the table `{name}` in the module's configuration file.

```toml
# modules-enabled/admin.toml
[main]
value = "${ADMIN_VALUE}"
```

```php
$registry->register('admin', 'main', AdminConfig::class);
```

A missing table hydrates the class from an empty array, and anything other than an array on that path throws. The
module name `engine` is reserved for core configuration, and registering under it throws. So does registering with a
module identifier or name that contains a dot.

In the catalogue, each module configuration object is placed under its module and name, so both
`$catalogue->get('admin', 'main')` and `$catalogue->for(AdminConfig::class)` return it.

#### Hydration errors

Any failure while a configuration object is hydrated is wrapped in `InvalidConfigException`, naming the file, section
and key it came from, so that sysadmins can find the problem without guessing:

```
Invalid config in modules-enabled/admin.toml at [main.value]: Value is not defined.
```

### Database configuration

`DatabaseConfig` and `ConnectionConfig`, from [RFC-0003](0003-database-component.md), are hydrated through
`fromArray()`, with their validation in their constructors.

```toml
primary    = "main"
persistent = false

[connections.main]
host     = "127.0.0.1"
port     = 3306
database = "panel"
username = "panel"
password = "${DB_PASSWORD}"

[connections.local]
socket   = "/var/run/mysqld/mysqld.sock"
database = "panel"
username = "panel"
password = ""
```

| `DatabaseConfig` key | Rule |
|---|---|
| `primary` | Required, and not empty. A connection must be configured under it. |
| `connections` | Required. A table of connections, keyed by name, with at least one. Each is a non-empty table hydrated into a `ConnectionConfig`, and a failure names the connection. |
| `persistent` | Optional, and a boolean. Defaults to `false`. |

| `ConnectionConfig` key | Rule |
|---|---|
| `database`, `username` | Required, and not empty. |
| `password` | Required, and may be empty. |
| `socket` | Optional. When given, it must not be empty, and the connection is made through it. |
| `host` | Required when no socket is given, and not empty. |
| `port` | Required when no socket is given, and an integer. |
| `options` | Optional, and a table. |

A connection through a host must configure its port, which no longer defaults to `3306`.

### Errors

Every exception implements `ConfigException`, per [RFC-0002](0002-configuration-objects.md).

| Exception | Extends | Thrown when |
|---|---|---|
| `InvalidConfigException` | `RuntimeException` | A file cannot be read or parsed, a reserved key is declared, a module configuration file's name contains a dot, an environment variable is not set and has no default, a section is not an array, or a configuration object fails to hydrate. |
| `ConfigLifecycleException` | `LogicException` | A registry method is called in a phase that does not permit it, a module registers as `engine`, or a module identifier or configuration name contains a dot. |
| `ConfigNotRegisteredException` | `RuntimeException` | `for()` is asked for a class that is not core configuration. |

### Out of scope

- **Where the files live, and constructing `ConfigPaths`.** Both belong to bootstrap.
- **Enabling and disabling modules.** Both belong to the command line.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- writing configuration in TOML rather than PHP, JSON or YAML, in
  [ADR-0010](../adr/0010-configuration-files-are-toml.md)
- sealing a mutable registry into an immutable catalogue in phases, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- reading environment variables only during bootstrap, in
  [ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md)

No other alternatives were weighed.

## Backwards compatibility

- `ConfigObject` requires `fromArray()` in place of `__set_state()`, so configuration objects are no longer restored
  from `var_export()` output. Every configuration object implements `fromArray()`.
- `BaseConfigObject`, which restored a configuration object by passing an exported array to its constructor as named
  arguments, is removed.
- `DatabaseConfig` and `ConnectionConfig` are hydrated through `fromArray()`, and a connection through a host must
  configure its port.

## Open questions

## Changelog

## Sources

- Issue [#31], Engine - Config - TOML, original text of 2026-03-30 in its edit history: that configuration need not
  be PHP files for a panel distributed as a binary, the TOML parser, and modules needing a way to define how their
  configuration is loaded. This is `created`.
- Issue [#31], rewritten on 2026-05-14 and revised on 2026-05-19, in its edit history: `ConfigPaths`, the loader and
  its order, module configuration namespaced by filename, the enabled module identifiers taken from filenames,
  `ModulesEnabled` as core configuration and the single source of which modules are enabled, `${VAR}` interpolation
  against `Env` keeping secrets out of plain-text files, the two seals with `for()` readable between them,
  validation on hydration with the file, section and key named, and the exceptions. The design follows the revision
  of 2026-05-19.
- Planning session, 2026-04-22, not publicly available and first written down on 2026-09-13: interpolation against
  the environment, and validation as configuration is loaded.
- PR [#33], refactor(engine:config): Switch to TOML loader and two-phase registry, merged 2026-06-01 and squashed as
  [44b0aaa]: the implementation, whose merge is `decided`. `ConfigRegistry`, `ConfigLifecycleException`, `CoreConfig`,
  `${VAR:-default}`, the reserved keys, the merge rules, missing directories, variable names, the registry's phases,
  the `engine` module and the database configuration keys are described from
  [`src/Config` at 44b0aaa](https://github.com/thegamepanel/panel/tree/44b0aaa/src/Config),
  [`src/Database/Config` at 44b0aaa](https://github.com/thegamepanel/panel/tree/44b0aaa/src/Database/Config) and
  [`tests/Unit/Config` at 44b0aaa](https://github.com/thegamepanel/panel/tree/44b0aaa/tests/Unit/Config).
- Automated review of PR [#33], 2026-05-20: checking reserved keys in each file so that the error names the file
  that declared one, rejecting dots in module identifiers and names, and refusing `seal()` before `sealCore()`.
- Commit [914a46c], "Add TOML parser and assertion helper", 2026-04-18, on the branch of PR [#33]: `internal/toml`
  and `webmozart/assert` added. No reason is given for choosing either.
- Commit [bb4f014], "Move config validation into constructors", 2026-05-20, on the branch of PR [#33]: validation in
  constructors so that `make()` cannot bypass it, and the port required for a connection through a host.
- A password being allowed to be empty, and a hydration error naming the key and the file that set the value: first
  written down on 2026-09-14. The code at [44b0aaa] differs: a password must not be empty, a hydration error never
  names the key, and a core section's error always names `config.toml`.
- Issue [#48], Engine - Database - Connection configuration fixes, 2026-08-08: that forbidding an empty password rules
  out socket peer authentication, taken from what followed rather than anticipated.

[#31]: https://github.com/thegamepanel/panel/issues/31
[#33]: https://github.com/thegamepanel/panel/pull/33
[#48]: https://github.com/thegamepanel/panel/issues/48
[44b0aaa]: https://github.com/thegamepanel/panel/commit/44b0aaa
[914a46c]: https://github.com/thegamepanel/panel/commit/914a46c
[bb4f014]: https://github.com/thegamepanel/panel/commit/bb4f014
