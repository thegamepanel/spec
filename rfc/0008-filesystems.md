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

The engine's files are reached through named filesystems, held in an immutable catalogue and resolved by name. Three
are given to the engine: configuration, data and cache. `Paths` keeps only the locations PHP includes from, and gains
a `compiled` root so that includable output no longer shares a directory with cached content. Configuration loading
moves onto the configuration filesystem.

## Motivation

Every file the engine reads or writes goes through Flysystem, per
[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md), but nothing provides the engine with a filesystem
to use. Configuration loading, the only part of the engine that touches a file, still reads through PHP's functions
from the paths in `Paths`.

`Paths` also has one root, `cache`, holding two different kinds of thing: compiled templates and module metadata,
which PHP includes and opcaches, and content that is merely cached. Only the first needs a real path, and while they
share a root the rule about where paths survive needs an exception.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Filesystem | A `League\Flysystem\FilesystemOperator` the engine reads and writes through, identified by name. |
| Adapter | What a filesystem stores through, such as the local disk or memory. |
| Given filesystem | A filesystem constructed by whatever composes the engine and handed to the catalogue. |
| Compiled output | A file the engine writes for PHP to include, such as a compiled template or module metadata. |

### Components

| Component | Responsibility |
|---|---|
| `FilesystemCatalogue` | Immutable. Holds the given filesystems and resolves them by name. |
| `Paths` | Holds the roots that remain paths, now including `compiled`. |
| `TomlLoader` | Reads configuration through the configuration filesystem. |
| `FilesystemException` | Marker contract implemented by every exception the component throws. |
| `UnknownFilesystemException` | Thrown when a name has no filesystem. |
| `FilesystemOperationException` | Wraps a Flysystem failure from an operation the engine performs. |

### The given filesystems

| Name | Adapter | Holds |
|---|---|---|
| `config` | Local | `config.toml`, `config.d/` and the module configuration files. |
| `data` | Local | The engine's and modules' data. |
| `cache` | Local | Cached content, and never includable output. |

All three are constructed by whatever composes the engine and passed to the catalogue fully formed. None is described
in configuration, and no adapter is built from configuration.

`data` and `cache` have no consumer in the engine yet. They are given because they are directories the engine already
names, and adding them later would mean revisiting every place a catalogue is constructed.

There is no composition root yet, so tests construct the filesystems, as they already construct `Paths`.

The given filesystems are constructed before the catalogue holding them, and one of them is needed before
configuration exists at all. Bootstrap therefore works in this order:

1. `Paths` is built from the arguments the panel is booted with.
2. Each given filesystem is constructed, rooted at the path it belongs to.
3. The configuration filesystem is handed to the configuration loader, and configuration is loaded.
4. The catalogue is constructed from the filesystems already built.

Nothing that loads configuration can therefore ask the catalogue for a filesystem: the catalogue does not exist yet.

### The catalogue

`FilesystemCatalogue` is constructed from filesystems keyed by name, and resolves them by name. It is the immutable
half of the pattern used elsewhere, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), with no mutable half, because
nothing registers a filesystem: a filesystem is either given when the catalogue is constructed or it does not exist.

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

`get()` returns Flysystem's `FilesystemOperator` itself, per
[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md), so no filesystem can hand out a path and every
guarantee of Flysystem's own path handling carries through.

Filesystems are `Process` lifetime, per [RFC-0007](0007-binding-lifetimes.md). A local adapter holds no connection
and is safe to reuse for the life of the worker. A remote adapter would hold one, so whatever adds the first remote
adapter decides its lifetime then.

### Paths

`Paths`, from [RFC-0005](0005-paths-value-object.md), gains a sixth root.

Three of its roots are paths because PHP itself reads from them:

| Root | Holds | Why it is a path |
|---|---|---|
| `modules` | The Composer project modules are installed into. | Composer discovery and autoloading include module classes. |
| `compiled` | Compiled templates and module metadata. | The engine writes them for PHP to include and opcache. |
| `logs` | Log files. | Monolog writes to them directly, and its handlers are already the abstraction over where a log goes. |

The other three exist so that the given filesystems have somewhere to be rooted, and nothing reads them afterwards:

| Root | Roots |
|---|---|
| `config` | The configuration filesystem. |
| `data` | The data filesystem. |
| `cache` | The cache filesystem. |

