---
author: Carlos Mendible
categories:
- Azure
date: "2010-10-01T11:05:49Z"
description: SQL Azure Requires Clustered Indexes in all tables
tags: ["SQL"]
title: SQL Azure Requires Clustered Indexes in all tables
---
Recently we performed some test against SQL Azure. We found that our system was throwing an exception cause SQL Azure requires a clustered index in each table ([Wanna know why?](http://blogs.msdn.com/b/sqlazure/archive/2010/05/12/10011257.aspx)).

So how can you find out what tables are causing the issue? The answer is simple, run the following query and you'll have the list of tables to fix:

``` sql
SELECT name
FROM sys.objects
WHERE type = 'U'
AND object_id NOT IN
(SELECT object_id FROM sys.indexes WHERE index_id = 1)
``` 

Hope it helps!