---
id: ADR-0017
title: SQL is produced by a compiler, not by the query objects
status: accepted
created: 2026-08-19
decided: 2026-08-19
backfilled: 2026-09-16
depends: [RFC-0003]
updates: []
obsoletes: []
---

# ADR-0017: SQL is produced by a compiler, not by the query objects

## Context

Every object in the query and schema builders produces its own SQL, per
[RFC-0003](../rfc/0003-database-component.md). Thirty-three classes each write their own, interpolating names
straight into the string and returning their bound values separately.

Two consequences follow from that shape rather than from any one class. Names are written unquoted in some places and
quoted in others, because each class decides for itself. And an insert computes its column list once for its SQL and
again for its bound values, so the two can disagree and bind values to the wrong columns.

Fixing either one class by class means fixing it thirty-three times and trusting whoever writes the thirty-fourth.

## Decision

SQL is produced by a compiler rather than by the objects being compiled. A query or schema object is a node
describing what is wanted, and a compiler resolved by node class turns it into SQL. Compilation returns the SQL and
its bound values together, produced by the same pass, and every name written into SQL goes through one identifier
value object that quotes and escapes it.

## Alternatives

**An identifier value object alone.** Quoting was originally to be fixed by introducing that object and using it
throughout. The object survives as part of this decision, but on its own it leaves thirty-three classes each
choosing whether to use it, and does nothing about SQL and bound values being produced separately.

**Retrofitting quoting across the self-compiling classes**, then dismantling it once a compiler arrived anyway.

## Consequences

Easier:

- A name becomes SQL in one place, so quoting and escaping are settled once rather than per class.
- A placeholder and its value cannot disagree, because both are produced by the same pass over the same list.
- Everything a complete statement allows arrives at one point before it becomes a string: logging it, explaining it,
  prefixing tables, and refusing a statement that should not run.
- A module can add a construct by registering a node and its compiler, rather than subclassing the engine's query
  objects.

Harder:

- The whole layer is restructured, every test asserting SQL is rewritten, and the connection needs a compiler to
  execute anything.
- The layer's coverage and mutation score have to be re-earned rather than preserved.

Constrained:

- Being executable stops meaning "has a method that returns SQL" and starts meaning "a compiler is registered for
  this node", so a half-configured object cannot be executed by accident.
- Identifiers are quoted unconditionally, which makes names case sensitive exactly as written.

## Sources

- Issue [#51], Engine - Database - PostgreSQL, 2026-08-19: the architecture, that thirty-three classes each define
  their own SQL production, that this is the cause of both the unquoted names and the insert binding defect, the
  node, compiler, catalogue and compiled-result shape, what it buys in order of value, and its cost. This is
  `created` and `decided`.
- Issue [#52], Engine - Database - Compiler seam, 2026-08-19, edited on 2026-09-13 only to replace names with issue
  links: the contracts, the compiled result carrying SQL and bindings together, the identifier value object with its
  quoting, qualification, wildcard and length rules, the literal quoter, and the hook where a complete statement can
  be refused.
- Issue [#41], Engine - Query Builder - Identifier quoting and escaping, 2026-08-07, closed 2026-08-19: the quoting
  and escaping that each class decided for itself. Its closing comment records the rejected retrofit across
  thirty-three self-compiling classes.
- Issue [#42], Engine - Query Builder - Binding and clause correctness, 2026-08-07, closed 2026-08-19: an insert's
  SQL and bound values computed separately and disagreeing. Its closing comment records the defect becoming
  unrepresentable once both come from one pass.
- The identifier value object having originally been the whole intended fix for quoting: first written down on
  2026-09-16.

[#41]: https://github.com/thegamepanel/panel/issues/41
[#42]: https://github.com/thegamepanel/panel/issues/42
[#51]: https://github.com/thegamepanel/panel/issues/51
[#52]: https://github.com/thegamepanel/panel/issues/52
