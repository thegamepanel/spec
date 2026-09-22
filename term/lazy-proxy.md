---
title: Lazy proxy
formerly: []
named_by: 
---

# Lazy proxy

A stand-in that the dependency injection container returns in place of an instance not yet resolved. When the proxy is first used, the container resolves the real instance, and the proxy forwards to it from then on, remaining a proxy.
