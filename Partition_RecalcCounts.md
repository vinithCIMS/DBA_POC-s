# Proof of Concept Document: Partitioning the 'RecalcCounts' Table

## Objective

The goal is to modify the [RecalcCounts] table to include a new column, update its primary key to utilize the existing partition scheme, and create a clustered index for efficient data organization and query performance.

## Modifications Overview

1. **Adding the New Column**: A persisted computed column, [RequestedOn], derived from the [RequestedTime] column.
2. **Updating the Primary Key**: Incorporate the new column [RequestedOn] into the primary key and align it with the partition scheme.
3. **Creating a Clustered Index**: Implement a clustered index on the partitioning column [RequestedOn] to optimize storage and retrieval.


## Steps to Implement

### 1. Add the Persisted Computed Column

```
alter table RecalcCounts add RequestedOn as convert(date, RequestedTime) Persisted not null;
```

### 2. Modify the Primary Key

Drop the existing primary key (if applicable) and create a new primary key using the partition scheme:

```
alter table RecalcCounts
add constraint pkRecalcCounts_RecordId primary key nonclustered (RecordId , RequestedOn)
on ps_DateMonthly_Secondary (RequestedOn);
```

### 3. Create a Partitioned Clustered Index

```
create clustered index ix_RecalcCounts_Partitioned  on RecalcCounts (RequestedOn) on ps_DateMonthly_Secondary (RequestedOn);
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
```
exec pr_Partition_GetInfo 'RecalcCounts'
```
![image](https://github.com/user-attachments/assets/0ff19881-0e3f-4a0b-a1a0-b5da4bdd884f)

![image](https://github.com/user-attachments/assets/c5e3020b-de67-4f92-8932-e36078968c73)


---
## Rollback Plan

In case of issues during implementation, the following steps will be taken to revert the changes:

### Drop the added column:

```
ALTER TABLE [dbo].[RecalcCounts] DROP COLUMN RequestedOn;
GO
```

Remove the partitioned indexes and constraints.

##
