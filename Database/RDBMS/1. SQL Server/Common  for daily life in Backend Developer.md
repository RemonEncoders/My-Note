## 1. Tables

```sql
SELECT *
FROM sys.tables;
```

Shows all user tables.

---

## 2. Columns

```sql
SELECT *
FROM sys.columns
WHERE object_id = OBJECT_ID('dbo.Employee');
```

Shows all columns of a table.

---

## 3. Data Types

```sql
SELECT *
FROM sys.types;
```

Shows all SQL Server data types.

---

## 4. Primary Keys

```sql
SELECT *
FROM sys.key_constraints
WHERE type = 'PK';
```

---

## 5. Unique Constraints

```sql
SELECT *
FROM sys.key_constraints
WHERE type = 'UQ';
```

Example:

```sql
ALTER TABLE Employee
ADD CONSTRAINT UQ_Employee_Email
UNIQUE(Email);
```

---

## 6. Foreign Keys

```sql
SELECT *
FROM sys.foreign_keys;
```

---

## 7. Indexes

```sql
SELECT *
FROM sys.indexes;
```

Includes:

- Clustered Index
    
- Nonclustered Index
    
- PK-backed indexes
    
- Unique indexes
    

---

## 8. Check Constraints

Example:

```sql
CHECK (Age >= 18)
```

Metadata:

```sql
SELECT *
FROM sys.check_constraints;
```

---

## 9. Default Constraints

Example:

```sql
Age INT DEFAULT 0
```

Metadata:

```sql
SELECT *
FROM sys.default_constraints;
```

---

## 10. Views

```sql
SELECT *
FROM sys.views;
```

---

## 11. Stored Procedures

```sql
SELECT *
FROM sys.procedures;
```

or

```sql
SELECT *
FROM sys.objects
WHERE type = 'P';
```

---

## 12. Functions

```sql
SELECT *
FROM sys.objects
WHERE type IN ('FN','IF','TF');
```

|Type|Meaning|
|---|---|
|FN|Scalar Function|
|IF|Inline Table Function|
|TF|Table Valued Function|

---

## 13. Triggers

```sql
SELECT *
FROM sys.triggers;
```

---

## 14. Schemas

```sql
SELECT *
FROM sys.schemas;
```

Examples:

- dbo
    
- hrm
    
- scm
    
- sales
    

---

## 15. Sequences

```sql
SELECT *
FROM sys.sequences;
```

---

## 16. Synonyms

```sql
SELECT *
FROM sys.synonyms;
```

Often used by DBAs for cross-database abstraction.

---

## 17. Database Users

```sql
SELECT *
FROM sys.database_principals;
```

---

## 18. Roles

```sql
SELECT *
FROM sys.database_role_members;
```

---

## 19. Permissions

```sql
SELECT *
FROM sys.database_permissions;
```

---

## 20. Partitions

Very DBA-focused.

```sql
SELECT *
FROM sys.partitions;
```

Used for huge tables.

---

## 21. Statistics

SQL Server Query Optimizer uses these.

```sql
SELECT *
FROM sys.stats;
```

A DBA checks this often when performance tuning.

---

## 22. Dependencies

"What uses this table?"

```sql
SELECT *
FROM sys.sql_expression_dependencies;
```

Useful before dropping columns.

---

# The catalog views a senior developer should know by heart

If I were mentoring a .NET developer working with EF Core, I would tell them to memorize these first:

```sql
sys.tables
sys.columns
sys.types
sys.indexes
sys.index_columns
sys.foreign_keys
sys.foreign_key_columns
sys.key_constraints
sys.check_constraints
sys.default_constraints
sys.views
sys.procedures
sys.triggers
sys.schemas
sys.stats
```

These cover about **80% of real-world debugging**.

---

# DBA-level catalog views

Later, learn:

```sql
sys.partitions
sys.allocation_units
sys.dm_db_index_usage_stats
sys.dm_exec_query_stats
sys.dm_exec_sql_text
sys.dm_exec_requests
sys.dm_os_wait_stats
```

These are where performance tuning starts.

---

### A mental model

Think of SQL Server metadata like this:

```text
sys.tables
    |
    +-- sys.columns
    |
    +-- sys.indexes
    |
    +-- sys.foreign_keys
    |
    +-- sys.key_constraints
    |
    +-- sys.check_constraints
    |
    +-- sys.default_constraints
```

Every table is the parent object. Almost everything else (PK, FK, Index, Check, Default, Trigger) hangs off that table through `object_id`.

Once you understand how `object_id` links these catalog views together, reading SQL Server metadata becomes much easier, and you'll be able to diagnose migration failures, missing constraints, index issues, and schema drift without relying entirely on SSMS.