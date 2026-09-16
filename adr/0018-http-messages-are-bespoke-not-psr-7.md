---
id: ADR-0018
title: HTTP messages are bespoke, not PSR-7
status: accepted
created: 2026-08-21
decided: 2026-08-21
backfilled: 2026-09-16
depends: [ADR-0009]
updates: []
obsoletes: []
---

# ADR-0018: HTTP messages are bespoke, not PSR-7

## Context

PSR-7 defines common HTTP message interfaces, and PSR-15 defines handler and middleware interfaces over them. Their
value is interchangeability: middleware written against them can be shared between frameworks, and a message can be
passed to any library that speaks the same interfaces.

The panel is a bespoke, purpose-built system running as a single binary on one runtime, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). Nothing outside it touches its
messages: module authors write actions, which never see a request object.

The interfaces are also nearly a decade old, and a message built to them is shaped by that rather than by what the
panel needs.

## Decision

The engine defines its own request and response objects, and the contracts for handling them. It does not implement
PSR-7 or PSR-15, and nothing in the panel depends on their interfaces.

## Alternatives

**Implementing PSR-7 and PSR-15**, so that middleware written for other frameworks could be used. The
interchangeability does not apply to a bespoke system where nothing outside the engine holds a message, and adopting
the interfaces would put a dated API on every internal call site, shaping the design around the standard rather than
around the panel.

No other alternative was weighed.

## Consequences

Easier:

- The messages will be shaped around what the panel needs, and built the way the rest of the engine is built, with
  immutable value objects.
- A request will describe what arrived and nothing more, since there is no interface requiring an attribute bag to
  carry state through a pipeline.

Harder:

- Middleware written for other frameworks cannot be used, and any that is wanted has to be written against the
  engine's own contracts.

Constrained:

- Every component that handles a request or produces a response depends on the engine's own message types.

## Sources

- Issue [#62], Engine - HTTP, 2026-08-21, edited through 2026-08-22 to narrow its scope and to state content
  directly rather than citing closed issues: the decision, which is `created` and `decided`, and the reasons, that
  PSR-7 exists so middleware can be shared between frameworks, that nothing outside the engine touches these classes
  because module authors write actions, and that it would put a 2016-era API on every internal call site for no
  interoperability.
- Issue [#64], Engine - HTTP - Messages, 2026-08-21: bespoke immutable messages built on property hooks and
  `private(set)`, a request with no attribute bag, and the reasoning for it recorded in [#62].
- The decision following the same reasoning as [ADR-0004](0004-the-container-does-not-implement-psr-11.md), that a
  PSR curbs how a design can be shaped and does not fit a bespoke system: first written down on 2026-09-16.

[#62]: https://github.com/thegamepanel/panel/issues/62
[#64]: https://github.com/thegamepanel/panel/issues/64
