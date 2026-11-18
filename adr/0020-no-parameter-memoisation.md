---
id: ADR-0020
title: Do not memoise resolution data per parameter
status: accepted
created: 2026-09-02
decided: 2026-09-02
backfilled: 2026-11-18
depends: [ADR-0009, ADR-0014]
updates: []
obsoletes: []
---

# ADR-0020: Do not memoise resolution data per parameter

## Context

The container supplies a constructor's or method's parameters by reflecting on each one: its type, whether it is
optional, its default and its attributes. Each parameter becomes a `Dependency`, which holds the parameter's
`ReflectionType` and the `Named`, [qualifier](../GLOSSARY.md#qualifier) and
[resolvable attribute](../GLOSSARY.md#resolvable-attribute) instances found on it. Handling parameter-level attributes
is the hot path of [resolution](../GLOSSARY.md#resolution), at about 3 µs per constructor parameter, 64% of it spent
reading the attributes.

The panel runs as a long-lived worker process, as
[ADR-0009](0009-frankenphp-worker.md) decides, so anything the container
caches is held for the life of the worker.

Attributes are already memoised at class level, as
[ADR-0014](0014-attribute-presence-flags.md) decides, where every attribute that matters is
a marker tested only for presence. Parameter-level attributes are not all like that. `Named` and resolvable attributes
carry values that are read when the [dependency](../GLOSSARY.md#dependency) is resolved, so they are needed as
instances.

## Decision

I will memoise nothing about a parameter beyond the [resolution](../GLOSSARY.md#resolution) that reflected on it. The
container will keep no `Dependency`, no attribute instance and no record of which type of attribute each attribute class
is, and no resolution plan, whether held in memory or compiled to a file at build time. Each parameter will be reflected
on when it is supplied, and attribute memoisation will stay at class level.

## Alternatives

**Memoising `Dependency` per parameter.** The container would keep each parameter's `Dependency` once built. It would
hold one entry for every class, method and parameter a worker ever resolves, for the life of the worker, and each
entry would keep attribute instances, with their values, and a `ReflectionType` alive. I rejected this because it is
an unbounded, permanent memory cost for a bounded, one-off saving in reflection.

**Caching the resolution plan in memory**, a prebuilt set of dependencies for each class and method. It measures about
fifteen times cheaper than preparing the parameters again, cutting roughly 20% from a representative
[resolution](../GLOSSARY.md#resolution). I rejected this because it carries the same unbounded, permanent cost, with
every entry keeping a `ReflectionType` alive, and because every entry would depend on the
[module scope](../GLOSSARY.md#module-scope) the resolution runs under, so a cached plan could not be shared between
scopes.

**Compiling the resolution plan to a file at build time.** I rejected this because only
external modules need compiled files, because of how they are installed. The rest of
the panel runs as one long-lived process, so a compiled plan would add more complexity than it saves.

**Caching which type of attribute each attribute class is**, so that a parameter's attributes are read in one pass and
matched by name. I rejected this because `Named` and resolvable attributes need instances, so each parameter's
attributes would still be read through reflection, while a marker needs no more than a boolean and a
[qualifier](../GLOSSARY.md#qualifier) only its class. Little reflection would be removed, and what remained would be a
small saving in reflection calls with no problem behind it to solve.

The text this option was weighed in measured it differently, at 5.85 µs of a 9.19 µs call, and preferred it. That
measurement is not disputed here. The reasoning that prevailed is that instances of some attributes are needed either
way.

## Consequences

Easier:

- The container's memory will not grow with the number of classes, methods and parameters a worker resolves over its
  life.
- Nothing cached about a parameter will need invalidating.

Harder:

- Parameter-level attribute handling will remain the hot path of [resolution](../GLOSSARY.md#resolution), at about 3 µs
  per constructor parameter, paid every time a constructor or method has its parameters supplied.

Constrained:

- Attribute memoisation will stay at class level, limited to markers tested only for presence.

## Sources

- Planning session, 2026-09-02, following a code review, not publicly available and first written down on
  2026-09-13: the decision, which is both `created` and `decided`, and the reason that memoising per parameter in a
  long-lived worker is an unbounded, permanent memory cost for a bounded, one-off saving in reflection.
- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08: parameter-level attribute
  handling as the hot path, its measurements, and caching the whole resolution plan suggested as a follow-up. These
  are in its original text, kept in its edit history.
- Issue [#70], Engine - Container - Module scopes, 2026-09-03, the first public record of the decision. Its original
  text, replaced three minutes after it was opened and kept in its edit history, weighed caching the resolution plan,
  each entry pinning a `ReflectionType` and depending on the module scope, against caching which type of attribute
  each attribute class is, which it measured at 5.85 µs of a 9.19 µs call and preferred.
- Compiling the resolution plan to a file at build time being ruled out, and why: first written down on 2026-09-13,
  with no record of when it was weighed.
- Caching which type of attribute each attribute class is being rejected, and why, against the measurement in that
  original text: first written down on 2026-09-14.
- [`src/Container/Dependency.php` at 728ad64](https://github.com/thegamepanel/panel/blob/728ad64/src/Container/Dependency.php):
  a `Dependency` holding the parameter's `ReflectionType` and its `Named`, qualifier and resolvable attribute instances.

[#43]: https://github.com/thegamepanel/panel/issues/43
[#70]: https://github.com/thegamepanel/panel/issues/70
