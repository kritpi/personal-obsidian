---
tags:
  - distributed-system
---
***distributed system***
	system that involves several machines (==node==) communicating via network

**Usage**
- **Inherent distribution** - communication between devices (2 more) occur via network (unavoidably)
- **Request between cloud services** 
	- Data stored in one service, process in another
	- ==Microservices== (data transferred over network)
- **Fault tolerance/high availability** - if one goes down, another one can take over
	use ==multiple machines to give a redundancy==, application continue working even if one/several (machine, network, datacenter) fail.
- **Scalability** - spread load to handle across multiple machines
- **Latency** - avoid waiting for network packets
	servers in various regions worldwide
- **Elasticity** - deployment can scale up-down to meet the demand
	pay-as-you-go is easier than handle on single machine 
	- provisioned (reserved) to handle maximum load every time
- **Specialized hardware** - different types of **hardware to match workload**
	(eg. object storage need many disk, few CPUs, 
	data analysis might use more CPU and memory with no disk,
	machine learning need more GPUs)
- **Legal compliance** - system must be obey laws and regulation
	- data privacy and security
	- data location
	- data can be distributed across server in several location
- **Sustainability** - flexibility to run service, jobs at the the time where renewable electricity is available reducing carbon emission, cheap power.

**Problems with Distributed Systems**
[[microservices-and-distributed-system-are-having-potential-for-security-breaches]]
Each API call traverses between each services
- **failure possibility handle**r (interrupt, overload, crash, timeout) 
- don't know if another service **received the request** (no response)
	(retry might not be safe)
- **service calling** still ==slower== than **function calling** in the same process
	- transfer large volume of data
	- single-threaded can perform better (in some case) than a cluster
- **troubleshooting** is slow to response
	- heading of observability (collecting execution data)
	- OpenTelemetry, Zipkin, etc.

**TL;DR** 
	perform task on single machine is often simpler and cheaper than distributed system
