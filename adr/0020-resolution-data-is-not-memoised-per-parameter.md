---
id: ADR-0020
title: Resolution data is not memoised per parameter
status: accepted
created: 2026-09-02
decided: 2026-09-02
backfilled: 2026-09-14
depends: [ADR-0009, ADR-0014]
updates: []
obsoletes: []
---

# ADR-0020: Resolution data is not memoised per parameter

## Context

The container supplies a constructor's or method's parameters by reflecting on each one: its type, whether it is
optional, its default, and its attributes. Each parameter becomes a `Dependency`, which holds the parameter's
`ReflectionType` and the `Named`, qualifier and resolvable attribute instances found on it. Handling parameter-level
attributes is the hot path of resolution, at about 3 µs per constructor parameter, 64% of it spent reading the
attributes.

The panel runs as a long-lived worker process, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), so anything the container caches is
held for the life of the worker.

Attribute memoisation already exists at class level, per
[ADR-0014](0014-class-level-attributes-are-memoised-as-presence-flags.md), where every attribute that matters is a
marker tested only for presence. Parameter-level attributes are not all like that. `Named` and resolvable attributes
carry values that are read when the dependency is resolved, so they are needed as instances.

## Decision

Resolution data is not memoised per parameter. The container keeps nothing about a parameter beyond the resolution
that reflected on it: no `Dependency`, no attribute instance, and no record of which kind of attribute each attribute
class is. It keeps no resolution plan either, whether held in memory or compiled to a file at build time. Each
parameter is reflected on when it is supplied, and attribute memoisation stays at class level.

## Alternatives

**Memoising `Dependency` per parameter.** It holds one entry for every class, method and parameter a worker ever
resolves, for the life of the worker: an unbounded, permanent memory cost for a bounded, one-off saving in reflection.
Each entry keeps attribute instances, with their values, and a `ReflectionType` alive.

**Caching the resolution plan in memory**, a prebuilt set of dependencies for each class and method. It measures about
fifteen times cheaper than preparing the parameters again, cutting roughly 20% from a representative resolution. It
carries the same unbounded, permanent cost, with every entry keeping a `ReflectionType` alive, and every entry would
depend on the module scope the resolution runs under, so a cached plan could not be shared between scopes.

**Compiling the resolution plan to a file at build time.** Only external modules need compiled files, because of how
they are installed. The rest of the panel runs as one long-lived process, so a compiled plan adds more complexity than
it saves.

**Caching which kind of attribute each attribute class is**, so that a parameter's attributes are read in one pass and
matched by name. It removes no reflection. `Named` and resolvable attributes need instances, so each parameter's
attributes are still read through reflection, a marker needs no more than a boolean, and a qualifier only its class.
What remains is a small saving in reflection calls, with no problem behind it to solve.

## Consequences

Easier:

- The container's memory will not grow with the number of classes, methods and parameters a worker resolves over its
  life.
- Nothing cached about a parameter will need invalidating, so context known only when a resolution runs, such as the
  module scope it runs under, can be stamped onto a `Dependency` at no cost.

Harder:

- Parameter-level attribute handling will remain the hot path of resolution, at about 3 µs per constructor parameter,
  paid every time a constructor or method has its parameters supplied.

Constrained:

- Attribute memoisation will stay at class level, limited to markers tested only for presence.

## Sources

- Planning session, 2026-09-02, following a code review, not publicly available and first written down on
  2026-09-13: the decision, which is both `created` and `decided`, and the reason that memoising per parameter in a
  long-lived worker is an unbounded, permanent memory cost for a bounded, one-off saving in reflection.
- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08: parameter-level attribute
  handling as the hot path, its measurements, and caching the whole resolution plan suggested as a follow-up.
- Issue [#70], Engine - Container - Module scopes, 2026-09-03, the first public record of the decision. Its original
  text, replaced three minutes after it was opened and kept in its edit history, weighed caching the resolution plan,
  each entry pinning a `ReflectionType` and depending on the module scope, against caching which kind of attribute
  each attribute class is. Its revised text records that `Dependency` is not memoised, so stamping a scope on it costs
  nothing, taken from what followed rather than anticipated.
- Compiling the resolution plan to a file at build time being ruled out, and why: first written down on 2026-09-13,
  with no record of when it was weighed.
- Caching which kind of attribute each attribute class is being rejected, and why: first written down on 2026-09-14.
- [`src/Container/Dependency.php` at 728ad64](https://github.com/thegamepanel/panel/blob/728ad64/src/Container/Dependency.php):
  a `Dependency` holding the parameter's `ReflectionType` and its `Named`, qualifier and resolvable attribute
  instances.

[#43]: https://github.com/thegamepanel/panel/issues/43
[#70]: https://github.com/thegamepanel/panel/issues/70
