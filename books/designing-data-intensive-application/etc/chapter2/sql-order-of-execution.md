---
title: SQL Order of Execution
tags:
  - database
---
# SQL Order of Execution

SQL order of execution defines the order of clause
- diagnose why query is not run
- ==performance optimization==

| Order | Clause   | Funciton                                 |
| ----- | -------- | ---------------------------------------- |
| 1     | FROM     | Choose, JOIN table - Get base data.      |
| 2     | WHERE    | Filter base data.                        |
| 3     | GROUP BY | Aggregrates the base data.               |
| 4     | HAVING   | Filter aggregated data.                  |
| 5     | SELECT   | Return final data.                       |
| 6     | ORDER BY | Sort final data.                         |
| 7     | LIMIT    | Limits the returned data to a row count. |

---

*Part of [[CHAPTER2-Defining-Nonfunctional-Requirements|Chapter 2 — Defining Nonfunctional Requirements]]*
