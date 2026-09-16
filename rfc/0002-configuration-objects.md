---
id: RFC-0002
title: Configuration objects
status: accepted
created: 2026-03-25
decided: 2026-03-29
backfilled: 2026-09-13
depends: [ADR-0003, ADR-0005, ADR-0006, RFC-0001]
updates: []
obsoletes: []
---

# RFC-0002: Configuration objects

## Abstract

A configuration component in which each configuration is an instance of a class unique to it, registered against a
module and a name, held in an immutable catalogue and injected by its class. Environment variables are available
during bootstrap through a static `Env` class, read from the process environment or from an env file, with typed
accessors that cast their values.

## Motivation

The panel needs a configuration system to configure the engine's components, and to provide additional
configuration for modules.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Configuration object | An instance of a class unique to one configuration, holding that configuration's values. |
| Module | The owner a configuration is registered against. The engine's own configuration is registered against a module like any other. |
| Configuration name | The name a configuration is registered under within its module. |
| Configuration catalogue | The immutable collection of every configuration object, looked up by module and name, or by class. |
| Environment variable | A value supplied through the process environment or an env file, read during bootstrap. |

### Components

| Component | Responsibility |
|---|---|
| `ConfigObject` | Contract every configuration object implements. |
| `ConfigCatalogue` | Immutable. Holds the configuration objects and looks them up by module and name, or by class. |
| `Env` | Static. Holds the environment variables during bootstrap and reads them, with typed accessors. |
| `ConfigException` | Marker contract implemented by every exception the component throws. |

### Configuration objects

Each configuration is an instance of a class unique to that configuration, per
[ADR-0005](../adr/0005-configuration-is-held-in-typed-objects.md). Its values are typed properties, and it may
contain child objects, but the configuration itself is contained within its class. Each configuration is registered
against a module and a name, and its class maps to exactly one module and name.

A configuration object implements `ConfigObject`, which requires `__set_state()`. PHP calls `__set_state()` when an
object exported with `var_export()` is restored, so configuration objects can be written to a cache and read back.

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

### Catalogue

`ConfigCatalogue` is immutable. It is constructed with every configuration object, keyed by module and then by name,
and builds a map from each object's class to its module and name as it is constructed. Nothing collects
registrations: the registry that seals into it arrives with [RFC-0004](0004-toml-configuration-loading.md).

| Method | Effect |
|---|---|
| `get(string $module, string $config): ?ConfigObject` | Returns the configuration object registered under the module and name, or `null`. |
| `has(string $module, string $config): bool` | Returns whether a configuration object is registered under the module and name. |
| `for(string $class): ?ConfigObject` | Returns the configuration object of the given class, or `null`. |

```php
$mail = $catalogue->get('engine', 'mail');
$mail = $catalogue->for(MailConfig::class);
```

### Injection

Because each configuration object has a class of its own, a component receives the configuration it needs by type.
The config component binds every configuration object into the container as a shared instance under its own class
name, so a parameter typed as a configuration class is resolved to that object through an ordinary binding, as
described in [RFC-0001](0001-dependency-injection-container.md).

```php
public function __construct(
    private MailConfig $mail,
) {}
```

### Environment variables

`Env` holds environment variables for the length of bootstrap, per
[ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md). It is a static singleton, initialised
once, from one of two sources:

| Method | Effect |
|---|---|
| `Env::createFromSuperglobal()` | Initialises `Env` with the values in the `$_ENV` superglobal. |
| `Env::createFromFile(string $path)` | Initialises `Env` with the values parsed from the `.env` file in the directory at the path. The file's values are held by `Env` alone and are not written into `$_ENV`. |
| `Env::destroy()` | Removes the instance, and with it the values read from an env file. |

Initialising `Env` a second time throws, and so does reading from it before it has been initialised. Once bootstrap
is over, `Env` is destroyed and nothing reads environment variables directly.

Values are read with these accessors. Each typed accessor returns the default when the variable is missing or
`null`, and throws when the value cannot be cast to its type.

| Accessor | Behaviour |
|---|---|
| `get(string $key, $default = null)` | Returns the value, or the default when the variable is missing or `null`. |
| `has(string $key)` | Returns whether the variable is present, including when its value is `null`. |
| `string(string $key, ?string $default = null)` | Casts a string, number or boolean to a string. |
| `int(string $key, ?int $default = null)` | Casts an integer, numeric string or boolean to an integer. |
| `float(string $key, ?float $default = null)` | Casts a float, numeric string or boolean to a float. |
| `bool(string $key, ?bool $default = null)` | Casts a boolean or integer to a boolean. The strings `true`, `1` and `yes` are `true`, and `false`, `0` and `no` are `false`. Any other string throws. |

```php
Env::createFromFile('/path/to/directory');

$port  = Env::int('MAIL_PORT', 25);
$debug = Env::bool('APP_DEBUG', false);

Env::destroy();
```

### Errors

Every exception implements `ConfigException`.

| Exception | Thrown when |
|---|---|
| `EnvInitialisationException` | `Env` is read before it has been initialised, or initialised a second time. |
| `InvalidEnvException` | A typed accessor's value cannot be cast to its type. |

### Out of scope

- **Loading configuration.** Reading configuration files and building the catalogue from them belongs to bootstrap.
- **Caching.** Writing configuration objects to a cache and restoring them from it belongs to whatever loads
  configuration.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- holding configuration in typed objects, in [ADR-0005](../adr/0005-configuration-is-held-in-typed-objects.md)
- reading environment variables only during bootstrap, in
  [ADR-0006](../adr/0006-environment-variables-are-only-read-during-bootstrap.md)
- the immutable catalogue, in [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)

No other alternatives were weighed.

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
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), decided on 2026-04-22,
  postdate this design.

[#22]: https://github.com/thegamepanel/panel/issues/22
[#25]: https://github.com/thegamepanel/panel/pull/25
[180794a]: https://github.com/thegamepanel/panel/commit/180794a
