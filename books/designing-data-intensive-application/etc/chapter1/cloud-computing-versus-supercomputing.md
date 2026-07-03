---
title: Cloud Computing Versus Supercomputing
tags:
  - distributed-system
---
# Cloud Computing Versus Supercomputing

***High-performance computing (HPC)*** is an alternative way for large-scale computing systems and use different techniques from the cloud computing

- **used for computationally** 
	- eg. forecasting, solving problems, equations
	- cloud computing used for online services, business data, etc.
- **large batch** of jobs with **checkpoint state** in disk
	- if node fails, stop, repair, then restart from the last checkpoint
	- not ideal for cloud computing
- **HPC** communicate through **shared memory** 
	- support high bandwidth, low latency
	- cloud is isolated (VMs)
		- need more security, encryption, authentication
- **specialized** network topologies
	- multidimensional meshes and toruses
	- better performance
	- cloud network based on IP and Ethernet
- HPC nodes are close together
	- cloud nodes can be distributed 
		- across multiple geographic regions

---

*Part of [[CHAPTER1-Trade-offs-in-data-systems-architecture|Chapter 1 — Trade-offs in Data Systems Architecture]]*
