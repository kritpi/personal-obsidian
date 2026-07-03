---
title: Operational Systems (OLTP, OLAP)
tags:
  - database
aliases:
  - OLTP, OLAP
---
# Operational Systems (OLTP, OLAP)

***Operational Systems***
- Served external users
- Read, stored, process based on users action
- **OLTP** *(online transaction processing)* database records
	inserted, updated, deleted based on user's interactive
- Usually looks up the record by a key (point query)
***Analytical Systems***
- Business analysts, Data scientists
- Read, copy data from ==Operational System==
- **OLAP** *(online analytical processing)* 
- Query large records, calculates aggregate statistics (count, sum, average)

| Property               | OLTP                                  | OLAP                                             |
| ---------------------- | ------------------------------------- | ------------------------------------------------ |
| Reading                | Point Query                           | Aggregate (large records)                        |
| Write Pattern          | CRUD individual<br>record             | Bulk import (ETL),<br>Event stream               |
| Human user<br>example  | End user<br>web/mobile app            | Internal analyst                                 |
| Machine use<br>example | Check if action is<br>authorized      | Detecting fraud/abuse<br>patterns                |
| Type of queries        | Fixed, predefined<br>(by application) | Query created dynamically,<br>No fixed structure |
| Query volume           | Lot of small queries                  | Few, complex queries                             |
| Data represents        | Latest state of data                  | History of events over time                      |
| Size                   | Large (Gb to Tb)                      | Larger (Tb to Pb)                                |
**OLTP** mostly run fixed queries embedded in application code
**OLAP** can be freely write an arbitrary queries by hand, or generate queries automatically(Tableau, Looker, Power BI)

### Data Warehousing
They stop using OLTP systems for analytics purpose, and run analytics on separate database system called ***data warehouse***
- OLTP data are spread across multiple systems (hard to combine dataset in single query *(data silos)*)
- Schema and layouts not suited for analytics
- Analytical queries are expensive, impact OLTP performance
- OLTP should not be able to *=='directly access'==*
**Data Warehouse** separate analytical database from OLTP system using
***extract-transform-load (ETL)***
- Extracted data from OLTP (periodic dump, continuous stream)
- Cleaned up, reformat (analysis-friendly schema)
	- Loaded into data warehouse
		![[etl-diagram.png]]

**HTAP** ***(hybrid transactional/analytical processing)*** 
- Single system
- Remove ETL system between OLTP and analytics system
- Does not replace ==**Data Warehouse**==
- Good use when
	- Perform large analytical queries
	- Read/Write individual records

**Data Warehouse** to **Data Lake**
- Usage
	- ==Data Warehouse== use ***relational*** data model (SQL)
	- ==Data Lake== use specialized (***BI***) software
- Transforming data into analytical forms
	- training ML model (transform database table into *features*(vector, matrix) )
	- use NLP on text data (natural languages) to extract structured info
- Using Data Lake is more flexible

**Systems of Record and Derived Data**
*==Systems of record==*
- Source of truth
- Data stored **once and only once**
*==Derived data systems==* - essential for read performance (data combined from multiple sources) 
- Taking data from another system
- Transforming / processing
- Can be replicate
- eg. cache, dataset

---

*Part of [[CHAPTER1-Trade-offs-in-data-systems-architecture|Chapter 1 — Trade-offs in Data Systems Architecture]]*
