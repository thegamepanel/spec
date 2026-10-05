---
title: Statement list
formerly: []
named_by: RFC-0012
---

# Statement list

The ordered statements one schema node compiles into, such as a table followed by its separate indexes and comments, executed in order inside one transaction. The exceptions are creating an index concurrently and creating a database, neither of which can run inside a transaction.
