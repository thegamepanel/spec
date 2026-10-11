---
title: Module dependency
formerly: []
named_by: ADR-0028
---

# Module dependency

A module's declared need for another module, recorded in its manifest, such as for that module's tables to exist before its own refer to them. The module system checks declared dependencies and orders modules by them.
