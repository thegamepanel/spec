---
id: ADR-0006
title: Read environment variables only during bootstrap
status: accepted
created: 2026-03-28
decided: 2026-03-29
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0006: Read environment variables only during bootstrap

## Context

Some values the panel needs, such as secrets, come from [environment variable](../GLOSSARY.md#environment-variable)s,
supplied either through the process environment or through a local env file. They are needed while the panel is being
configured. Once it has been, nothing should depend on reading them directly.

## Decision

I will hold [environment variable](../GLOSSARY.md#environment-variable)s in `Env`, a static singleton initialised once
during bootstrap from either the `$_ENV` superglobal or an env file, and `Env` will exist only during initial
bootstrapping. Initialising it a second time, or reading from it before it has been initialised, will throw, and its
typed accessors will throw when a value cannot be cast to the type requested. Code outside bootstrap will not read
environment variables.

## Alternatives

No alternatives were weighed.

## Consequences

Easier:

- [Environment variable](../GLOSSARY.md#environment-variable)s will be available anywhere during bootstrap without being
  passed around.
- Reading a variable as a specific type will fail loudly when its value cannot be cast.

Harder:

- Nothing in the type system will stop code outside bootstrap from reading `Env`, so keeping it to bootstrap will rely
  on discipline.
- An env file will be read into `Env` alone, without populating `$_ENV`, so nothing else in the process will see its
  values.

Constrained:

- Components will receive values drawn from the environment through their configuration, not by reading the
  environment.
- Configuration will resolve environment variables while it is loaded.

## Sources

- Issue [#22], Engine - Config, 2026-03-25: environment variables loaded from the `$_ENV` superglobal or a local env
  file.
- Commit [c4c9f51], "Initial config work", 2026-03-28, on the branch of PR [#25]: the first code of `Env`, which is
  `created`.
- Commit [599a0ea], "Introduce an exception around env initialisation", 2026-03-29, on the same branch: throwing when
  `Env` is used before initialisation or initialised twice.
- PR [#25], feat(config): Add config component, merged 2026-03-29 and squashed as [180794a]: `Env` as merged, which
  is `decided`.
- `Env` being a static singleton by design that exists only during initial bootstrapping, and its typed accessors
  throwing when a value cannot be cast: first written down on 2026-09-14.
- Issue [#31], Engine - Config - TOML, revision of 2026-05-14 in its edit history: configuration resolving environment
  variables against `Env` while it is loaded, taken from what followed rather than anticipated.

[#22]: https://github.com/thegamepanel/panel/issues/22
[#25]: https://github.com/thegamepanel/panel/pull/25
[#31]: https://github.com/thegamepanel/panel/issues/31
[c4c9f51]: https://github.com/thegamepanel/panel/commit/c4c9f51
[599a0ea]: https://github.com/thegamepanel/panel/commit/599a0ea
[180794a]: https://github.com/thegamepanel/panel/commit/180794a
