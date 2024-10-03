# **SQL Server Login Audit Security POC**

### **Objective**:

Implement and monitor login auditing (both failed and successful logins) in SQL Server, using SQL Audit to create audit logs with rollover for space management.

### **Pre-requirements**:

- **Disk Space**: Ensure sufficient space is allocated based on the audit file configuration.
    - Example: 50 audit files, each 1GB. The total space requirement is 50GB.
    - SQL audit files will automatically rollover, deleting the oldest files when the limit is reached.

### **Steps**:

### **1. Enable Login Auditing (Failed and Successful Logins)**

- Open SQL Server Management Studio (SSMS).
- Navigate to **Server Properties** → **Security** tab.
- Under **Login Auditing**, select **Both failed and successful logins**.

### **2. Create SQL Server Audit**

- Define an audit that will capture login events and store the results in audit files.

```sql

USE [master]
GO
CREATE SERVER AUDIT [LOCAL2019_Login_Audit]
TO FILE
(
    FILEPATH = N'P:\SCT\Temp_VIA\Audit_Files',  -- Define folder path
	MAXSIZE = 1 GB,                             -- Each file's size
	MAX_FILES = 50,                             -- Max number of rollover files
	RESERVE_DISK_SPACE = OFF                    -- Reserve disk space option
)
WITH
(
    QUEUE_DELAY = 1000,                          -- Delay in milliseconds before logging
	ON_FAILURE = CONTINUE                        -- Continue logging on failure
)
GO

```

### **3. Create SQL Server Audit Specification**

- Use the audit created in the previous step to specify what events should be audited (e.g., login attempts).

```sql

CREATE SERVER AUDIT SPECIFICATION [LOCAL_2019_ServerAuditSpecification]
FOR SERVER AUDIT [LOCAL2019_Login_Audit]
ADD (FAILED_LOGIN_GROUP),                       -- Capture failed logins
ADD (SUCCESSFUL_LOGIN_GROUP)                    -- Capture successful logins
GO

```

### **4. Enable SQL Server Audit and Audit Specification**

- Enable the server audit and the audit specification in the correct order (audit specification first).

```sql

-- Enable Server Audit Specification
ALTER SERVER AUDIT SPECIFICATION [LOCAL_2019_ServerAuditSpecification] WITH (STATE = ON)
GO

-- Enable Server Audit
ALTER SERVER AUDIT [LOCAL2019_Login_Audit] WITH (STATE = ON)
GO

```

### **5. Monitor Audit File Generation**

- After enabling the audit, files should start generating in the specified folder (e.g., `P:\SCT\Temp_VIA\Audit_Files`).

### **6. Query Audit Logs**

- Use the following script to read login events from the audit files:

```sql
SELECT
	event_time,
    CONVERT(datetime, SWITCHOFFSET(CONVERT(datetimeoffset, [event_time]), '+05:30')) AS event_time_in_Local_Time,
    server_principal_name AS login_name,
    action_id,
    client_ip,
    host_name,
    application_name,
    statement
FROM sys.fn_get_audit_file(
	'P:\SCT\Temp_VIA\Audit_Files\LOCAL2019_Login_Audit_5BF356FC-F72F-4950-948E-93BEB07E15C0_0_133724503190460000.sqlaudit',
	DEFAULT,
	DEFAULT
)
-- Optional: Filter by specific login
-- WHERE server_principal_name = 'cimsdba'
GO

```

---

### **Additional Notes**:

- Ensure the folder `P:\SCT\Temp_VIA\Audit_Files` has sufficient permissions for SQL Server to write files.
- Regularly monitor the disk space usage to prevent failures due to insufficient space.
- For long-term storage or regulatory compliance, consider exporting audit logs periodically.
