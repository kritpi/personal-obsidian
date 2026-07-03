---
title: Avoiding Overloaded Systems
tags:
  - distributed-system
---

# Avoiding Overloaded Systems

## Exponential Backoff

### Without backoff
- Wait time increases exponentially
- 1000 clients timeout → 1000 immediate retries

### With backoff
- Spread retries across time (not 1000 at the same time)

## Jitter (Randomization)

1000 requests retry after 1 sec → another traffic spike

### With randomization
- Retries after 0.5–2.0 sec
- Retries are distributed

## [[circuit-breaker|Circuit Breaker]]

Prevent cascading failures, giving services time to recover.

- Stop sending requests to failing services
- Return graceful response in case of failure

## Token Bucket Algorithm

Rate limiter — prevent unlimited retries.

- Bucket holds tokens (eg. 100)
- 1 request = 1 token consumed
- Bucket empty → request rejected

## Load Shedding

Reject some requests (eg. accept 85%, reject 15%).

- Monitor if resources are hitting threshold
- Better than having 100% of requests time out

## Backpressure

Tell clients the server is overloaded instead of silently rejecting.

- HTTP 429 — Too many requests
- gRPC rate limiting
- TCP flow control

## Load Balancing
