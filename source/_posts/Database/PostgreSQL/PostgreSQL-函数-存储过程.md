---
title: PostgreSQL 函数 存储过程
date: 2022-12-06 18:22:53
tags:
---

# 参考引用

https://www.postgresql.org/docs/11/plpgsql.html

https://www.postgresqltutorial.com/postgresql-plpgsql/postgresql-create-function/

# Function

```sql
create [or replace] function function_name(param_list)
   returns return_type 
   language plpgsql
  as
$$
declare 
-- variable declaration
begin
 -- logic
end;
$$
```

pgSQL 的 function 可以声明返回 `void`


# Procedure

Procedure 没有返回值。因此，Procedure 可以在没有 RETURN 语句的情况下结束。如果希望使用 RETURN 语句提前退出代码，只需编写不带表达式的 RETURN 语句即可。

如果 Procedure 有 OUTPUT 参数，OUTPUT 参数变量的最终值将返回给调用者。


## 调用 Procedure

在 function，procedure，或者 `DO` 块中，可以使用 `CALL` 调用过程。