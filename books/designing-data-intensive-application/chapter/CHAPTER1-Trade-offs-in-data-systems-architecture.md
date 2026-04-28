---
title: Trade-offs in Data Systems Architecture
tags:
  - distributed-system
aliases:
  - CHAPTER 1 - Trade-offs in data systems architecture
---
# Trade-offs in Data Systems Architecture

**Single Machine**
- Small data
- Easy to deal with

**Distributed**
- Complex
- Multiple storage, processing
- Highly available
- Parallel, Concurrency

**Application Needs**
 - Store data ***(databases)***
 - Fast lookup for expensive operation ***(caches)***
 - Search data by keywords ***(search indexes)***
 - Handle events ASAP ***(stream processing)***
 - Large amount of data ***(batch processing)***

**Challenges**
- Different databases, different purposes. How to choose it?
- Which tools  for which approaches, how about their trade-offs?
	- Caching strategy, Building search indexes 
	- There's no single tool that can do it all

[[Operational Systems(OLTP, OLAP)]]
**TL;DR**
	Operational and Analytics system is used for different purposes
	- Operational (OLTP)
		- end user(CRUD)
		- point query
	- Analytics (OLAP)
		- ETL process
		- bulk query, aggregate, calculation

[[Cloud Versus Self-Hosting]]
**TL;DR**
	Build vs buy depends on business priorities: in-house for competitive advantage/control (big investment), cloud/SaaS for non-core needs (less control, smaller investment), or middle ground like self-hosted on IaaS. Cloud offers fast scaling and cost-effective unpredictable loads but less control and vendor lock-in. Cloud-native architectures separate storage/compute for scalability, shifting operations from capacity planning to automated pay-as-you-go models.

[[Distributed Versus Single-Node Systems]]
TL;DR
	Distributed systems (multiple networked machines) provide benefits like fault tolerance, scalability, and specialized hardware utilization but introduce complexity in failure handling, security, and performance. Single-machine solutions are often simpler and cheaper.

[[Microservices and Serverless]]
TL;DR
	Microservices decompose applications into independent services with well-defined APIs, enabling flexible implementation changes but introducing complexity, testing challenges, and infrastructure requirements (addressed via orchestration like Kubernetes). Serverless (Function-as-a-Service) shifts infrastructure management to cloud providers with automatic resource scaling, though it entails cold-start latency and directly correlated scaling costs.

[[Cloud Computing Versus Supercomputing]]
TL;DR
-  HPC is built for tightly coupled computational jobs, with nodes close together, low-latency/high-bandwidth communication, and checkpointing for large batch workloads.
  - Cloud is built for isolated, elastic services, with VM-based security boundaries, IP/Ethernet networking, and infrastructure spread across regions.
  - The two solve different problems, so cloud is not a drop-in replacement for supercomputing.

[[Data Systems, Law, and Society]]
## TL;DR

- One system rarely fits every workload; the right choice depends on the job.
- OLTP and OLAP serve different access patterns and should usually be separated.
- Distributed systems add scale, but they also add operational and coordination trade-offs.
