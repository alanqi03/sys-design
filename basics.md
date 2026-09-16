---
title: Basics
---

# Basics

Distributed systems are programs that communicate across a network and run on
machines that can fail independently. Networking therefore comes first: before
designing larger architectures, understand how remote machines communicate and
how those calls can fail.

In this part, you will learn to:

- trace a request from a client to a server;
- distinguish latency from throughput;
- recognize the roles of DNS, TCP, TLS, HTTP, proxies, and load balancers;
- distinguish encryption from signing and know when a system needs both;
- compare vertical and horizontal scaling; and
- reason about timeouts, retries, idempotency, and overload.

The key theme is uncertainty. A remote call can be slow, fail, or complete even
when the caller never receives the response. Good systems make that uncertainty
manageable.
