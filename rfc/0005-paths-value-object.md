---
id: RFC-0005
title: Paths value object
status: accepted
created: 2026-06-01
decided: 2026-07-04
backfilled: 2026-09-14
depends: [ADR-0012]
updates: [RFC-0004]
obsoletes: []
---

# RFC-0005: Paths value object

## Abstract

A single `Paths` value object holds every filesystem location the engine needs: configuration, data, modules, cache
and logs. Bootstrap constructs it once, from the arguments the panel is booted with, and it replaces the
configuration component's `ConfigPaths`.

## Motivation

The configuration loader receives its locations through `ConfigPaths`, which holds configuration locations and
nothing else. More components need locations of their own: modules, the cache and logs. A single object holding
every location, constructed once during early bootstrap, is cleaner than a paths object for each component.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Root | An absolute directory under which the engine keeps one type of file. |
| Relative path | A path beneath a root, joined onto it by `Paths`. |

### Components

| Component | Responsibility |
|---|---|
| `Paths` | Final readonly value object holding every root, and joining relative paths onto them. |

### Roots

| Property | Holds |
|---|---|
| `config` | The configuration directory, holding `config.toml`, `config.d/` and `modules-enabled/`. |
| `data` | The application data directory. |
| `modules` | The Composer project that external modules are installed into, per [ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md). |
| `cache` | The cache directory. |
| `logs` | The log directory. |

Each root is an absolute path. `Paths` holds the roots and does nothing else with them: it does not work out where
they are, check that they exist, or create them.

```php
$paths = new Paths(
    config: '/etc/tgp',
    data: '/var/lib/tgp',
    modules: '/var/lib/tgp/modules',
    cache: '/var/lib/tgp/cache',
    logs: '/var/log/tgp',
);
```

### Joining paths

Each root has a method of the same name that joins a relative path onto it:

| Method | Returns |
|---|---|
| `config(string $path): string` | The path beneath the configuration directory. |
| `data(string $path): string` | The path beneath the data directory. |
| `modules(string $path): string` | The path beneath the modules directory. |
| `cache(string $path): string` | The path beneath the cache directory. |
| `logs(string $path): string` | The path beneath the log directory. |

The root's trailing separator and the path's leading separator are removed, and the two are joined with a single
separator. A path passed to a method is always treated as relative to the root, so `$paths->config('config.d')` and
`$paths->config('/config.d')` both return `/etc/tgp/config.d`.

### Construction

Bootstrap constructs `Paths` once, during early bootstrap, before configuration is loaded or modules are discovered.
Its roots come from arguments given when the panel is booted, which are passed in during bootstrapping.

### Configuration loading

`TomlLoader::load()` takes `Paths` in place of `ConfigPaths`, and reads from the configuration directory:
`config.toml`, `config.d/` and `modules-enabled/` beneath `$paths->config`. `ConfigPaths` is removed.

```php
$tree = new TomlLoader()->load($paths);
```

### Out of scope

- **Populating the roots.** How bootstrap takes the roots from the arguments the panel is booted with, and resolves
  them, belongs to bootstrapping.

## Alternatives considered

**A paths object for each component**, of which `ConfigPaths` is the existing one. Once modules, the cache and logs
need locations as well as configuration, a single object holding every root, constructed once during early
bootstrap, is cleaner.

No other alternatives were weighed.

## Backwards compatibility

- `ConfigPaths` is removed, and `TomlLoader::load()` takes `Paths` instead.
- The main configuration file is always `config.toml` in the configuration directory, and the drop-in and module
  configuration directories are always `config.d/` and `modules-enabled/` beside it. `ConfigPaths` held each of the
  three as a path of its own.

## Open questions

## Changelog

## Sources

- Issue [#34], Engine - Paths Value Object, 2026-06-01: the motivation, the single object and the per-component
  objects it replaces, the five roots, construction during early bootstrap before configuration is loaded or modules
  are discovered, and configuration loading reading from `$paths->config`. This is `created`. Its tasks, ticked on
  2026-07-04, include resolving the roots in layers from compiled defaults, environment variables and command-line
  flags, and testing that resolution. Nothing at [a50a9ab] resolves the roots, and that resolution is left to
  bootstrapping.
- PR [#40], refactor(engine:config): Refactor ConfigPaths VO to Paths, merged 2026-07-04 and squashed as [a50a9ab]:
  the implementation, whose merge is `decided`. `Paths`, its methods and `TomlLoader`'s use of it are described from
  [`src/Config/Paths.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/src/Config/Paths.php),
  [`src/Config/TomlLoader.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/src/Config/TomlLoader.php)
  and
  [`tests/Unit/Config/ConfigPathsTest.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/tests/Unit/Config/ConfigPathsTest.php).
- The roots coming from arguments given when the panel is booted, and populating and resolving them belonging to
  bootstrapping: first written down on 2026-09-14.

[#34]: https://github.com/thegamepanel/panel/issues/34
[#40]: https://github.com/thegamepanel/panel/pull/40
[a50a9ab]: https://github.com/thegamepanel/panel/commit/a50a9ab
