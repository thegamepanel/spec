---
title: Ghost object
formerly: []
named_by: 
---

# Ghost object

An object of the final class that the dependency injection container creates uninitialised. When it is first accessed, its constructor runs on that same object through the container, and from then on it is the instance itself rather than a stand-in for one. A dependency is resolved as a ghost object with the `Ghost` attribute.
