---
title: Expression
formerly: []
named_by: RFC-0003
---

# Expression

An object that produces SQL, with a `?` placeholder for each bound value, together with the values bound to those placeholders. Every object in the query builder and the schema builder is one, and an expression containing other expressions builds its SQL from theirs and gathers their bound values in the same order.
