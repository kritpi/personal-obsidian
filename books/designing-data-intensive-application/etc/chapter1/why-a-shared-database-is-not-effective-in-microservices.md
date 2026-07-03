---
title: Why a Shared Database Is Not Effective in Microservices
tags:
  - database
  - microservices
  - distributed-system
---
# Why a Shared Database Is Not Effective in Microservices

### Independent Database
- each service owns its data
- independent
	- schema changes without affecting others
	- service deploy changes, no coordinating db
- scalability - scale per service
- flexibility (eg. PostgreSQL, MongoDB, etc.)
**Trade-offs**
- complexity
- data duplication (in multiple services)
- overhead (get other data through APIs, events)
- data sync (between services, not immediate)

### Shared Database
- multiple services, one database
- easier to join data
**Trade-offs**
- tight coupling (**==biggest problem==**)
	- schema changes effect all services
- coordinate db changes
- limit scalability (harder to scale)

**TL;DR**
	Shared databases create tight coupling between microservices, making schema changes affect all services, requiring coordination for DB changes, and limiting scalability—undermining microservices' core benefits of independence and flexibility.
---

*Part of [[CHAPTER1-Trade-offs-in-data-systems-architecture|Chapter 1 — Trade-offs in Data Systems Architecture]]*
