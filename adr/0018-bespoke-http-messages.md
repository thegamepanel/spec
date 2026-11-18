---
id: ADR-0018
title: Use bespoke HTTP messages, not PSR-7
status: accepted
created: 2026-08-21
decided: 2026-08-21
backfilled: 2026-11-18
depends: [ADR-0009]
updates: []
obsoletes: []
---

# ADR-0018: Use bespoke HTTP messages, not PSR-7

## Context

PSR-7 is a shared set of HTTP message interfaces for PHP, and PSR-15 adds handler and middleware interfaces over them.
What they offer is interchangeability: middleware written against them works in any framework that implements them,
and a message can be handed to any library that speaks the same interfaces.

The panel is one purpose-built system running as a single binary on one runtime, as
[ADR-0009](0009-frankenphp-worker.md) decides. Nothing outside it handles its
messages: [module](../GLOSSARY.md#module) authors write actions, and an action never sees a request object.

The interfaces are nearly a decade old. A message built to them takes its shape from that API rather than from what the
panel needs.

## Decision

I will define the [engine](../GLOSSARY.md#engine)'s own request and response objects, and its own contracts for handling
them. The panel will not implement PSR-7 or PSR-15, and nothing in it will depend on their interfaces.

## Alternatives

**Implementing PSR-7 and PSR-15**, so that middleware written for other frameworks could run in the panel. I rejected
this because that interchangeability buys nothing in a bespoke system where no code outside the
[engine](../GLOSSARY.md#engine) holds a message, and it would put a dated API on every internal call site, shaping the
design around the standard rather than around the panel.

## Consequences

Easier:

- The messages will be shaped around what the panel needs, and built as immutable value objects like the rest of the
  [engine](../GLOSSARY.md#engine).
- A request will describe what arrived and nothing more, since no interface will require an attribute bag for carrying
  state through a pipeline.

Harder:

- Middleware written for other frameworks will not run in the panel. Any that is wanted will have to be written
  against the engine's own contracts.

Constrained:

- Every component that handles a request or produces a response will depend on the engine's own
  message types.

## Sources

- Issue [#62], Engine - HTTP, 2026-08-21, edited through 2026-08-22 to narrow its scope and to state content
  directly rather than citing closed issues: the decision, which is `created` and `decided`, and its reasons: PSR-7
  exists so middleware can be shared between frameworks, nothing outside the engine touches these classes because
  module authors write actions, and it would put a 2016-era API on every internal call site for no interoperability.
- Issue [#64], Engine - HTTP - Messages, 2026-08-21: bespoke immutable messages built on property hooks and
  `private(set)`, a request with no attribute bag, and the statement that the reasoning is recorded in [#62].
- The decision following the same reasoning as [ADR-0004](0004-no-psr-11.md), that a
  PSR constrains how a design can be shaped and does not fit a bespoke system: first written down on 2026-09-16.

[#62]: https://github.com/thegamepanel/panel/issues/62
[#64]: https://github.com/thegamepanel/panel/issues/64
