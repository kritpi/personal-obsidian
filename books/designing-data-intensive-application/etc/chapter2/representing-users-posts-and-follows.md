---
title: Representing Users, Posts, and Follows
tags:
  - database
aliases:
  - CHAPTER 2
---
# Representing Users, Posts, and Follows

Data stored in ==**Relational Database**== (users, post, relationship)
![[Screenshot 2569-05-15 at 12.01.49 AM.png]]

***Home timelines***
- recent posts
- by following accounts
- ignore ads, suggestion post (for now)

```sql
SELECT posts.*, users.* FROM posts
	JOIN follows ON posts.sender_id = follows.followee_id
	JOIN users   ON posts.sender_id = users.id
	WHERE follows.follower_id = current_user
	ORDER BY posts.timestamp DESC
	LIMIT 1000
```
**Execution order**
- find users that `current_user` is following
- look up recent posts
- sort by timestamp
- limit (1,000)
[[sql-order-of-execution|SQL Order of Execution]]

**Additional**
- Post supposed to be timely
- Someone post - others sees within 5 seconds
	- query polling (every 5 seconds) while user is online
	- a lot of query, (many users are online)
- User with high relation (many following/follower)
	- could be up to 400 million lookups/second (10 million users active)

**Query are**
- **Expensive** to execute
- **Difficult** to make **fast**
---

*Part of [[CHAPTER2-Defining-Nonfunctional-Requirements|Chapter 2 — Defining Nonfunctional Requirements]]*
