---
id: ADR-0021
title: All file access goes through Flysystem
status: accepted
created: 2026-09-03
decided: 2026-09-03
backfilled: 2026-09-16
depends: []
updates: []
obsoletes: []
---

# ADR-0021: All file access goes through Flysystem

## Context

The engine reads and writes files: configuration, cached content, its own data, and the compiled output that PHP
includes. Everything that touches a file does so through PHP's own functions and a path taken from `Paths`.

A path only means anything where the files are on local disk. Some of what the engine keeps could sit elsewhere:
configuration in object storage, or cached content in a database table, where no host path exists. Anything that
hands out a path lets its consumers depend on local disk, and nothing but review would notice.

PHP's `include` is the exception. An includable, opcached file has to be a real file at a real path.

## Decision

Every file the engine reads or writes goes through Flysystem. Filesystems are constructed by whatever composes the
engine and handed in fully formed, and nothing exposes a filesystem path at runtime. Consumers are given Flysystem's
own `FilesystemOperator`, not an engine-owned wrapper around it.

A path is kept only where PHP's own `include` is involved: the modules directory, for Composer-based discovery and
autoloading; the compiled directory, for compiled templates and module metadata; and the log directory, which is
where Monolog writes.

## Alternatives

**PHP's file functions with a path**, as the engine uses today. A consumer can come to depend on local disk, and a
filesystem backed by object storage or a database table has no path to give it, so substitutability would rest on
review rather than on the type.

**An engine-owned interface wrapping Flysystem.** It would cost twenty-one delegating methods, kept in step with a
library that already has them, and module authors work with filesystems directly, so the type they already know is
the one to expose.

Whether any other library was weighed against Flysystem is not recorded.

## Consequences

Easier:

- A filesystem can be backed by anything Flysystem supports without any consumer noticing, because no consumer can
  obtain a path. Configuration can be loaded with every filesystem in memory, touching no disk at all.
- Flysystem's own guarantees carry through: its path normaliser rejects path traversal and control characters, and
  its `FilesystemException` is a marker interface, which is the convention the engine's components already follow.
- Module authors will meet a type they already know.

Harder:

- Every filesystem has to be constructed by whatever composes the engine and handed in. There is no composition root
  yet, so tests construct them, as they already construct `Paths`.
- Anything that wants a real path has to justify itself against the `include` rule. Compiled output is separated from
  cached content so that the rule needs no exceptions.

Constrained:

- `league/flysystem` becomes a dependency of the binary, with `league/flysystem-local` and
  `league/mime-type-detection` behind it, and an adapter for each backend.
- Local paths remain in three places only: modules, compiled output and logs.
- A remote adapter holds a connection, so whatever adds the first one inherits the question of how long a filesystem
  lives.

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
