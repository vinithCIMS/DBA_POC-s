# SQL Server Partitioning POC for `LPNDetails` Table Non-Clustered Index

This POC demonstrates how to set up partitioning in SQL Server for the `LPNDetails` table, validate the initial setup, and dynamically extend the partition function to accommodate additional data ranges.

## Prerequisites

- SQL Server 2016 or later (partitioning supported).
- A database with sufficient space in the `[PRIMARY]` filegroup.

---

## Setup Steps

### 1. **Create Partition Function and Create Partition Scheme**

Define a partition function with initial boundary values for 0, 1,000,000, and 2,000,000 rows.

from D:\SVN_VIA\CIMS 3.0\branches\Dev3.0\SQL\Functions\pfn_Partitions\pfn_Int.sql

```sql
/*----------------------------------------------------------------------------*/
/* Create Default Partition Functions based upon Integer (RecordId, LPNId etc) fields. The below definitions
   are only for prelim definition only i.e. the boundaries are not accurate and the functions
   would later be modified with accurate boundaries */
/*----------------------------------------------------------------------------*/

/* Setup with initial boundary. Later, these would be extended to have boundary values at every million */
create partition function [pf_Int1M](int) as range right for values (0, 1000000, 2000000);

Go

/* This partition scheme is to partition for each 1M records, but all partitions in Primary. This
   scheme does not need to be changed in future, only the function needs to be changed */
create partition scheme [ps_Int1M_Primary] as partition [pf_Int1M] ALL TO ([PRIMARY]);

Go
```
![image](https://github.com/user-attachments/assets/4d7abf63-36ae-45d9-b2ff-c538bb9b11e1)

### 2. **Create a Non-Clustered Index for Partitioning**

Create a non-clustered index on the `LPNDetails` table using the partition scheme.

```sql
create nonclustered index ix_LPNDetails_LPNId2 on LPNDetails (LPNId) include (LPNDetailId, InnerPacks, OnhandStatus, Quantity, UnitsPerPackage, ReservedQty, ReceiptId, ReceiptDetailId, [Weight], ReplenishOrderId, SKUId, OrderId, OrderDetailId, Warehouse, [Ownership], Lot, CoO, InventoryClass1, InventoryClass2, InventoryClass3, InventoryKey)
On [ps_Int1M_Primary](LPNId)
```
![image](https://github.com/user-attachments/assets/67c10da0-f208-4c6e-ac4e-62743eea4c14)

### 3. **Validate Partition Function and Scheme and Check Table Partition Information**

Run the following query to verify the partition function and scheme:

Use a stored procedure (`pr_Partition_GetInfo_all`) to validate partitioning for the `LPNDetails` table:

```sql
SELECT
    pf.name AS Partition_Function,
    ps.name AS Partition_SCHEME,
    pf.fanout AS Partition_Count,
    pf.create_date,
    pf.modify_date
FROM sys.partition_functions pf
JOIN sys.partition_schemes ps ON pf.function_id = ps.function_id
WHERE pf.name = 'pf_Int1M';

EXEC pr_Partition_GetInfo_all @TableName = 'LPNDetails';
```
![image](https://github.com/user-attachments/assets/48fd76f0-6811-463f-820c-eadbb65c40fe)

## Extending Partition Function

### Scenario

When the data volume exceeds the existing boundary (2,000,000 rows), the partition function needs to be extended dynamically to accommodate additional ranges.

### 1. **Extend Partition Function**

Run the stored procedure `pr_Partition_ExtendedPFS_Int` to add partitions dynamically:

```sql
EXEC pr_Partition_ExtendedPFS_Int @pf = 'pf_Int1M', @FG = 'PRIMARY', @PartitionRange = 1000000, @ExtentionCount = 198;

```
![image](https://github.com/user-attachments/assets/380beda6-6679-41fd-ad10-ee51fc9141f3)

### Outcome

This will create 198 new partitions, extending the partition boundary up to 200,000,000.

---

## Verify Extended Partitions

1. Re-run the query to check updated partition information: Validate updated partition details for the `LPNDetails` table:
    
    ```sql
    SELECT
        pf.name AS Partition_Function,
        ps.name AS Partition_SCHEME,
        pf.fanout AS Partition_Count,
        pf.create_date,
        pf.modify_date
    FROM sys.partition_functions pf
    JOIN sys.partition_schemes ps ON pf.function_id = ps.function_id
    WHERE pf.name = 'pf_Int1M';
    
    EXEC pr_Partition_GetInfo_all @TableName = 'LPNDetails';
    
    ```
![image](https://github.com/user-attachments/assets/538367a2-eb24-447d-ad62-6ac0863dc93d)

---

## Key Notes

- The partition scheme remains unchanged as it is mapped to the `[PRIMARY]` filegroup.
- Only the partition function is extended to handle additional data ranges.
- Ensure `pr_Partition_ExtendedPFS_Int` is implemented with necessary logic to handle dynamic partition extensions.

---

## Results

- **Initial Setup**: Partition function and scheme created with boundaries up to 2,000,000.
- **Post Extension**: Partition function extended to cover 200,000,000 rows.

This dynamic partitioning approach enables scalability and efficient data management in SQL Server.
