### SQL Server Partition Information Retrieval

This stored procedure, `pr_DB_Partition_GetInfo`, helps retrieve partitioning details for tables within a specified database. It provides insights into whether a table is partitioned, the associated partition scheme and function, the partition key, the filegroups used, and the count of partitions. Additionally, it supports filtering by a single table or multiple tables using a comma-separated list.

### 1. Stored Procedure Creation

Here’s the T-SQL code to create the stored procedure:

```sql
USE master;
GO

CREATE OR ALTER PROCEDURE pr_DB_Partition_GetInfo
    @DatabaseName NVARCHAR(128),         -- Parameter to specify the database name
    @TableNames NVARCHAR(MAX) = NULL     -- Optional parameter to filter by multiple table names (comma-separated)
AS
BEGIN
    SET NOCOUNT ON;

    -- Check if the database exists
    IF DB_ID(@DatabaseName) IS NULL
    BEGIN
        PRINT 'Error: Database not found.';
        RETURN;
    END;

    -- Declare the dynamic SQL variable
    DECLARE @SQL NVARCHAR(MAX);

    -- Build the dynamic SQL to query the specified database
    SET @SQL = '
    WITH TablePartitionInfo AS
    (
        SELECT
            s.name AS SchemaName,
            t.name AS TableName,
            ps.name AS PartitionSchemeName,
            pf.name AS PartitionFunctionName,
            c.name AS PartitionKey,
            CASE WHEN ps.data_space_id IS NOT NULL THEN ''Partitioned'' ELSE ''Non-Partitioned'' END AS Partitioned_Status,
            COUNT(DISTINCT fg.name) AS FilegroupCount,
            CASE
                WHEN COUNT(DISTINCT fg.name) = 1 THEN MAX(fg.name)
                ELSE ''Partitions are split by multiple filegroups''
            END AS FilegroupName,
            CASE
                WHEN ps.data_space_id IS NOT NULL THEN COUNT(DISTINCT p.partition_number)
                ELSE 1
            END AS PartitionCount
        FROM ' + QUOTENAME(@DatabaseName) + '.sys.tables AS t
        INNER JOIN ' + QUOTENAME(@DatabaseName) + '.sys.schemas AS s ON t.schema_id = s.schema_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.indexes AS i ON i.object_id = t.object_id AND i.type IN (0, 1) -- Include Heaps and Clustered indexes
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.partition_schemes AS ps ON i.data_space_id = ps.data_space_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.partition_functions AS pf ON ps.function_id = pf.function_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.index_columns AS ic ON i.object_id = ic.object_id AND i.index_id = ic.index_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.columns AS c ON c.object_id = ic.object_id AND c.column_id = ic.column_id AND ic.partition_ordinal = 1
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.partitions AS p ON i.object_id = p.object_id AND i.index_id = p.index_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.destination_data_spaces AS dds ON ps.data_space_id = dds.partition_scheme_id
        LEFT JOIN ' + QUOTENAME(@DatabaseName) + '.sys.filegroups AS fg ON dds.data_space_id = fg.data_space_id
        GROUP BY t.object_id, s.name, t.name, ps.name, pf.name, c.name, ps.data_space_id
    )
    SELECT
        SchemaName,
        TableName,
        ISNULL(PartitionSchemeName, ''Non-Partitioned'') AS PartitionSchemeName,
        ISNULL(PartitionFunctionName, ''Non-Partitioned'') AS PartitionFunctionName,
        ISNULL(PartitionKey, ''Non-Partitioned'') AS PartitionKey,
        Partitioned_Status,
        FilegroupCount,
        FilegroupName,
        PartitionCount
    FROM TablePartitionInfo';

    -- Add filtering by multiple table names if @TableNames is provided
    IF @TableNames IS NOT NULL
    BEGIN
        SET @SQL += '
        WHERE TableName IN (
            SELECT value FROM STRING_SPLIT(@TableNames, '','')
        )';
    END

    -- Append ordering clause
    SET @SQL += ' ORDER BY SchemaName, TableName;';

    -- Execute the dynamic SQL with parameters
    EXEC sp_executesql @SQL, N'@TableNames NVARCHAR(MAX)', @TableNames;
END;
GO
```

### 2. Usage Examples

### **Example 1: Retrieve Partition Info for All Tables**

To get partition details for all tables in the database:

```sql
EXEC pr_DB_Partition_GetInfo @DatabaseName = 'YourDatabase';
```

### **Example 2: Retrieve Partition Info for a Specific Table**

To get partition details for a single table:

```sql
EXEC pr_DB_Partition_GetInfo @DatabaseName = 'YourDatabase', @TableNames = 'AuditDetails';
```

### **Example 3: Retrieve Partition Info for Multiple Tables**

To get partition details for multiple tables:

```sql
EXEC pr_DB_Partition_GetInfo @DatabaseName = 'YourDatabase', @TableNames = 'LPNDetails,APIInboundTransactions';

```

### 3. Sample Output

| SchemaName | TableName | PartitionSchemeName | PartitionFunctionName | PartitionKey | Partitioned_Status | FilegroupCount | FilegroupName | PartitionCount |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| dbo | AuditDetails | PS_AuditDetails | PF_AuditDetails | AuditID | Partitioned | 1 | PRIMARY | 4 |
| dbo | LPNDetails | PS_LPNDetails | PF_LPNDetails | LPNDetailId | Partitioned | 2 | Partitions are split by multiple filegroups | 10 |
| dbo | APIInboundTransactions | Non-Partitioned | Non-Partitioned | Non-Partitioned | Non-Partitioned | 1 | PRIMARY | 1 |

### 4. Error Handling

If the specified database does not exist, the procedure will return the following message:

```jsx
Error: Database not found.
```

### 5. Compatibility

- **SQL Server Versions**: This procedure is compatible with SQL Server 2016 and above due to the use of the `STRING_SPLIT()` function.

### 6. Notes

- Make sure the user executing the procedure has `VIEW DEFINITION` permission on the specified database.
- The procedure dynamically handles both partitioned and non-partitioned tables.
