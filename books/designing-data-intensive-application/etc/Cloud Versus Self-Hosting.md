---
tags:
  - cloud-infrastructure
---
**Whether build it or buy it?** it's up to business priorities

By business priorities
- In-house (eg. application code)
	- competency / competitive advantage
	- more control
	- big investment
- Vendor (off the shelf software eg. cloud, SaaS)
	- non-core, commonplace
	- less control
	- smaller investment
- Middle ground (eg. self hosted db on IaaS)
	- vendor software, self hosted/deploy (open source or vendor)
		- your own hardware (on premises)
		- rented datacenter / VM in the cloud (IaaS)

### Cloud Services
**Pros**
- fast moving (than setting up own infrastructure)
- better for unpredictable load (cost-effective)
- easy to scale up, down (changes in demand)
**Cons**
- less customization, no control
- harder to debug, get performance metrics
- vendor lock-in (forced to migrate software version / migrate to alternative services)

**Cloud Native System Architecture**
How system implemented to take advantage of cloud services
- better performance (same hardware)
- faster to build, scale recovery

| Category             | Self-hosted                 | Cloud native                     |
| -------------------- | --------------------------- | -------------------------------- |
| Operational/**OLTP** | MySQL, Postgres, MongoDB    | AWS Aurora, Google Cloud Spanner |
| Analytical/**OLAP**  | Teradata, ClickHouse, Spark | Google BigQuery, Snowflake       |
**Layering of cloud services**
- dynamic resource allocation (managed by provider)
- another idea of resources management
	eg. (==higher-level system are built on top of lower-level infrastructure==)
	- cloud object storage (eg. Amazon S3, Cloudflare R2) can distributes data across machine (no worry about running out of disk space on each machine)
	- Snowflake (compute+query engine) is built on top S3 (db)
- self-hosted have to manage
	- OS (Windows, Linux)
	- data storing (filesystem)
	- networking (TCP/IP)
	- special hardware (GPUs, RDMA, network interfaces)
	- use a lot of resources
	
**Separation of storage and compute**
	**On-premise** systems - storage and compute live together (stored and process data)
	**Cloud-native** systems - splits into services
	- Storage layer (eg. RDBMS, Object storage) - independent, durable, scalable
	- Compute layer (processing, APIs, queries) - can be shut down, data safely stored
	**Pros**
	- detached from one instance
	**Cons**
	- sensitive to network glitch (I/O called by network)

**Operations in the Cloud Era**
teams with both software development and operation responsibilities together
- ensure stable environment
	- monitoring, diagnosing
- from **capacity planning** to **pay-as-you-go** model
	- financial > hardware capacity
- set up automation workflows (reduce repeatable process, manual jobs)
- choose appropriate service for each task
	- include integration between services
