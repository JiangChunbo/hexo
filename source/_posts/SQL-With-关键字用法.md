---
title: SQL With 关键字用法
date: 2022-10-27 15:13:08
tags:
---


- 递归树形


```sql
WITH RECURSIVE temp_dept ( "depth", "id", "pid", "name", "full_path" ) AS (
    SELECT
        1,
        "id",
        pid,
        "name",
        "name" 
    FROM
        dept 
    WHERE
        pid IS NULL UNION ALL
    SELECT
        parent."depth" + 1,
        child."id",
        child."pid",
        child."name",
        CONCAT ( parent.full_path, '/', child."name") :: varchar(100)
    FROM
        dept as child
        INNER JOIN temp_dept AS parent ON child.pid = parent."id"
    ) 
SELECT * FROM temp_dept
```