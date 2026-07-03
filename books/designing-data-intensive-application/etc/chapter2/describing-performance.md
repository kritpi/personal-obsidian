---
title: Describing Performance
tags:
  - distributed-system
---

# Describing Performance

**2 _metrics_ mostly discussed on software _performance_**

- **_response time_** - elapsed time from user request until receive the response (seconds, milliseconds)
- **_throughput_** - number of requests/second, data volume (_maximum throughput_ that system can be handled)
- response time and throughput are often **related**
  - low response time : low throughput
  - high response time : high throughput
  - queueing, CPU process earlier request, delays increase

> **Overloaded System** (Throughput pushed close to the limit)
>
> - Requests waiting to be handled (response time increase / timeout)
>   - More timeout, more retries (request overload)

#### [[to-avoid-overloaded-system|Avoiding Overloaded Systems]]
- Overloaded systems often get worse because retries amplify load. Use exponential backoff, jitter, circuit breakers, token buckets, load shedding, and backpressure to keep failures from cascading.

**Emphasize Response time > Throughput**

| Matrix        | System A  |   System B   |
| :------------ | :-------: | :----------: |
| Response time |   50ms    |      5s      |
| Throughput    | 100 req/s | 10,000 req/s |

- ==Response time== is more noticeable 
- ==Throughput== can be scaled (both vertical, horizontal)
- Response time doesn't scale like throughput
- Could be limited by 
	- Query 
	- Network 
	- Cache miss 
	- etc.

#### Latency and Response Time

---

*Part of [[CHAPTER2-Defining-Nonfunctional-Requirements|Chapter 2 — Defining Nonfunctional Requirements]]*
