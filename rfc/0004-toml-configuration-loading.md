---
id: RFC-0004
title: TOML configuration loading
status: accepted
created: 2026-03-30
decided: 2026-06-01
backfilled: 2026-11-18
depends: [ADR-0003, ADR-0006, ADR-0010]
updates: [RFC-0002, RFC-0003]
obsoletes: []
---

# RFC-0004: TOML configuration loading

## Abstract

I will read configuration from TOML files: a main file, [drop-in](../GLOSSARY.md#drop-in) overrides applied in order,
and one file for each enabled [module](../GLOSSARY.md#module), merged into a single
[configuration tree](../GLOSSARY.md#configuration-tree) with
[environment variable](../GLOSSARY.md#environment-variable)s interpolated. A [registry](../GLOSSARY.md#registry) will
hydrate [configuration object](../GLOSSARY.md#configuration-object)s from that tree in two seals,
[core configuration](../GLOSSARY.md#core-configuration) first so that the module system can read which modules are
enabled, then each module's configuration, and will produce the immutable configuration
[catalogue](../GLOSSARY.md#catalogue).

## Motivation

The panel's configuration is written in TOML files that sysadmins edit, as
[ADR-0010](../adr/0010-toml-configuration.md) decides. The configuration component holds
[configuration object](../GLOSSARY.md#configuration-object)s in a [catalogue](../GLOSSARY.md#catalogue), as
[RFC-0002](0002-configuration-objects.md) describes, but has no way to read them from files.

[Module](../GLOSSARY.md#module)s provide configuration of their own. To register it, the module system needs to know
which modules are enabled, and which modules are enabled is itself configuration, so it has to be readable before any
module registers anything.

## Proposal

### Where configuration is read from

Configuration will be read from three places: the main configuration file, `config.toml`, which is the base of the
configuration; the [drop-in](../GLOSSARY.md#drop-in) directory, `config.d/`, whose files are applied over it; and the
[module](../GLOSSARY.md#module) configuration directory, `modules-enabled/`, which holds one file for each enabled
module. A module's file being present is what enables it.

```
config.toml
config.d/
    10-database.toml
    20-secrets.toml
modules-enabled/
    admin.toml
    billing.toml
```

`ConfigPaths` will hold the absolute path of each, as `configFile`, `configDir` and `modulesEnabledDir`. Bootstrap will
construct it from the environment, command-line flags or compiled defaults, and the configuration component will never
work out a path itself.

Modules that are installed but not enabled will be a concern of the command line that enables and disables them. Only
`modules-enabled/` will matter to loading.

### Building the tree

`TomlLoader::load(ConfigPaths $paths): array` will read every file and return the
[configuration tree](../GLOSSARY.md#configuration-tree), an array holding all of them. It will work in this order:

1. The main configuration file is parsed. If it cannot be read or parsed, loading throws.
2. The [drop-in](../GLOSSARY.md#drop-in)s are parsed in order of filename, and each is merged over the tree in turn. A
   missing drop-in directory is treated as empty.
3. The [module](../GLOSSARY.md#module) configuration files are parsed in order of filename, and each is placed in the
   tree under `modules`, keyed by its [module identifier](../GLOSSARY.md#module-identifier), which is the file's name
   without the `.toml` extension. A missing module configuration directory is treated as empty.
4. The module identifiers, in the same order, are placed in the tree under `__enabled_modules`.
5. [Environment variable](../GLOSSARY.md#environment-variable)s are interpolated throughout the tree, module
   configuration included.

Only files with the `.toml` extension will be read. The loader will know nothing of
[configuration object](../GLOSSARY.md#configuration-object)s: it translates files into an array, and nothing more.

```php
[
    // keys from config.toml, with config.d/*.toml merged over them
    'modules'           => ['admin' => [/* admin.toml */], 'billing' => [/* billing.toml */]],
    '__enabled_modules' => ['admin', 'billing'],
]
```

`modules` and `__enabled_modules` will be reserved for the loader. The main configuration file or a drop-in that
declares either at its top level will throw, naming the file that declared it.

A module identifier cannot contain a dot, because dots separate the parts of a
[section](../GLOSSARY.md#section)'s path, so a module configuration file whose name contains a dot will throw.

#### Merging

A [drop-in](../GLOSSARY.md#drop-in) will be merged over the tree from the top level down. A key that only the tree has
is kept, and a key that only the drop-in has is added. Where both have a key:

- If either value is not an array, the drop-in's value replaces the tree's.
- If both are arrays and either is a non-empty list, the drop-in's value replaces the tree's. Lists, including arrays
  of tables, are never merged.
- Otherwise both are tables, and they are merged by these same rules.

An empty array counts as a table. An empty drop-in file therefore changes nothing, and an empty array replaces a list
but leaves a table as it was.

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

### Environment variables in configuration

A string in the tree may contain any number of references to
[environment variable](../GLOSSARY.md#environment-variable)s, each replaced by the variable's value as `Env` reads it,
following [ADR-0006](../adr/0006-environment-at-bootstrap.md). A reference takes one of two
forms:

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

A variable's name is made of uppercase letters, digits and underscores, and does not start with a digit. A reference
using any other name, such as `${name}`, will be left as it was written. Only strings will be interpolated, including
strings inside lists, and the result will always be a string; numbers, booleans and other values will be left as they
are. A variable that is not set and has no default will throw, naming the variable and the path to the value, such as
`database.password` or `database.tags[0]`.

Interpolation keeps secrets, such as database passwords, out of configuration files, which are plain text. Because it
happens while the tree is loaded, `Env` must be initialised before the loader runs.

### Validating as configuration is hydrated

`ConfigObject` will require `fromArray(array $data): static`, which creates a
[configuration object](../GLOSSARY.md#configuration-object) from its [section](../GLOSSARY.md#section) of the tree. It
will replace the `__set_state()` of [RFC-0002](0002-configuration-objects.md).

A configuration object will validate what it is given, as it is hydrated. Its typed readonly properties will enforce the
shape and types of its values by construction. Assertions in its constructor will enforce ranges, formats and the
relationships between values, so that an object created any other way, such as through a `make()` factory, is
validated the same way. `fromArray()` will assert only that the keys it needs exist and have the right shape. A failed
assertion throws.

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

### Hydrating in two seals

`ConfigRegistry` will be the configuration [registry](../GLOSSARY.md#registry), the mutable half of the configuration
component, sealed into the immutable `ConfigCatalogue`, following
[ADR-0003](../adr/0003-sealed-catalogues.md). It will be constructed with the
[configuration tree](../GLOSSARY.md#configuration-tree) and the core mapping, will exist only during bootstrap, and will
be discarded once it has been sealed.

```php
$tree     = new TomlLoader()->load($paths);
$registry = new ConfigRegistry($tree, CoreConfig::MAPPING);

$registry->sealCore();

$enabled = $registry->for(ModulesEnabled::class);

// Each enabled module registers its configuration.
$registry->register('admin', 'main', AdminConfig::class);

$catalogue = $registry->seal();
```

Hydration will happen in a fixed sequence. First, `sealCore()` hydrates every
[core configuration](../GLOSSARY.md#core-configuration) object. The [module](../GLOSSARY.md#module) system then reads
which modules are enabled, through `for(string $class): ConfigObject`, which returns a hydrated core configuration
object by its class. Each enabled module then registers its configuration classes, through
`register(string $module, string $name, string $class)`, under its identifier and a name. Last,
`seal(): ConfigCatalogue` hydrates every registered module configuration object and returns the
[catalogue](../GLOSSARY.md#catalogue) holding them alongside the core configuration.

The registry will pass through three phases, and each method will be callable only in some of them:

| Phase | Entered by | Permits |
|---|---|---|
| Open | Construction | `sealCore()` |
| Core sealed | `sealCore()` | `for()`, `register()` and `seal()` |
| Sealed | `seal()` | Nothing. The catalogue is used instead. |

A method called in a phase that does not permit it will throw `ConfigLifecycleException`.

#### Core configuration

[Core configuration](../GLOSSARY.md#core-configuration) is what must be hydrated and available before the
[module](../GLOSSARY.md#module) system runs. `CoreConfig::MAPPING` will map a top-level key of the tree to the class
hydrated from it:

```php
public const array MAPPING = [
    '__enabled_modules' => ModulesEnabled::class,
];
```

`sealCore()` will hydrate each class from its key. A missing key will hydrate the class from an empty array, and a key
holding anything other than an array will throw. In the [catalogue](../GLOSSARY.md#catalogue), core configuration will
be placed under the module `engine`, named by its key.

`ModulesEnabled` will hold the enabled [module identifier](../GLOSSARY.md#module-identifier)s as a list of strings, and
nothing else, and its `has(string $module): bool` will say whether a module is enabled. It will be the only core
configuration.

The module system will read `ModulesEnabled` through `for()`, between the two seals. It will not scan the filesystem or
repeat the logic that decides which modules are enabled: the configuration component will be the single source of
which modules are enabled.

#### Module configuration

A [module](../GLOSSARY.md#module) will register each of its configuration classes under its
[module identifier](../GLOSSARY.md#module-identifier) and a name. `seal()` will hydrate the class from
`modules.{module}.{name}` in the tree, which is the table `{name}` in the module's configuration file.

```toml
# modules-enabled/admin.toml
[main]
value = "${ADMIN_VALUE}"
```

```php
$registry->register('admin', 'main', AdminConfig::class);
```

A missing table will hydrate the class from an empty array, and anything other than an array on that path will throw.
The module name `engine` will be reserved for [core configuration](../GLOSSARY.md#core-configuration), and registering
under it will throw. So will registering with a module identifier or a name that contains a dot.

In the [catalogue](../GLOSSARY.md#catalogue), each module configuration object will be placed under its module and name,
so both `$catalogue->get('admin', 'main')` and `$catalogue->for(AdminConfig::class)` return it.

#### Hydration errors

Any failure while a [configuration object](../GLOSSARY.md#configuration-object) is hydrated will be wrapped in
`InvalidConfigException`, naming the file, [section](../GLOSSARY.md#section) and key the problem came from, so that
sysadmins can find it without guessing:

```
Invalid config in modules-enabled/admin.toml at [main.value]: Value is not defined.
```

### Database configuration

`DatabaseConfig` and `ConnectionConfig`, from [RFC-0003](0003-database-component.md), will be hydrated through
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
| `primary` | Required, and not empty. A [connection](../GLOSSARY.md#connection) must be configured under it. |
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

A connection through a host will have to configure its port, which will no longer default to `3306`.

### Errors

Every exception will implement `ConfigException`, from [RFC-0002](0002-configuration-objects.md).

`InvalidConfigException`, a `RuntimeException`, will be thrown for anything wrong with the configuration itself: a file
that cannot be read or parsed, a reserved key declared, a [module](../GLOSSARY.md#module) configuration file whose name
contains a dot, an [environment variable](../GLOSSARY.md#environment-variable) that is not set and has no default, a
[section](../GLOSSARY.md#section) that is not an array, or a [configuration object](../GLOSSARY.md#configuration-object)
that fails to hydrate.

`ConfigLifecycleException`, a `LogicException`, will be thrown for misuse of the [registry](../GLOSSARY.md#registry): a
method called in a phase that does not permit it, a module registering as `engine`, or a module identifier or
configuration name that contains a dot.

`ConfigNotRegisteredException`, a `RuntimeException`, will be thrown when `for()` is asked for a class that is not
[core configuration](../GLOSSARY.md#core-configuration).

### Out of scope

- **Where the files live, and constructing `ConfigPaths`.** Both belong to bootstrap.
- **Enabling and disabling modules.** Both belong to the command line.

## Alternatives considered

This design rests on [ADR-0010](../adr/0010-toml-configuration.md), which writes configuration in TOML rather
than PHP, JSON or YAML; [ADR-0003](../adr/0003-sealed-catalogues.md), which seals
a mutable [registry](../GLOSSARY.md#registry) into an immutable [catalogue](../GLOSSARY.md#catalogue) in phases; and
[ADR-0006](../adr/0006-environment-at-bootstrap.md), which reads
[environment variable](../GLOSSARY.md#environment-variable)s only during bootstrap.

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
