---
id: RFC-0008
title: Filesystems
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0003, ADR-0021, RFC-0007]
updates: [RFC-0004, RFC-0005]
obsoletes: []
---

# RFC-0008: Filesystems

## Abstract

We will reach the [engine](../GLOSSARY.md#engine)'s files through named filesystems, held in an immutable
[catalogue](../GLOSSARY.md#catalogue) that resolves them by name, and give the engine three of them: configuration, data
and cache. `Paths` will keep only the locations PHP includes from, and will gain a `compiled`
[root](../GLOSSARY.md#root) so that includable output no longer shares a directory with cached content. Configuration
loading will move onto the configuration filesystem.

## Motivation

[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md) requires every file the
[engine](../GLOSSARY.md#engine) reads or writes to go through Flysystem, but nothing provides the engine with a
filesystem to use. Configuration loading, the only part of the engine that touches a file, still reads through PHP's
functions from the paths in `Paths`.

`Paths` also has one [root](../GLOSSARY.md#root), `cache`, holding two different types of file. Compiled templates and
[module](../GLOSSARY.md#module) metadata are included by PHP and opcached; other content is merely cached. Only the
first type needs a real path, and while the two share a root, the rule about where paths survive needs an exception.

## Proposal

A filesystem will be a Flysystem `FilesystemOperator` that the [engine](../GLOSSARY.md#engine) reads and writes through,
identified by name. It will store its files through an adapter, such as the local disk or memory.

### The given filesystems

The [engine](../GLOSSARY.md#engine) will be given three filesystems:

| Name | Adapter | Holds |
|---|---|---|
| `config` | Local | `config.toml`, `config.d/` and the module configuration files. |
| `data` | Local | The engine's and modules' data. |
| `cache` | Local | Cached content, and never includable output. |

Each [given filesystem](../GLOSSARY.md#given-filesystem) will be constructed by whatever composes the engine and passed
to the catalogue fully formed. None is described in configuration, and no adapter is built from configuration.

`data` and `cache` have no consumer in the engine yet. They will be given because they are directories the engine
already names, and adding them later would mean revisiting every place a catalogue is constructed.

There is no composition root yet, so tests will construct the filesystems, as they already construct `Paths`.

The given filesystems will be constructed before the catalogue holding them, and one of them will be needed before
configuration exists at all. Bootstrap will therefore work in this order:

1. `Paths` is built from the arguments the panel is booted with.
2. Each given filesystem is constructed, rooted at the path it belongs to.
3. The configuration filesystem is handed to the configuration loader, and configuration is loaded.
4. The catalogue is constructed from the filesystems already built.

Nothing that loads configuration will therefore be able to ask the catalogue for a filesystem: the catalogue will not
exist yet.

### The catalogue

`FilesystemCatalogue` will be constructed from filesystems keyed by name, and will resolve them by name. It will be the
immutable half of the [registry](../GLOSSARY.md#registry) and [catalogue](../GLOSSARY.md#catalogue) pattern from
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), with no mutable half, because
nothing will register a filesystem: a filesystem will either be given when the catalogue is constructed or not exist.

```php
$filesystems = new FilesystemCatalogue([
    'config' => $config,
    'data'   => new Filesystem(new LocalFilesystemAdapter($paths->data)),
    'cache'  => new Filesystem(new LocalFilesystemAdapter($paths->cache)),
]);

$data = $filesystems->get('data');
```

| Method | Effect |
|---|---|
| `get(string $name): FilesystemOperator` | Returns the filesystem given under the name, and throws when there is none. |
| `has(string $name): bool` | Returns whether a filesystem is given under the name. |

`get()` will return Flysystem's `FilesystemOperator` itself, as
[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md) requires, so no filesystem can hand out a path and
every guarantee of Flysystem's own path handling carries through.

Filesystems will have the `Process` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md). A
local adapter holds no connection and is safe to reuse for the life of the worker. A remote adapter would hold one, so
whatever adds the first remote adapter will decide its lifetime then.

### Paths

`Paths`, from [RFC-0005](0005-paths-value-object.md), will gain a sixth [root](../GLOSSARY.md#root).

Three of its roots will be paths because PHP itself reads from them:

| Root | Holds | Why it is a path |
|---|---|---|
| `modules` | The Composer project modules are installed into. | Composer discovery and autoloading include module classes. |
| `compiled` | Compiled templates and module metadata. | The engine writes them for PHP to include and opcache. |
| `logs` | Log files. | Monolog writes to them directly, and its handlers are already the abstraction over where a log goes. |

The other three will exist so that each [given filesystem](../GLOSSARY.md#given-filesystem) has somewhere to be rooted,
and nothing will read them afterwards: `config` will root the configuration filesystem, `data` the data filesystem and
`cache` the cache filesystem.

`compiled` will have a method joining a [relative path](../GLOSSARY.md#relative-path) onto it, as the other roots do.
Nothing in this design will read or write it: compiled templates and [module](../GLOSSARY.md#module) metadata are its
consumers, and each belongs to its own design.

Separating [compiled output](../GLOSSARY.md#compiled-output) from cached content is what will leave the rule in
[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md) without exceptions. `compiled` will be a path because
PHP includes what is written there; `cache` will be a filesystem because its contents are only ever read as content.

`Paths` will not be bound into the container, and nothing will resolve it at runtime. Bootstrap will use it to construct
the filesystems and to hand each of the three path consumers its directory.

### Configuration loading

`TomlLoader` will read through the configuration filesystem instead of the paths it takes today, replacing how
[RFC-0004](0004-toml-configuration-loading.md) reads files. Everything else about loading will be unchanged: the order
files are read in, how they are merged, and how each [environment variable](../GLOSSARY.md#environment-variable) is
interpolated.

`TomlLoader` will be constructed with the configuration filesystem, and `load()` will take nothing. It will be given the
filesystem directly, not the catalogue, because configuration will be loaded before the catalogue is constructed.

```php
$config = new Filesystem(new LocalFilesystemAdapter($paths->config));
$tree   = new TomlLoader($config)->load();
```

| Was | Becomes |
|---|---|
| Testing that a directory exists | `directoryExists()` |
| Listing `*.toml` in a directory | `listContents()`, filtered on the extension |
| Reading a file | `read()` |

The main configuration file, the [drop-in](../GLOSSARY.md#drop-in) directory and the [module](../GLOSSARY.md#module)
configuration directory will be found at `config.toml`, `config.d/` and `modules-enabled/` within the filesystem.

Both listings will be sorted by name after they are read. `listContents()` promises no order at all, so the sorting is
what will keep drop-in precedence and module order the same whatever adapter is behind the filesystem.

A file that cannot be read will throw `InvalidConfigException`, naming the file, as it does today.

### Errors

Every exception this component throws will implement `FilesystemException`, a marker contract the component defines,
as `ContainerException` and `ConfigException` do for their components. Flysystem
defines a marker of the same name, and the two are told apart by namespace.

| Exception | Thrown when |
|---|---|
| `UnknownFilesystemException` | A name has no given filesystem. |
| `FilesystemOperationException` | An operation the engine performs through a filesystem fails. It carries Flysystem's exception as its previous exception. |
| `InvalidConfigException` | A configuration file cannot be read, parsed or hydrated, as in [RFC-0004](0004-toml-configuration-loading.md). |

Where the [engine](../GLOSSARY.md#engine) reads or writes through a filesystem itself, a Flysystem failure will be
wrapped in an exception of the component's own, so that a caller catches one marker. Configuration loading will wrap
further, into `InvalidConfigException`.

A consumer given a `FilesystemOperator` will call Flysystem directly, and the exceptions it raises there will be
Flysystem's own, including the path traversal and control character failures from its path normaliser.

### Out of scope

- **Filesystems described in configuration**, an adapter factory and a configuration section for them. Nothing in the
  [engine](../GLOSSARY.md#engine) declares a filesystem, so the first consumer that needs one brings them.
- **Remote adapters.** The first one arrives with a consumer that needs remote storage, along with the lifetime
  question it brings.
- **Mounting filesystems together.** Addressing filesystems by name from a string only makes sense when built for one
  holder from what that holder already has, and the first such holder is a [module](../GLOSSARY.md#module).
- **Module-scoped filesystems, path prefixing and sharing files between modules**, which belong to the design of module
  resources.
- **Quotas and streaming files into responses.**
- **A filesystem for templates.** Templates are content, but there is no single root to hand in: sources are resolved
  as a chain, and which sources exist is only known once modules are discovered.
- **Creating directories.** Nothing here provisions anything.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- reaching every file through Flysystem, and keeping a path only where PHP includes, in
  [ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md)

**A mutable registry sealed into the catalogue, as configuration and the container have.** We rejected this because
nothing registers a filesystem, so there would be nothing for a registry to collect.

**Keeping compiled output in `cache`.** We rejected this because the rule about where paths survive would then need an
exception for one directory that is both a path and a filesystem.

No other alternatives were weighed.

## Backwards compatibility

- `TomlLoader` will read through a filesystem rather than from paths, so it will take a `FilesystemOperator` in place
  of `Paths`.
- `Paths` will gain `compiled`, so everything that constructs it will supply six roots instead of five.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

## Sources

- Issue [#71], Engine - Filesystems, 2026-09-03 and not edited since: every file going through Flysystem, the three
  given filesystems and their adapters, filesystems handed in fully formed with no configuration and no registry,
  `FilesystemOperator` returned rather than a wrapper, the roots that stay paths and why each one does, `compiled`
  splitting includable output from cached content, `Paths` staying unbound, process lifetime and the remote adapter
  question, and every boundary listed under Out of scope.
- Issue [#72], Engine - Filesystems - Catalogue, 2026-09-03, edited on 2026-09-13 only to replace names with issue
  links: `FilesystemCatalogue` built from filesystems keyed by name, resolution by name, throwing on an unknown name,
  `data` and `cache` given without consumers, and construction by tests while there is no composition root.
- Issue [#73], Engine - Filesystems - Paths: compiled, 2026-09-03, edited on 2026-09-13 only to replace names with
  issue links: `compiled` as a sixth root with a method like its siblings, its consumers being elsewhere, and
  `Paths` remaining unbound.
- Issue [#74], Engine - Filesystems - TomlLoader migration, 2026-09-03, edited on 2026-09-13 only to replace names
  with issue links: the calls that change, sorting kept because `listContents()` promises no order, a read failure
  mapping onto the existing exception, and loading configuration with every filesystem in memory.
- `FilesystemCatalogue::has()`, the name `UnknownFilesystemException`, the component's own `FilesystemException`
  marker with Flysystem's failures wrapped where the engine performs the operation, `TomlLoader` being constructed
  with the configuration filesystem itself, and the order bootstrap constructs paths, filesystems, configuration and
  the catalogue in: first written down on 2026-09-16.

[#71]: https://github.com/thegamepanel/panel/issues/71
[#72]: https://github.com/thegamepanel/panel/issues/72
[#73]: https://github.com/thegamepanel/panel/issues/73
[#74]: https://github.com/thegamepanel/panel/issues/74
