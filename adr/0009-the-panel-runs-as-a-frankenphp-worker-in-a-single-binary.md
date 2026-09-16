---
id: ADR-0009
title: The panel runs as a FrankenPHP worker in a single binary
status: accepted
created: 2026-03-30
decided: 2026-03-30
backfilled: 2026-09-15
depends: [ADR-0001]
updates: []
obsoletes: []
---

# ADR-0009: The panel runs as a FrankenPHP worker in a single binary

## Context

The panel is one system, shipped as a single binary, per
[ADR-0001](0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md).

The panel is installed and run by the people who host it. A PHP application is usually installed as a project: the
right PHP version, its extensions, its Composer dependencies and a web server all have to be in place before it runs,
as they do for Pterodactyl. That is a high barrier to entry.

A PHP application served the usual way also boots again for every request, and discards everything it built once the
response is sent.

## Decision

The panel is distributed as a single binary built with FrankenPHP, with the whole application compiled into it, so
that it is installed as a product rather than set up as a project. It runs in FrankenPHP's worker mode, with Caddy
underneath: the application boots once, and a long-running process serves every request. The engine commits to this
one runtime, and nothing in it abstracts over server APIs it will never run under.

## Alternatives

**Installing the panel as a PHP project**, as Pterodactyl is installed. Whoever installs it has to provide the right
PHP version, the Composer dependencies and the rest, which is a much higher barrier to entry than a single binary.

**Other PHP runtimes.** FrankenPHP is chosen over them because it can compile the entire application into a single
binary, which is what lets the panel be distributed as an installable product. Which runtimes were weighed against
it is not recorded.

**Booting the application for every request.** A long-running worker process is more efficient.

**Abstracting over server APIs**, so that the engine could run under more than one. The panel only ever runs inside
this binary, so the engine commits to one runtime instead.

## Consequences

Easier:

- The panel will install as one file, with no PHP version, extensions or Composer dependencies to provide.
- Caddy will handle TLS and certificate management, HTTP/2, HTTP/3, compression, static files and reverse proxying,
  so the engine will not.
- Work done at boot will be done once for the life of a worker, rather than once for every request.

Harder:

- Anything cached at boot will be shared by every request for the life of the worker, and anything cached during a
  request will leak into the next one unless something ends it.
- Lifetimes will change meaning without any code changing: a shared instance will last until the worker restarts,
  not until the end of the request.
- Every PHP extension the application or its dependencies require will have to be compiled into the binary.

Constrained:

- Code will only ever run inside FrankenPHP's worker, so nothing will be written for any other server API.
- Anything holding request state will need an explicit scope that ends with the request.

## Sources

- Issue [#31], Engine - Config - TOML, original text of 2026-03-30 in its edit history: the first written record of
  the panel being distributed as a binary using FrankenPHP. The decision, worker mode included, was taken at the start
  of the project on a date not recalled, so this first record is both `created` and `decided`.
- The reasons, that FrankenPHP can compile the entire application into a single binary so that the panel can be
  distributed as an installable product, that installing the panel as a PHP project, as Pterodactyl is installed, is
  a high barrier to entry, and that a long-running worker process is more efficient, and worker mode being decided at
  the start of the project: first written down on 2026-09-15, with no record of when they were weighed.
- Issue [#62], Engine - HTTP, 2026-08-21: worker mode with Caddy underneath, committing to one runtime rather than
  abstracting over server APIs, what Caddy handles, and what a long-running worker forces on cached state, taken from
  what followed rather than anticipated.
- Issue [#63], Engine - Container - Binding lifetimes, 2026-08-21: shared instances lasting until the worker restarts
  rather than until the end of the request, taken from what followed rather than anticipated.
- PR [#75], chore: Rename package, add licence and patch Composer CVE, merged 2026-09-13 and squashed as [3372f20]:
  extensions a dependency declares needing to be compiled into the single-binary build, taken from what followed
  rather than anticipated.
- Commit [0d62ceb], "Correct README badges and repository framing", 2026-08-21: the README recording the FrankenPHP
  worker runtime.

[#31]: https://github.com/thegamepanel/panel/issues/31
[#62]: https://github.com/thegamepanel/panel/issues/62
[#63]: https://github.com/thegamepanel/panel/issues/63
[#75]: https://github.com/thegamepanel/panel/pull/75
[3372f20]: https://github.com/thegamepanel/panel/commit/3372f20
[0d62ceb]: https://github.com/thegamepanel/panel/commit/0d62ceb
