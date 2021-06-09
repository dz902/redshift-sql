# Redshift 常用 SQL

Redshift 是虽然是基于 PostgreSQL 没研发，但有一些修改，而且部分常用的权限查询语句没有简化版，所以这里列一下常用的 SQL 语句。

## 查询用户权限

Redshift 用户相关的权限需要按 Schema、Table、External Table 分别来查询。

### 查询内部 Schema 权限

Schema 权限查询使用 `HAS_SCHEMA_PRIVILEGE()` 函数，我们把用户（`pg_user`）列出来，然后把所有的内部 Schema（`pg_tables`）列出来，做一个十字联结，这样就能查出所有用户在所有表上的权限。

```
SELECT
  u.usename, s.schemaname,
  HAS_SCHEMA_PRIVILEGE(u.usename,s.schemaname,'CREATE') AS user_has_create_permission,
  HAS_SCHEMA_PRIVILEGE(u.usename,s.schemaname,'USAGE') AS user_has_usage_permission
FROM pg_user u
CROSS JOIN
(
  SELECT DISTINCT schemaname FROM pg_tables
) AS s
```

### 查询外部 Schema 权限

```
SELECT
  u.usename, s.schemaname,
  HAS_SCHEMA_PRIVILEGE(u.usename,s.schemaname,'USAGE') AS user_has_usage_permission
FROM pg_user u
CROSS JOIN
(
  SELECT DISTINCT schemaname FROM SVV_EXTERNAL_TABLES
) AS s
```

## 查询组权限

组权限的查询要困难得多。Redshift 的组权限信息以「关系-ACL」的形式存放在 `pg_class` 表里面。这个表不仅包含表的信息，还包含其他类似表的内部数据结构的信息，ACL 也是以文本形式写成，需要进行处理。

### 查询组对表的权限

```
SELECT relacl, 
  'GRANT ' || SUBSTRING(
       CASE WHEN CHARINDEX('r',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',SELECT ' ELSE '' END 
    || CASE WHEN CHARINDEX('w',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',UPDATE ' ELSE '' END 
    || CASE WHEN CHARINDEX('a',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',INSERT ' ELSE '' END 
    || CASE WHEN CHARINDEX('d',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',DELETE ' ELSE '' END 
    || CASE WHEN CHARINDEX('R',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',RULE ' ELSE '' END 
    || CASE WHEN CHARINDEX('x',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',REFERENCES ' ELSE '' END 
    || CASE WHEN CHARINDEX('t',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',TRIGGER ' ELSE '' END 
    || CASE WHEN CHARINDEX('X',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',EXECUTE ' ELSE '' END 
    || CASE WHEN CHARINDEX('U',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',USAGE ' ELSE '' END 
    || CASE WHEN CHARINDEX('C',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',CREATE ' ELSE '' END 
    || CASE WHEN CHARINDEX('T',split_part(split_part(array_to_string(relacl, '|'),pu.groname,2 ) ,'/',1)) > 0 THEN ',TEMPORARY ' ELSE '' END 
  , 2, 10000)
  ||' ON '||namespace||'.'||item||' TO "'||pu.groname||'";' AS GRANTSQL
FROM 
(
  SELECT 
    usr.usename AS subject, 
    nsp.nspname AS namespace, 
    c.relname AS item, 
    c.relkind AS type, 
    usr2.usename AS owner, 
    c.relacl 
  FROM pg_user usr 
  CROSS JOIN pg_class c 
  LEFT JOIN pg_namespace nsp on (c.relnamespace = nsp.oid) 
  LEFT JOIN pg_user usr2 on (c.relowner = usr2.usesysid)
  WHERE c.relowner = usr.usesysid AND nsp.nspname NOT IN ('pg_catalog', 'pg_toast', 'information_schema')
  ORDER BY subject, namespace, item 
)
JOIN pg_group pu ON array_to_string(relacl, '|') LIKE '%'||pu.groname||'%' 
WHERE relacl IS NOT NULL
ORDER BY 2
```

上面一大堆 `SUBSTRING()` 相关的，其实是为了把文本形式的 ACL 解析成各个不同的权限名字。

中间部分：

- 先从 `pg_user` 选择所有的用户
- 再从 `pg_class` 中选择所有 owner 为某个用户的记录
- 再联结 `pg_namespace`，确保 `pg_class` 中的记录不包括几个系统内置表

更细节的更新待续…

### 查询组对外部表的权限

待续















