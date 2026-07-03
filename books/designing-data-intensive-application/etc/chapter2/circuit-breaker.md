---
title: Circuit Breaker
tags:
  - distributed-system
  - microservices
---
# Circuit Breaker

## Circuit Breaker pattern in microservices
- Safety switch
- trips(turn off) before further calls, preventing cascade failures

### Characteristics
- Isolate failing dependencies 
	- **fault tolerance** - continue operating without interrupts even some components failed
- Monitoring
	- latency, error rate, timeouts
- Prevents cascades - temporary stop calling unhealthy service
- Fallback support (graceful response, degradation)
- Auto-recovers - reconnect when the service stabilized

### States in circuit breaking
**Close**
- Operates normally, allow requests to flow between services
- Service healthcheck
	- Monitoring / analyzing metrics (response time, error rates, timeouts)

**Open**
- Metrics breach predetermines thresholds (potential issues)
	- switch from closed to open
	- signaling issues of downstream services
- Stop forwarding requests to failing service (isolating)
- Maintain system stability

**Half-Open**
- Transition after open state timeout period
- Let limited number of trial requests pass through
- Monitor response
	- if **succeed**, switch to **close** state
	- if **failed**, switch to **open** state

### Challenges
- Extra layer of complexity
- Parameter tuning (timeout, failure threshold, recovery period)
	- unoptimized params lead to 
		- unnecessary service disruption  
		- too many failed attempt (causing cascade failure)
- Testing in dev environment
	- Need to carefully plan for real-world situation
- Complex services interdependencies (bidirectional relationship)
### When to use
- Relying on 3rd party services that were likely to failed
- Preventing high response time
- Protect services to failed in others
- Prevent attempts to reconnect during recovery phase

---

*Part of [[CHAPTER2-Defining-Nonfunctional-Requirements|Chapter 2 — Defining Nonfunctional Requirements]]*
