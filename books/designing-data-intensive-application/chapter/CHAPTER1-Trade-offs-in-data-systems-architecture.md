---
title: Trade-offs in Data Systems Architecture
tags:
  - distributed-system
aliases:
  - CHAPTER 1 - Trade-offs in data systems architecture
---
# Trade-offs in Data Systems Architecture

## Overview
Data-intensive applications are shaped by trade-offs.

- Small systems can often run on a single machine
  - simpler to operate
  - easier to reason about
- Larger systems usually need distribution
  - more storage and processing capacity
  - higher availability
  - parallelism and concurrency

## Application Needs
Different system capabilities support different needs.

- databases for storing data
- caches for fast lookup of expensive operations
- search indexes for keyword-based access
- stream processing for handling events as soon as possible
- batch processing for large volumes of data

## Core Trade-Offs
No single tool fits every workload.

- different databases exist for different purposes
- different architectures optimize different access patterns
- the hard part is choosing the right trade-off for the job

## Related Notes
- [[operational-systems-oltp-olap|Operational Systems (OLTP, OLAP)]]
  - Operational systems serve end-user CRUD and point queries, while analytical systems are better suited to ETL, bulk queries, and aggregation.
- [[cloud-versus-self-hosting|Cloud Versus Self-Hosting]]
  - Build-vs-buy depends on control, cost, and scaling needs, with cloud offering elasticity but less direct control.
- [[distributed-versus-single-node-systems|Distributed Versus Single-Node Systems]]
  - Distributed systems improve scale and resilience but add complexity; single-machine systems remain simpler when they are sufficient.
- [[microservices-and-serverless|Microservices and Serverless]]
  - Microservices split applications into independently deployable services, while serverless shifts infrastructure management to the cloud provider.
- [[cloud-computing-versus-supercomputing|Cloud Computing Versus Supercomputing]]
  - Cloud and HPC optimize for different kinds of workloads and communication patterns.
- [[data-systems-law-and-society|Data Systems, Law, and Society]]
  - System design also includes governance, legal, and societal constraints.
