```sql title:"Find All index"
	SELECT 
		* 
	FROM 
		sys.indexes
```

If find a specific index then use this:

```sql title:"search specific index(s/es)"
	SELECT 
		* 
	FROM
		 sys.indexes
	Where 
		name like '%ChartOfAcc%';
```


```sql title:"Index on Which table"
	SELECT 
	    t.name AS TableName,
	    i.name AS IndexName
	FROM 
		sys.indexes i
	INNER JOIN 
		sys.tables t 
			ON i.object_id = t.object_id
	WHERE i.name = 'IX_Companies_ChartOfAccId';
```

```sql title:""
	SELECT 
	    OBJECT_NAME(i.object_id) AS TableName,
	    i.name AS IndexName,
	    c.name AS ColumnName
	FROM 
		sys.indexes i
	INNER JOIN 
		sys.index_columns ic 
			ON i.object_id = ic.object_id 
			AND i.index_id = ic.index_id
	INNER JOIN 
		sys.columns c 
			ON ic.object_id = c.object_id 
			AND ic.column_id = c.column_id
	WHERE 
		i.name = 'IX_Companies_ChartOfAccId';
```