`compiled` has a method joining a relative path onto it, as the other roots do. Nothing in this design reads or
writes it: compiled templates and module metadata are its consumers, and each belongs to its own design.

Separating compiled output from cached content is what leaves the rule in
[ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md) without exceptions. `compiled` is a path because
PHP includes what is written there; `cache` is a filesystem because its contents are only ever read as content.

`Paths` is not bound into the container, and nothing resolves it at runtime. Bootstrap uses it to construct the
filesystems and to hand each of the three path consumers its directory.

### Configuration loading

`TomlLoader` reads through the configuration filesystem instead of the paths it takes today, which replaces how
[RFC-0004](0004-toml-configuration-loading.md) reads files. Everything else about loading is unchanged: the order
files are read in, how they are merged, and how environment variables are interpolated.

`TomlLoader` is constructed with the configuration filesystem, and `load()` takes nothing. It is given the filesystem
directly, not the catalogue, because configuration is loaded before the catalogue is constructed.

```php
$config = new Filesystem(new LocalFilesystemAdapter($paths->config));
$tree   = new TomlLoader($config)->load();
```

| Was | Becomes |
|---|---|
| Testing that a directory exists | `directoryExists()` |
| Listing `*.toml` in a directory | `listContents()`, filtered on the extension |
| Reading a file | `read()` |

The main configuration file, the drop-in directory and the module configuration directory are found at
`config.toml`, `config.d/` and `modules-enabled/` within the filesystem.

Both listings are sorted by name after they are read. `listContents()` promises no order at all, so the sorting is
what keeps drop-in precedence and module order the same whatever adapter is behind the filesystem.

A file that cannot be read throws `InvalidConfigException`, naming the file, as it does today.

### Errors

Every exception this component throws implements `FilesystemException`, the marker contract it defines, as
`ContainerException` and `ConfigException` do for their components. Flysystem defines a marker of the same name, and
the two are told apart by namespace.

| Exception | Thrown when |
|---|---|
| `UnknownFilesystemException` | A name has no given filesystem. |
| `FilesystemOperationException` | An operation the engine performs through a filesystem fails. It carries Flysystem's exception as its previous exception. |
| `InvalidConfigException` | A configuration file cannot be read, parsed or hydrated, as in [RFC-0004](0004-toml-configuration-loading.md). |

Where the engine reads or writes through a filesystem itself, a Flysystem failure is wrapped in an exception of the
component's own, so that a caller catches one marker. Configuration loading wraps further, into
`InvalidConfigException`.

A consumer given a `FilesystemOperator` calls Flysystem directly, and the exceptions it raises there are Flysystem's
own, including the path traversal and control character failures from its path normaliser.

### Out of scope

- **Filesystems described in configuration**, an adapter factory and a configuration section for them. Nothing in the
  engine declares a filesystem, so the first consumer that needs one brings them.
- **Remote adapters.** The first one arrives with a consumer that needs remote storage, along with the lifetime
  question it brings.
- **Mounting filesystems together.** Addressing filesystems by name from a string only makes sense when built for one
  holder from what that holder already has, and the first such holder is a module.
- **Module-scoped filesystems, path prefixing and sharing files between modules**, which belong to module resources.
- **Quotas and streaming files into responses.**
- **A filesystem for templates.** Templates are content, but there is no single root to hand in: sources are resolved
  as a chain, and which sources exist is only known once modules are discovered.
- **Creating directories.** Nothing here provisions anything.

## Alternatives considered

The decision this design rests on is recorded separately, with the alternatives it rejected:

- reaching every file through Flysystem, and keeping a path only where PHP includes, in
  [ADR-0021](../adr/0021-all-file-access-goes-through-flysystem.md)

**A mutable registry sealed into the catalogue**, as configuration and the container have. Nothing registers a
filesystem, so there would be nothing for a registry to collect.

**Keeping compiled output in `cache`.** The rule about where paths survive would then need an exception for one
directory that is both a path and a filesystem.

No other alternatives were weighed.

## Backwards compatibility

- `TomlLoader` reads through a filesystem rather than from paths, so it takes a `FilesystemOperator` in place of
  `Paths`.
- `Paths` gains `compiled`, so everything that constructs it supplies six roots instead of five.

## Open questions

## Changelog

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
