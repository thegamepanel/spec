---
title: Disposal list
formerly: []
named_by: RFC-0007
---

# Disposal list

The cycle instances the dependency injection container disposes of when their cycle closes, in the order they were constructed. They are disposed of from the last to the first, so an instance is disposed of before the instances it depended on when it was constructed.
