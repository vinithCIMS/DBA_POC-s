# Proof of Concept Document: Partitioning the 'RecalcCounts' Table

## Objective

The goal is to modify the [RecalcCounts] table to include a new column, update its primary key to utilize the existing partition scheme, and create a clustered index for efficient data organization and query performance.

## Modifications Overview

1. **Adding the New Column**: A persisted computed column, [RequestedDate], derived from the [RequestedTime] column.
2. **Updating the Primary Key**: Incorporate the new column [RequestedDate] into the primary key and align it with the partition scheme.
3. **Creating a Clustered Index**: Implement a clustered index on the partitioning column [RequestedDate] to optimize storage and retrieval.


## Steps to Implement

### 1. Add the Persisted Computed Column

```
ALTER TABLE [RecalcCounts]
ADD RequestedDate AS CONVERT(date, RequestedTime) PERSISTED NOT NULL;
GO
```

### 2. Modify the Primary Key

Drop the existing primary key (if applicable) and create a new primary key using the partition scheme:

```
ALTER TABLE [dbo].[RecalcCounts]
ADD CONSTRAINT [pkRecalcCounts_RecordId] PRIMARY KEY NONCLUSTERED
(
    [RecordId] ASC,
    [RequestedDate] ASC
)
ON ps_DateMonthly_Secondary(RequestedDate);
GO
```

### 3. Create a Partitioned Clustered Index

```
CREATE CLUSTERED INDEX ix_RecalcCounts_Partitioned
ON [dbo].[RecalcCounts](RequestedDate)
ON ps_DateMonthly_Secondary(RequestedDate);
GO
```

## Validation

Existing Indexes Migrated to Partition Scheme i.e ps_DateMonthly_Secondary(RequestedDate)

```
SELECT
    t.name AS [Table],
    i.name AS [Index],
    i.type_desc AS [Index Type],
    i.is_primary_key AS [Is Primary Key],
    ds.name AS [Data Space]
FROM sys.tables t
INNER JOIN sys.indexes i
    ON t.object_id = i.object_id
    AND i.type > 0 -- Exclude heaps
INNER JOIN sys.data_spaces ds
    ON i.data_space_id = ds.data_space_id
WHERE t.name in ('RecalcCounts')
ORDER BY t.name, i.name;
```
![image](https://github.com/user-attachments/assets/74a87ac6-7c41-40cc-b1ea-cf648bfc7cae)
```
exec pr_Partition_GetInfo 'RecalcCounts'
```
![image](https://github.com/user-attachments/assets/75da3714-58c2-4a91-848c-9a373fa56461)

![image](https://github.com/user-attachments/assets/3969c353-6592-4dbe-8577-d3443d78085b)

---
## Rollback Plan

In case of issues during implementation, the following steps will be taken to revert the changes:

### Drop the added column:

```
ALTER TABLE [dbo].[RecalcCounts] DROP COLUMN RequestedDate;
GO
```

Remove the partitioned indexes and constraints.

##
