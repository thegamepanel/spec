---
id: ADR-0021
title: Route all file access through Flysystem
status: accepted
created: 2026-09-03
decided: 2026-09-03
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0021: Route all file access through Flysystem

## Context

The [engine](../GLOSSARY.md#engine) reads files. Configuration loading is the only part of it that touches one today,
through PHP's own functions and a path taken from `Paths`. More will follow as components land: cached content, the
engine's own data, and the compiled output that PHP includes, which the engine will write as well as read.

A path only means anything where the files are on local disk. Some of what the engine keeps
could sit elsewhere, such as configuration in object storage or cached content in a database table, where no host path
exists. Anything that hands out a path lets its consumers come to depend on local disk, and nothing but review would
notice.

PHP's `include` is the exception. A file that is included and opcached has to be a real file at a real path.

## Decision

I will route every file the [engine](../GLOSSARY.md#engine) reads or writes through Flysystem. Filesystems will be
constructed by whatever composes the engine and handed in fully formed, nothing will expose a filesystem path at
runtime, and consumers will be given Flysystem's own `FilesystemOperator` rather than a wrapper the engine owns. A path
will be kept only where PHP's own `include` is involved: the modules directory, for Composer-based discovery and
autoloading; the compiled directory, for compiled templates and [module](../GLOSSARY.md#module) metadata; and the log
directory, which is where Monolog writes.

## Alternatives

**PHP's file functions with a path**, as the [engine](../GLOSSARY.md#engine) uses today. A consumer could come to depend
on local disk, and a filesystem backed by object storage or a database table would have no path to give it. I rejected
this because substitutability would rest on review rather than on the type.

**An interface the engine owns, wrapping Flysystem.** I rejected this because it would cost twenty-one delegating
methods, kept in step with a library that already has them, and because [module](../GLOSSARY.md#module) authors work
with filesystems directly, so the type they already know is the one to expose.

Whether any other library was weighed against Flysystem is not recorded.

## Consequences

Easier:

- A filesystem will be backed by anything Flysystem supports without any consumer noticing, because no consumer can
  obtain a path. Configuration will be loadable with every filesystem in memory, touching no disk at all.
- Flysystem's own guarantees will carry through: its path normaliser rejects path traversal and control characters, and
  its `FilesystemException` is a marker interface, which is the convention the [engine](../GLOSSARY.md#engine)'s
  components already follow.
- [Module](../GLOSSARY.md#module) authors will meet a type they already know.

Harder:

- Every filesystem will have to be constructed by whatever composes the engine and handed in.
  There is no composition root yet, so until there is, tests will construct them, as they already construct `Paths`.
- Anything that wants a real path will have to justify itself against the `include` rule. Compiled output will be
  kept apart from cached content so that the rule needs no exceptions.

Constrained:

- `league/flysystem` will become a dependency of the binary, with `league/flysystem-local` and
  `league/mime-type-detection` behind it, and an adapter for each backend it uses.
- Local paths will remain in three places only: modules, compiled output and logs.
- A remote adapter holds a connection, so whatever adds the first one will inherit the question of how long a
  filesystem lives.

## Sources

- Issue [#71], Engine - Filesystems, 2026-09-03 and not edited since: the decision, which is both `created` and
  `decided`. It records every file going through Flysystem, filesystems handed in fully formed, nothing exposing a
  path at runtime, `FilesystemOperator` returned rather than a wrapper and why, the three places a path is kept and
  why each is one, Flysystem's weight and its guarantees, and the lifetime question a remote adapter would bring.
- Issue [#72], Engine - Filesystems - Catalogue, 2026-09-03, edited on 2026-09-13 only to replace names with issue
  links: the same reasoning for returning `FilesystemOperator`, and Flysystem's guarantees carrying through
  unchanged.
- Issue [#74], Engine - Filesystems - TomlLoader migration, 2026-09-03, edited on 2026-09-13 only to replace names
  with issue links: loading configuration with every filesystem in memory and no disk access as the property that
  proves nothing depends on a local path.

[#71]: https://github.com/thegamepanel/panel/issues/71
[#72]: https://github.com/thegamepanel/panel/issues/72
[#74]: https://github.com/thegamepanel/panel/issues/74
