---
title: Liminal
formerly: []
named_by: RFC-0001
---

# Liminal

Describes a shared instance with a process lifetime, held by the dependency injection container through a weak reference, so that garbage collection can clear it once nothing else uses it, after which the next resolution produces a new instance.
