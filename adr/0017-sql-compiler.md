---
id: ADR-0017
title: Produce SQL with a compiler
status: accepted
created: 2026-08-19
decided: 2026-08-19
backfilled: 2026-11-18
depends: [RFC-0003]
updates: []
obsoletes: []
---

# ADR-0017: Produce SQL with a compiler

## Context

Every object in the query and schema builders produces its own SQL, as [RFC-0003](../rfc/0003-database-component.md)
describes. Thirty-two classes each write their own, interpolating names straight into the string and returning their
bound values separately.

Two defects come from that shape rather than from any one class. Names are written unquoted in some places and quoted
in others, because each class decides for itself. An insert works out its column list once for its SQL and again for
its bound values, so the two can disagree and bind values to the wrong columns.

Fixing either defect class by class means fixing it thirty-two times, and trusting whoever writes the thirty-third
class to do the same.

## Decision

I will have SQL produced by a compiler rather than by the objects being compiled: a query or
schema object will be a node describing what is wanted, the compiler registered for the node's
class will turn it into SQL, compiling will return the SQL and its bound values together from the same pass, and every
name written into SQL will go through one identifier value object that quotes and escapes it.

## Alternatives

**An identifier value object alone.** Quoting was originally to be fixed by introducing that object and using it
throughout. The object survives as part of this decision. I rejected it as the whole fix because on its own it leaves
thirty-two classes each choosing whether to use it, and does nothing about SQL and bound values being produced
separately.

**Retrofitting quoting across the self-compiling classes.** Each class would quote the names it writes, as it writes
them. I rejected this because the retrofit would have to be dismantled once a compiler arrived anyway.

## Consequences

Easier:

- A name will become SQL in one place, so quoting and escaping will be settled once rather than per class.
- A placeholder and its value will not be able to disagree, because both will come from the same pass over the same
  list.
- Everything a complete statement allows will happen at one point before it becomes a string: logging it, explaining
  it, prefixing its tables, and refusing a statement that should not run.
- A [module](../GLOSSARY.md#module) will be able to add a construct by registering a node and its
  compiler, rather than subclassing the [engine](../GLOSSARY.md#engine)'s query objects.

Harder:

- The whole layer will be restructured, every test asserting SQL will be rewritten, and a
  [connection](../GLOSSARY.md#connection) will need a compiler to execute anything.
- The layer's coverage and mutation score will have to be re-earned rather than preserved.

Constrained:

- Being executable will stop meaning "has a method that returns SQL" and start meaning "a compiler is registered for
  this node", so a half-configured object will not be executable by accident.
- Names will be quoted unconditionally, which will make them case sensitive exactly as written.

## Sources

- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19: the architecture; that thirty-three classes define their own
  SQL production, a count that includes the contract declaring the method alongside the thirty-two implementing it;
  that this is the cause of both the unquoted names and the insert binding defect; the node, compiler, catalogue and
  compiled-result shape; what it buys, in order of value; and its cost. This is `created` and `decided`.
- Issue [#52], Engine - Database - Compiler seam, 2026-08-19, with no edits: the contracts, the compiled result
  carrying SQL and bindings together, the identifier value object with its quoting, qualification, wildcard and
  length rules, the literal quoter, and the hook where a complete statement can be refused.
- Issue [#41], Engine - Query Builder - Identifier quoting and escaping, 2026-08-07, closed 2026-08-19: the quoting
  and escaping that each class decided for itself. Its closing comment records the rejected retrofit across
  thirty-three self-compiling classes, which would have been built and then dismantled when the compiler arrived.
- Issue [#42], Engine - Query Builder - Binding and clause correctness, 2026-08-07, closed 2026-08-19: an insert's
  SQL and bound values computed separately and disagreeing. Its closing comment records the defect becoming
  unrepresentable once both come from one pass.
- The identifier value object having originally been the whole intended fix for quoting: first written down on
  2026-09-16.

[#41]: https://github.com/thegamepanel/panel/issues/41
[#42]: https://github.com/thegamepanel/panel/issues/42
[#51]: https://github.com/thegamepanel/panel/issues/51
[#52]: https://github.com/thegamepanel/panel/issues/52
