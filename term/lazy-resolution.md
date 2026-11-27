---
title: Lazy resolution
formerly: []
named_by: RFC-0006
---

# Lazy resolution

A resolution that asks for a lazy proxy, or whose binding or class is lazy. Where the resolution reaches the lazy proxy step, it is given a lazy proxy in place of the instance. For a class with no properties that could trigger initialisation, no lazy proxy is created and the instance is resolved eagerly instead.
