---
title: Cycle
formerly: []
named_by: RFC-0007
---

# Cycle

A unit of work that whatever drives the dependency injection container opens and closes around it, such as handling a request, running a job or a scheduler tick. An instance with a cycle lifetime is kept until its cycle closes.
