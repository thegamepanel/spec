---
title: Ghost object
formerly: []
named_by: RFC-0001
---

# Ghost object

An object of the final class that the dependency injection container creates uninitialised. When it is first accessed, its constructor runs on that same object through the container. From then on it is the instance itself, not a stand-in for one. A dependency marked with the `Ghost` attribute receives a ghost object only where the container would construct the instance itself.
