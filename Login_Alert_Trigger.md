# Proof of Concept (POC) Document for SQL Server Login Auditing

## 1. Objective

The objective of this POC is to establish a system to log all login attempts for the SQL Server login `cimsdba`, providing a mechanism for monitoring and alerting on unauthorized access attempts. The system will log successful logins into a dedicated audit table and send email alerts to designated recipients when such logins occur.

## 2. Components

### 2.1. Database Structure

1. **LoginAudit Table**: Stores details of each login attempt.
2. **Settings Table**: Manages the status of alerts (enabled/disabled).
3. **Stored Procedure**: Allows toggling of alert status.
4. **Trigger**: Captures login events and logs them into the `LoginAudit` table and sends alerts if configured.

### 2.2. SQL Scripts

### 2.2.1. Create LoginAudit Table

```sql
USE AdminDB;
GO

CREATE TABLE dbo.LoginAudit (
    EventID INT IDENTITY(1,1) NOT NULL,
    EventType NVARCHAR(50) NULL,
    LoginName NVARCHAR(256) NULL,
    Hostname NVARCHAR(256) NULL,
    IPAddress NVARCHAR(50) NULL,
    Application NVARCHAR(256) NULL,
    SessionID INT NULL,
    EventTime DATETIME NULL,
    PRIMARY KEY CLUSTERED (EventID ASC)
);
GO
```

### 2.2.2. Create Settings Table

```sql
USE AdminDB;
GO

CREATE TABLE dbo.Settings (
    AlertEnabled BIT NOT NULL DEFAULT 1
);
GO

-- Initialize the settings table if it doesn't already exist
IF NOT EXISTS (SELECT 1 FROM dbo.Settings)
BEGIN
    INSERT INTO dbo.Settings (AlertEnabled) VALUES (1);
END
GO
```

### 2.2.3. Create Stored Procedure for Alert Status

```sql
USE AdminDB;
GO

CREATE PROCEDURE dbo.SetAlertStatus
    @EnableAlert BIT
AS
BEGIN
    UPDATE dbo.Settings
    SET AlertEnabled = @EnableAlert;
END;
GO
```

### 2.2.4. Create Trigger for Logging

```sql
USE master;
GO

CREATE OR ALTER TRIGGER trg_LogCimsDBALogins
ON ALL SERVER
FOR LOGON
AS
BEGIN
    DECLARE @LoginName NVARCHAR(256);
    DECLARE @Hostname NVARCHAR(256);
    DECLARE @IPAddress NVARCHAR(50);
    DECLARE @Application NVARCHAR(256);
    DECLARE @SessionID INT;
    DECLARE @EventTime DATETIME;
    DECLARE @AlertEnabled BIT;

    SET @LoginName = ORIGINAL_LOGIN(); -- The login name of the user
    SET @Hostname = HOST_NAME(); -- The name of the client computer
    SET @IPAddress = CONVERT(NVARCHAR(50), CONNECTIONPROPERTY('client_net_address')); -- Modify if needed to get actual IP
    SET @Application = APP_NAME(); -- The application name
    SET @SessionID = @@SPID; -- The session ID
    SET @EventTime = GETDATE(); -- Current time

    -- Check if alerts are enabled
    SELECT @AlertEnabled = AlertEnabled FROM AdminDB.dbo.Settings;

    IF @LoginName = 'cimsdba'
    BEGIN
        -- Log the successful login to the LoginAudit table
        INSERT INTO AdminDB.dbo.LoginAudit (EventType, LoginName, Hostname, IPAddress, Application, SessionID, EventTime)
        VALUES ('LOGIN_SUCCESS', @LoginName, @Hostname, @IPAddress, @Application, @SessionID, @EventTime);

        -- Send an alert if enabled
        IF @AlertEnabled = 1
        BEGIN
            DECLARE @Subject NVARCHAR(100);
            DECLARE @Body NVARCHAR(MAX);
            DECLARE @ServerName NVARCHAR(100);

            -- Get the server name
            SET @ServerName = @@SERVERNAME;

            -- Set the subject
            SET @Subject = 'ALERT: ' + @ServerName + ' - cimsdba login attempt succeeded at ' + CONVERT(NVARCHAR(50), @EventTime, 120);

            -- Build the email body
            SET @Body = '<h1>Unauthorized Access Attempt</h1>' +
                        'Dear Team,<br><br>' +
                        'It has come to our attention that an unauthorized access attempt was made using the "cimsdba" account on "' + @ServerName + '" server.<br><br>' +
                        'As per the compliance guidelines, the use of the "cimsdba" login is strictly prohibited for general access, and this incident has triggered a non-compliance alert.<br><br>' +
                        'Below are the details of the unauthorized attempt:<br><br>' +
                        '<table border="1">' +
                        '<tr><th>Event Type</th><th>Login Name</th><th>Hostname</th><th>IPAddress</th>' +
                        '<th>Application</th><th>Session ID</th><th>Event Time</th></tr>' +
                        '<tr><td>LOGIN_SUCCESS</td>' +
                        '<td>' + @LoginName + '</td>' +
                        '<td>' + @Hostname + '</td>' +
                        '<td>' + @IPAddress + '</td>' +
                        '<td>' + @Application + '</td>' +
                        '<td>' + CONVERT(NVARCHAR(10), @SessionID) + '</td>' +
                        '<td>' + CONVERT(NVARCHAR(20), @EventTime, 120) + '</td>' +
                        '</tr>' +
                        '</table>' +
                        '<br>Regards,<br>DBA Team';

            -- Send the email
            EXEC msdb.dbo.sp_send_dbmail
                @profile_name = 'CIMSAlerts',
                @recipients = 'vinith.ankam@cloudimsystems.com', -- Replace with actual recipient
                @subject = @Subject,
                @body = @Body,
                @body_format = 'HTML'; -- Set body format to HTML
        END
    END
END;
GO
```

