---
tags:
  - distributed-system
  - microservices
---
**common way of distributing system**
- multiple machines
- client requests to servers
	- client - make outbound requests
	- server - handle income requests
- HTTP, RPC

**Service-oriented architecture (==SOA==)**
- refined into *==**microservices** architecture==*
- service
	- one well-defined purpose
	- exposes API endpoint 
	- called by clients via network
- application can be decomposed into many services
	**Pros**
	- independent
	- like a **black box** with API endpoint
	- free to change implementation without affecting clients
	**Cons**
	- no shared databases between services (tightly coupled)
	- complexity, harder to test (services depends on others)
	- each service requires infrastructure (can be resolve with *orchestration* frameworks eg. **Kubernetes**)
		- hardware resources (to match the load)
		- logging, monitoring, health check, alerting
		
Read more: [[Why a shared database is not effective in microservices]]

**Serverless** *(function as a service)* is another spproach to deploying services
- infrastructure management by cloud vendor
- choose when to start up, shut down an instance (VMs)
- auto allocate/free hardware, resources as needed
**Trade-offs**
- might have a slow cold start times (first invoked)
- autoscale services = autoscale bills