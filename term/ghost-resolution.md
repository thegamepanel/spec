---
title: Ghost resolution
formerly: []
named_by: RFC-0018
---

# Ghost resolution

A resolution that asks for a ghost object. It is made for a dependency marked with `Ghost`, or requested through `ghost()`. Where nothing is cached for it, and the container would construct the instance itself, a ghost object is created before its constructor runs. A ghost resolution of a binding that holds an instance or a factory is resolved as every other resolution is. So is a ghost resolution of a class with no properties that could trigger initialisation.
