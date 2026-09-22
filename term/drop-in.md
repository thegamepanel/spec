---
title: Drop-in
formerly: []
named_by: RFC-0004
---

# Drop-in

A TOML file in `config.d`, merged over the main configuration file in filename order. Where both values are arrays and neither is a non-empty list they merge recursively, and otherwise the drop-in's value replaces the one already there.
