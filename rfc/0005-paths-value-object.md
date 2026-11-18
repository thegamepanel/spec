---
id: RFC-0005
title: Paths value object
status: accepted
created: 2026-06-01
decided: 2026-07-04
backfilled: 2026-11-18
depends: [ADR-0012]
updates: [RFC-0004]
obsoletes: []
---

# RFC-0005: Paths value object

## Abstract

The [engine](../GLOSSARY.md#engine) needs its files in five places: configuration, application data, installed
[module](../GLOSSARY.md#module)s, the cache and logs. I will give it one immutable object, `Paths`, that knows all five
and builds paths beneath them. Bootstrap will create it once, and it will replace `ConfigPaths`, which knows only where
configuration lives.

## Motivation

The configuration loader receives its locations through `ConfigPaths`, which holds configuration locations and nothing
else. As more of the [engine](../GLOSSARY.md#engine) needs locations of its own, [module](../GLOSSARY.md#module)s, the
cache and logs each need a directory as well. A single object holding every location, filled in once early in bootstrap,
is cleaner than a paths object for each part of the engine that needs one.

## Proposal

### Roots

Each type of file the [engine](../GLOSSARY.md#engine) keeps lives under a [root](../GLOSSARY.md#root), an absolute
directory. There will be five:

- **Configuration**, holding `config.toml`, the `config.d/` [drop-in](../GLOSSARY.md#drop-in) directory and
  `modules-enabled/`.
- **Data**, for the application's own data.
- **Modules**, the Composer project that external modules are installed into, as
  [ADR-0012](../adr/0012-composer-installs.md) decides.
- **Cache**.
- **Logs**.

`Paths` will hold these five and nothing more. It will be a final, readonly value object that is handed absolute
paths and takes them as given: it will not work out where the directories are, check that they exist or create them.
Producing absolute paths is bootstrap's responsibility.

```php
$paths = new Paths(
    config: '/etc/tgp',
    data: '/var/lib/tgp',
    modules: '/var/lib/tgp/modules',
    cache: '/var/lib/tgp/cache',
    logs: '/var/log/tgp',
);
```

### Building a path

Each [root](../GLOSSARY.md#root) will have a method named after it, such as `config(string $path): string`, which takes
a [relative path](../GLOSSARY.md#relative-path) and returns it joined onto that root. The root's trailing separator and
the path's leading separator will be trimmed and a single separator put between them, so the path is always read as
relative, with or without a leading slash:

```php
$paths->config('config.d');   // /etc/tgp/config.d
$paths->config('/config.d');  // /etc/tgp/config.d
```

### Creating it

Bootstrap will create `Paths` once, early, before configuration is loaded or [module](../GLOSSARY.md#module)s are
discovered. The [root](../GLOSSARY.md#root)s will come from the arguments the panel is booted with.

### Loading configuration

The configuration loader will take `Paths` in place of `ConfigPaths`, and find `config.toml`, `config.d/` and
`modules-enabled/` beneath the configuration [root](../GLOSSARY.md#root). Their names will be fixed relative to that
root rather than each given separately. `ConfigPaths` will be removed.

```php
$tree = new TomlLoader()->load($paths);
```

### Out of scope

- **Populating the roots.** How bootstrap takes the [root](../GLOSSARY.md#root)s from the arguments the panel is booted
  with, and resolves them, belongs to bootstrapping.

## Alternatives considered

This design rests on [ADR-0012](../adr/0012-composer-installs.md), which installs
external modules into a Composer project of their own and so gives modules a
[root](../GLOSSARY.md#root).

**A paths object for each component.** `ConfigPaths` would stay as it is, and modules, the cache and logs would each
gain an object of their own describing their locations. I rejected this because once more than configuration needs a
location, one object holding every root, created once early in bootstrap, is cleaner.

## Backwards compatibility

- `ConfigPaths` is removed, and `TomlLoader::load()` takes `Paths` instead.
- The main configuration file is always `config.toml` in the configuration directory, and the drop-in and module
  configuration directories are always `config.d/` and `modules-enabled/` beside it. `ConfigPaths` held each of the
  three as a path of its own.

## Open questions

## Changelog

## Sources

- Issue [#34], Engine - Paths Value Object, 2026-06-01: the motivation, the single object and the per-component
  objects it replaces, the five roots and their default locations, creation during early bootstrap before
  configuration is loaded or modules are discovered, and configuration loading reading from `$paths->config`. This is
  `created`. Its tasks, ticked on 2026-07-04, include resolving the roots in layers from compiled defaults,
  environment variables and command-line flags, and testing that resolution. Nothing at [a50a9ab] resolves the
  roots, and that resolution is left to bootstrapping.
- PR [#40], refactor(engine:config): Refactor ConfigPaths VO to Paths, merged 2026-07-04 and squashed as [a50a9ab]: the
  implementation, whose merge is `decided`. `Paths`, its methods and `TomlLoader`'s use of it are described from
  [`src/Config/Paths.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/src/Config/Paths.php), whose
  docblock states that bootstrap is responsible for producing absolute paths,
  [`src/Config/TomlLoader.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/src/Config/TomlLoader.php)
  and
  [`tests/Unit/Config/ConfigPathsTest.php` at a50a9ab](https://github.com/thegamepanel/panel/blob/a50a9ab/tests/Unit/Config/ConfigPathsTest.php).
- The roots coming from arguments given when the panel is booted, and populating and resolving them belonging to
  bootstrapping: first written down on 2026-09-14.

[#34]: https://github.com/thegamepanel/panel/issues/34
[#40]: https://github.com/thegamepanel/panel/pull/40
[a50a9ab]: https://github.com/thegamepanel/panel/commit/a50a9ab
