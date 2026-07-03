---
title: Materializing and Updating Timelines
tags:
  - distributed-system
  - database
  - microservices
---
# Materializing and Updating Timelines

From **traditional** method (use query polling to fetch new post from following users)
- precompute query and cache
	- push new posts into followers timeline
- more work
	- every time user makes a post
- 5,800 posts/seconds, 
	- post reaches 200 followers (fan-out factor of 200)
	- 1 million home timeline writes/seconds
	- saving from 400 million/sender post lookups/second
![[Screenshot 2569-05-16 at 11.44.38 PM.png]]

> Fan-out pattern might be harder to keep synchronized 
> or having a large fan-out factor

- ==***materialization***==
	- precomputed data and updating results of query 
	- **speed up reads**
	- **massive write**
- cache the precomputed data = ***==materialized view==***

### Extreme cases example
- If a user is following a very large number of accounts (eg. following 1,000,000 accounts)
	- fan-out ==on read== is **expensive** (1 timeline request = >1,000,000 queries)
	- **solve** using fan-out ==on write== (1 post created, push post to follower = materialized)
	- drop some posts on their timeline (user not likely reading all posts)
- Celebrity account with a very large number of followers (eg. millions of followers)
	- dropping some posts on write is not OK (someone might not see the posts)
	- fan-out ==on write== **problems** (1 post created = millions of writes)
		- update many db shards
		- overload caches
		- large enqueue background jobs
	- **solve** using fan-out ==on read==
	- or handle this case separately

---

*Part of [[CHAPTER2-Defining-Nonfunctional-Requirements|Chapter 2 — Defining Nonfunctional Requirements]]*