### 2.2.5. Query to Retrieve Login Audit Logs

```sql
USE AdminDB;
GO

SELECT
    EventID,
    EventType,
    LoginName,
    Hostname,
    IPAddress,
    Application,
    SessionID,
    EventTime
FROM
    dbo.LoginAudit
WHERE
    EventTime > DATEADD(HOUR, -24, GETDATE()) -- Last 24 hours
ORDER BY
    EventTime DESC;
```
![image](https://github.com/user-attachments/assets/5f9d0d96-9a9f-48a9-9cc7-cf2462380ac8)


## 3. Implementation Steps

1. **Create Tables**: Execute the SQL scripts to create the `LoginAudit` and `Settings` tables.
2. **Create Stored Procedure**: Implement the `SetAlertStatus` stored procedure.
3. **Create Trigger**: Set up the `trg_LogCimsDBALogins` trigger on the server to monitor logins.
4. **Configure Email Profile**: Ensure that the `CIMSAlerts` database mail profile is correctly configured and operational.
5. **Test the System**: Log in as the `cimsdba` user to validate that events are recorded and alerts are sent.
6. **Monitor Logs**: Run the query to retrieve audit logs and review entries.

## 4. Testing and Validation

- Log in using the `cimsdba` account and verify that the entry is created in the `LoginAudit` table.
- Check the specified email address to confirm that the alert was received.
- Change the alert status using the `SetAlertStatus` procedure and observe if alerts are sent accordingly.
- **Verify Email Alerts**: Check your email to confirm that you received the alert with the modified subject and body.
- **Manage Alert Settings**:
    - **Enable Alerts**:
        
        ```sql
        EXEC AdminDB.dbo.SetAlertStatus @EnableAlert = 1;
        ```
        
    - **Disable Alerts**:
        
        ```sql
        EXEC AdminDB.dbo.SetAlertStatus @EnableAlert = 0;
        ```
##  Alert Mail Results:
![image](https://github.com/user-attachments/assets/41779e33-ce25-4b27-ac84-b73c01ef2da8)

## 5. Conclusion

This POC demonstrates a working login auditing system for the `cimsdba` account in SQL Server. The implemented components ensure effective monitoring and alerting on access attempts, facilitating better security and compliance management.
