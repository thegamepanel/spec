---
id: ADR-0001
title: Build the panel as one system
status: accepted
created: 2026-03-04
decided: 2026-03-04
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0001: Build the panel as one system

## Context

The panel was first built as two systems. A panel held the frontend, and an [engine](../GLOSSARY.md#engine) behind it
held everything else. Each lived in its own repository, and the panel was installed as an application that depended on
the engine as a library.

The panel ships as a single binary, so whatever repositories it is built from all have to end up inside that one file.

Modules extend the panel. A module that adds something to the frontend has to reach into both
halves when the frontend and the engine are separate systems.

## Decision

I will build the panel as one system: no frontend panel sitting over a backend [engine](../GLOSSARY.md#engine), and no
engine repository of its own. One repository will hold the engine, the modules built on it, the
entry point and the build that produces the binary, and the frontend will be part of that system, rendered on the
server, rather than a client application talking to it.

## Alternatives

**A panel and an engine as separate systems**, each in its own repository, with the panel installed as an application
that depends on the [engine](../GLOSSARY.md#engine) as a library. I rejected this because both repositories would still
have to end up in one binary, and because a module that adds to the frontend becomes much more
complex when the frontend is a system of its own.

## Consequences

Easier:

- A module will extend one system, including whatever it adds to the frontend.
- The build will take one repository and produce the binary from it.

Harder:

- Everything will live in one repository, so the boundary between the [engine](../GLOSSARY.md#engine) and the features
  built on it will be a matter of layering rather than of separate repositories.

Constrained:

- The frontend will be rendered on the server, and the panel will not be a client application talking to an API. That
  removes cross-origin handling, token refresh, API versioning for a first-party consumer, validation written twice,
  and a second toolchain in the binary. It costs a view layer and session handling.
- Nothing will be published for others to install as a library, so the Composer package will be a project rather than a
  library.

## Sources

- The archived repositories `panel_poc`, created 2026-02-17, and `engine_poc`, created 2026-02-25: the two systems,
  both last pushed to on 2026-03-02.
- The `panel` repository, created 2026-03-04: the single system that replaced them, and the first record of the
  decision, which is both `created` and `decided`. The decision was taken before that, on a date not recalled.
- The reasons, that both repositories would have to be part of one binary and that a module adding to the frontend
  becomes much more complex across two systems: first written down on 2026-09-16.
- Commit [0d62ceb], "Correct README badges and repository framing", 2026-08-21: the README dropping the framing of the
  engine as a separately installed library, and recording the server-rendered frontend. That everything lives in one
  repository, and what the frontend is, are taken from what followed rather than anticipated.
- Issue [#62], Engine - HTTP, 2026-08-21: the SPA recorded as scrapped, and what a server-rendered frontend removes and
  costs, taken from what followed rather than anticipated.
- PR [#75], chore: Rename package, add licence and patch Composer CVE, merged 2026-09-13 and squashed as [3372f20]: the
  Composer package renamed from `thegamepanel/engine` to `thegamepanel/panel` and retyped from a library to a project,
  which the commit describes as left over from the abandoned split, and the Packagist badge removed because neither name
  is published and the panel ships as a binary.

[#62]: https://github.com/thegamepanel/panel/issues/62
[#75]: https://github.com/thegamepanel/panel/pull/75
[0d62ceb]: https://github.com/thegamepanel/panel/commit/0d62ceb
[3372f20]: https://github.com/thegamepanel/panel/commit/3372f20
