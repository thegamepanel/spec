---
id: ADR-0003
title: Seal registries into catalogues
status: accepted
created: 2026-03-25
decided: 2026-04-22
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0003: Seal registries into catalogues

## Context

Several parts of the [engine](../GLOSSARY.md#engine) collect things while the panel boots and consult them afterwards.
The container collects [binding](../GLOSSARY.md#binding)s and [resolver](../GLOSSARY.md#resolver)s, and the
configuration component collects [configuration object](../GLOSSARY.md#configuration-object)s. Registration happens in
stages and from more than one place, and some of it depends on what is already registered:
[module](../GLOSSARY.md#module)s register their own configuration, but which modules are enabled is itself
configuration, and has to be readable before any module registers anything.

Once boot is over, the code that reads these collections needs them to stay as they were when it read them.

## Decision

I will have anything registered during boot collected by a mutable [registry](../GLOSSARY.md#registry) and sealed into
an immutable [catalogue](../GLOSSARY.md#catalogue), which is what the rest of the panel reads. The registry is a
short-lived builder, discarded once sealed. The catalogue is read-only, and immutability is enforced at the type
boundary between the two rather than by convention. Where registration depends on something already registered, sealing
happens in phases, each producing something the next phase can read.

## Alternatives

No alternatives were weighed.

## Consequences

Easier:

- Code that reads a [catalogue](../GLOSSARY.md#catalogue) will be able to rely on it not changing underneath it, because
  nothing can register into a catalogue.
- Registration that depends on earlier registration will be a sealing phase rather than an ordering convention:
  configuration seals its [core configuration](../GLOSSARY.md#core-configuration), including the enabled
  [module](../GLOSSARY.md#module)s, before modules register their own.

Harder:

- Every collection that takes registrations will need two types, a [registry](../GLOSSARY.md#registry) and a
  catalogue, and a step that turns one into the other.
- Nothing will be registrable once a registry is sealed. Registration attempted after sealing
  will throw, so a component that needs to register late will have to fit into a sealing phase.

Constrained:

- Components that collect registrations will follow the same shape: the container's [binding](../GLOSSARY.md#binding)s
  and [resolver](../GLOSSARY.md#resolver)s, and configuration with its core and full seals.

## Sources

- Commit [bdc32d7], "Initial dependency injection work", 2026-03-25: `BindingRegistry` and `BindingCatalogue`, the
  first registry and catalogue in the code, which is `created`.
- Commit [798470f], "Add resolution and invocation to the container", 2026-03-25: `ResolverRegistry` and
  `ResolverCatalogue`.
- Planning session, 2026-03-29, not publicly available and first written down on 2026-09-14: the container's binding
  registry in the panel's feature roadmap.
- Planning session, 2026-04-22, not publicly available and first written down on 2026-09-14: the split between
  builder and catalogue, the two-phase seal, and the principle that immutability is enforced at type-system
  boundaries with short-lived, discarded mutable builders. This is `decided`.
- Planning session, 2026-06-15, not publicly available and first written down on 2026-09-14: that the phased seal
  exists specifically so that module configuration can depend on configuration read earlier.
- PR [#25], feat(config): Add config component, merged 2026-03-29: the configuration catalogue, renamed from a
  registry during the pull request.
- PR [#33], refactor(engine:config): Switch to TOML loader and two-phase registry, merged 2026-06-01:
  `ConfigRegistry` sealing into `ConfigCatalogue` through `sealCore()` and `seal()`, with registration after sealing
  throwing. The consequences for configuration are taken from what it built rather than anticipated.

[bdc32d7]: https://github.com/thegamepanel/panel/commit/bdc32d7
[798470f]: https://github.com/thegamepanel/panel/commit/798470f
[#25]: https://github.com/thegamepanel/panel/pull/25
[#33]: https://github.com/thegamepanel/panel/pull/33
