---
title: Defining Nonfunctional Requirements
tags:
  - distributed-system
aliases:
  - CHAPTER 2
---
# Defining Nonfunctional Requirements

> **Functional** - what the system does.
> **Nonfunctional** - how well it performs.

## Overview
Application design is driven by requirements.

- Most requirements are **functional**
  - what the application must offer
  - which actions and behaviors it must support
- Nonfunctional requirements are often less explicit
  - performance
  - reliability
  - maintainability

## Case Study: Social Network Home Timelines
Implementing a Twitter-style social network highlights the gap between functional requirements and nonfunctional constraints.

- users can post messages
- users can follow other users
- scale reaches 500 million posts per day
  - about 5,800 to 150,000 posts per second, depending on activity
- each user follows or is followed by about 200 accounts on average

## Related Notes
- [[representing-users-posts-and-follows|Representing Users, Posts, and Follows]]
  - A relational join can model users, posts, and follows simply, but fresh home timelines become expensive at social-network scale.
- [[materializing-and-updating-timelines|Materializing and Updating Timelines]]
  - Materializing timelines speeds up reads by precomputing home timelines, but shifts cost to writes and requires different fan-out strategies.
- [[describing-performance|Describing Performance]]
  - Response time and throughput are the first performance metrics needed to reason about overload, retries, and backpressure.
- [[circuit-breaker|Circuit Breaker]]
  - The circuit breaker pattern isolates failing dependencies and prevents cascading failures across services.
- [[sql-order-of-execution|SQL Order of Execution]]
  - Understanding clause evaluation order helps diagnose unexpected query results and optimize performance.
