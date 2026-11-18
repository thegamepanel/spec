---
id: RFC-0002
title: Configuration objects
status: accepted
created: 2026-03-25
decided: 2026-03-29
backfilled: 2026-11-18
depends: [ADR-0005, ADR-0006, RFC-0001]
updates: []
obsoletes: []
---

# RFC-0002: Configuration objects

## Abstract

I will give the panel a configuration component in which each configuration is an object of a class unique to it,
registered against a [module](../GLOSSARY.md#module) and a name, held in an immutable catalogue and injected by its
class. [Environment variable](../GLOSSARY.md#environment-variable)s will be available during bootstrap through a static
`Env` class, read from the process environment or from an env file, with typed accessors that cast their values.

## Motivation

The panel needs a configuration system to configure the [engine](../GLOSSARY.md#engine)'s components, and to provide
additional configuration for [module](../GLOSSARY.md#module)s.

## Proposal

### One class for each configuration

Each configuration will be a [configuration object](../GLOSSARY.md#configuration-object): an instance of a class unique
to that configuration, as [ADR-0005](../adr/0005-typed-configuration.md) decides. Its values will be
typed properties, and it may contain child objects, but the configuration itself will be contained within its class.
Each configuration will be registered against a [module](../GLOSSARY.md#module) and a name, and its class will map to
exactly one module and name. The engine's own configuration will be registered against a module like any other.

Every configuration object will implement `ConfigObject`, which requires `__set_state()`. PHP calls `__set_state()`
when it restores an object exported with `var_export()`, so a configuration object can be written to a cache and read
back from it.

```php
final readonly class MailConfig implements ConfigObject
{
    public function __construct(
        public string $host,
        public int $port,
    ) {}

    public static function __set_state(array $data): static
    {
        return new static($data['host'], $data['port']);
    }
}
```

### Finding a configuration

`ConfigCatalogue` will hold every [configuration object](../GLOSSARY.md#configuration-object) and will be immutable. It
will be constructed with the objects keyed by [module](../GLOSSARY.md#module) and then by name, and will build a map
from each object's class to its module and name as it is constructed. Nothing will collect registrations: the registry
that seals into the catalogue arrives with [RFC-0004](0004-toml-configuration-loading.md).

A configuration will be found in one of two ways. `get(string $module, string $config): ?ConfigObject` will return the
object registered under a module and name, and `has()` will say whether there is one.
`for(string $class): ?ConfigObject` will return the object of a given class. Both lookups will return `null` when there
is nothing to find.

```php
$mail = $catalogue->get('engine', 'mail');
$mail = $catalogue->for(MailConfig::class);
```

### Injection

Because each [configuration object](../GLOSSARY.md#configuration-object) has a class of its own, a component will
receive the configuration it needs by its type. The configuration component will bind every configuration object into
the container as a shared instance under its own class name, so that a parameter typed as a configuration class resolves
to that object through an ordinary binding, as described in [RFC-0001](0001-dependency-injection-container.md).

```php
public function __construct(
    private MailConfig $mail,
) {}
```

### Environment variables

`Env` will hold [environment variable](../GLOSSARY.md#environment-variable)s for the length of bootstrap, as
[ADR-0006](../adr/0006-environment-at-bootstrap.md) decides. It will be a static singleton.

At the start of bootstrap, `Env` will be initialised once, from one of two sources: the `$_ENV` superglobal, through
`Env::createFromSuperglobal()`, or the `.env` file in a directory, through `Env::createFromFile(string $path)`. Values
read from a file will be held by `Env` alone and never written into `$_ENV`. Initialising it a second time will throw.

While configuration is built, values will be read from it, and reading before it has been initialised will throw.
`get()` will return a variable's value, or a default when the variable is missing or `null`, and `has()` will say
whether a variable is present, including when its value is `null`. Four typed accessors will read a value as a
particular type. Each takes the key and an optional default of that type, which is `null` unless one is given. It
returns the default when the variable is missing or `null`, so an accessor called without a default returns `null`, and
it throws when the value cannot be cast:

| Accessor | Casts |
|---|---|
| `string()` | A string, number or boolean. |
| `int()` | An integer, numeric string or boolean. |
| `float()` | A float, numeric string or boolean. |
| `bool()` | A boolean or integer. The strings `true`, `1` and `yes` are `true`, and `false`, `0` and `no` are `false`. Any other string throws. |

```php
Env::createFromFile('/path/to/directory');

$port  = Env::int('MAIL_PORT', 25);
$debug = Env::bool('APP_DEBUG', false);

Env::destroy();
```

Once bootstrap is over, `Env::destroy()` will remove the instance, and with it any values read from a file. Nothing will
read environment variables directly after that.

### Errors

Every exception the component throws will implement `ConfigException`, a marker contract. Reading `Env` before it has
been initialised, or initialising it a second time, will throw `EnvInitialisationException`. A typed accessor whose
value cannot be cast to its type will throw `InvalidEnvException`.

### Out of scope

- **Loading configuration.** Reading configuration files and building the catalogue from them belongs to bootstrap.
- **Caching.** Writing [configuration object](../GLOSSARY.md#configuration-object)s to a cache and restoring them from
  it belongs to whatever loads configuration.

## Alternatives considered

This design rests on [ADR-0005](../adr/0005-typed-configuration.md), which holds configuration in
typed objects, and [ADR-0006](../adr/0006-environment-at-bootstrap.md), which reads
[environment variable](../GLOSSARY.md#environment-variable)s only during bootstrap.

## Backwards compatibility

Nothing breaks. No configuration component exists before this change.

## Open questions

## Changelog

## Sources

- Issue [#22], Engine - Config, 2026-03-25 and not edited since: the motivation, configuration objects registered
  against a name and a module, environment variables from `$_ENV` or a local env file, and injection because each
  configuration is a unique class. This is `created`.
- PR [#25], feat(config): Add config component, merged 2026-03-29 and squashed as [180794a]: the implementation,
  whose merge is `decided`. `ConfigObject`, `ConfigCatalogue`, `Env` and the exceptions are described from
  [`src/Config` at 180794a](https://github.com/thegamepanel/panel/tree/180794a/src/Config) and
  [`tests/Unit/Config` at 180794a](https://github.com/thegamepanel/panel/tree/180794a/tests/Unit/Config).
- Loading configuration belonging to bootstrap, the config component binding each configuration object into the
  container as a shared instance under its class name, every typed `Env` accessor throwing when a value cannot be
  cast, and `Env` existing only during bootstrap: first written down on 2026-09-14. The code at [180794a] differs on
  three of these: nothing binds configuration objects into the container, `Env::string()` returns the default
  instead of throwing, and nothing confines `Env` to bootstrap. It also defines a
  `MissingEnvVariableException` that is not part of the design.
- The catalogue being described as immutable rather than as the sealed half of a registry pattern: corrected on
  2026-09-16. PR [#25]'s commits show the class arrived as a configuration registry and was renamed to a catalogue,
  and neither a registry nor a seal exists at [180794a]. Both the registry and the pattern in
  [ADR-0003](../adr/0003-sealed-catalogues.md), decided on 2026-04-22,
  postdate this design.

[#22]: https://github.com/thegamepanel/panel/issues/22
[#25]: https://github.com/thegamepanel/panel/pull/25
[180794a]: https://github.com/thegamepanel/panel/commit/180794a
