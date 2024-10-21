## POC Document: DB Diving into Secondary Datafile and Partitioning Tables

### Objective:

To implement partitioning for certain database tables, move them to a secondary datafile, and recreate the primary key constraints in the secondary filegroup for improved performance and storage management.

![image](https://github.com/user-attachments/assets/51f667b9-6613-4f72-93b6-8d743491d69b)

### Steps Overview:

1. **Secondary Filegroup Creation**
2. **Partition Function Creation**
3. **Partition Scheme Creation**
4. **Moving Tables from Primary to Secondary Filegroup**
5. **Re-creating Primary Key Constraints on Partitioned Tables**

---

### 1. **Secondary Filegroup Creation:**

Create a secondary filegroup and associate a new .ndf file with it.

```sql
ALTER DATABASE [VIA_JLCA_CIMSProd] ADD FILEGROUP [VIA_JLCA_CIMSProd_Secondary];
ALTER DATABASE [VIA_JLCA_CIMSProd] ADD FILE (
    NAME = 'VIA_JLCA_CIMSProd_Secondary',
    FILENAME = 'W:\Temp\!!!POC_DB_Dive\VIA_JLCA_CIMSProd_Secondary.ndf'
) TO FILEGROUP [VIA_JLCA_CIMSProd_Secondary];
```

### 2. **Partition Function Creation:**

Create a partition function for the selected tables. The partitioning is based on an integer ID column, partitioning the data every 1 million records.

```sql
CREATE PARTITION FUNCTION pf_Id1M (int)
AS RANGE RIGHT FOR VALUES
(1000000, 2000000, 3000000, ..., 200000000); -- Extend as needed
```

### 3. **Partition Scheme Creation:**

Create a partition scheme to place all partitions on the secondary filegroup.

```sql
CREATE PARTITION SCHEME ps_Id1MSecondary
AS PARTITION pf_Id1M
ALL TO ([VIA_JLCA_CIMSProd_Secondary]);
```

### 4. **Re-creating Primary Key Constraints on Partitioned Tables:**

For tables that need partitioning, drop and re-create the primary key on the partition scheme.

Example for `InvSnapshot` table:

```sql
-- Drop the existing Primary Key constraint
ALTER TABLE [dbo].[InvSnapshot] DROP CONSTRAINT [pk_InvSnapshot_RecordId];

-- Recreate the Primary Key constraint on the partition scheme
ALTER TABLE [dbo].[InvSnapshot] ADD CONSTRAINT [pk_InvSnapshot_RecordId]
PRIMARY KEY CLUSTERED ( [RecordId] ASC ) ON ps_Id1MSecondary ([RecordId]);
```

Results:
![image](https://github.com/user-attachments/assets/933e4e4e-d427-4176-b498-eac16cc55e85)


### 5. **Moving Tables from Primary to Secondary Filegroup:**

In order to move tables from the primary filegroup to the secondary filegroup, we need to drop and recreate the primary key constraint, specifying the secondary filegroup.

**Example Script for `InterfaceLogDetails` Table:**

```sql
-- Drop the existing Primary Key constraint
ALTER TABLE [dbo].[InterfaceLogDetails] DROP CONSTRAINT [pkInterfaceLogDetails] WITH ( ONLINE = OFF );
GO
-- Recreate the Primary Key constraint on the secondary filegroup
ALTER TABLE [dbo].[InterfaceLogDetails] ADD CONSTRAINT [pkInterfaceLogDetails] PRIMARY KEY CLUSTERED
( [RecordId] ASC ) ON [VIA_JLCA_CIMSProd_Secondary];
GO
```

This process ensures the data and indexes are moved to the secondary filegroup.

Results:

![image](https://github.com/user-attachments/assets/cedda224-8ee6-452b-b4ed-16c6576580f1)


---
