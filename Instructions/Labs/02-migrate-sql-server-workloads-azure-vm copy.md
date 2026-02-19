---
lab:
    title: 'Migrate SQL Server databases to SQL Server on Azure Virtual Machine'
---

# Migrate SQL Server databases to SQL Server on Azure Virtual Machine

In this exercise, you'll learn how to migrate a SQL Server database to a SQL Server running on an Azure Virtual Machine using the backup and restore method with Azure Blob Storage. You'll back up the source database to an Azure Storage Account, and then restore it on the target SQL Server on Azure VM. This is a straightforward offline migration approach that's well-suited for smaller databases or when downtime is acceptable.

This exercise will take approximately **45** minutes.

> **Note**: To complete this exercise, you need access to an Azure subscription to create Azure resources. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?azure-portal=true) before you begin.

## Before you start

To run this exercise, you'll need:

| Item | Description |
| --- | --- |
| **Target Server** | A SQL Server on an Azure Virtual Machine. To learn more, visit [Provision a SQL Server on an Azure Virtual Machine](https://microsoftlearning.github.io/dp-300-database-administrator/Instructions/Labs/01-provision-sql-vm.html). **Note:** The SQL Server version between target and server must be the same. |
| **Source Server** | The latest [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) version installed on a server of your choice. |
| **Source Database** | The lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) database to be restored on the SQL Server instance. |
| **SSMS** | Install [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) on both the source and target servers for database backup, restore, and verification. |

## Restore a SQL Server database

Let's restore the *AdventureWorksLT* database on the SQL Server instance. This database will serve as the source database for this lab exercise. You can skip these steps if the database is already restored.

1. Select the Windows Start button and type SSMS. Select **Microsoft SQL Server Management Studio 18** from the list.  

1. When SSMS opens, notice that the **Connect to Server** dialog will be pre-populated with the default instance name. Select **Connect**.

1. Select the **Databases** folder, and then **New Query**.

1. In the new query window, copy and paste the below T-SQL into it. Ensure that the database backup file name and path match your actual backup file. If they don't, the command will fail. Execute the query to restore the database.

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\<FolderName>\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\<FolderName>\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\<FolderName>\AdventureWorksLT2019.ldf';
    ```

    > **Note**: Make sure you have the lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) backup file on the SQL Server machine before running the T-SQL command.

1. You should see a successful message after the restore is complete.

## Provision an Azure Storage Account with a blob container

The purpose of creating an Azure Storage Account is to store the full and transaction log backups for the migration. We use the storage account later on in this exercise.

1. Sign in to the [Azure portal](https://portal.azure.com/).
1. From the left portal menu, select **Storage accounts** to display a list of your storage accounts. If the portal menu isn't visible, select the menu button to toggle it on.
1. On the **Storage accounts** page, select **Create**.
1. Under **Project details**, select the same Azure subscription in which you created the Azure Virtual Machine.
1. Select the same resource group in which you created the Azure Virtual Machine. 
1. Choose a unique name for your storage account, and select the same region  as the Azure Virtual Machine.
1. Choose **Standard** as the level of service.
1. Leave all remaining options as their default values.
1. Select **Review + create**, then select **Create**.

Once the storage account is created, you can create a container by following these steps:

1. In the Azure portal, navigate to your newly created storage account.
1. In the left menu for the storage account, scroll to **Blob service** and select **Containers**.
1. Select **+ Container** to create a new container.
1. On the New container side page, provide the following information:
    - **Name:** *name of your preference*
    - **Public access level:** Private
1. Select **Create**.

## Create a SQL Server credential for the storage account

Before you can back up the source database to Azure Blob Storage, you need to create a credential that stores the storage account access key information.

1. Open SSMS and connect to your **source** SQL Server instance.

1. Open a **New Query** window and run the following T-SQL statement to create a credential. Replace the placeholder values with your storage account name and access key.

    ```sql
    CREATE CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/<container_name>]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<SAS token>';
    ```

    > **Note:** To generate a SAS token, navigate to your storage account in the Azure portal, select **Shared access signature** under **Security + networking**, grant **Blob** and **Container** permissions with **Read**, **Write**, **List**, and **Create**, set an appropriate expiry date, and then select **Generate SAS and connection string**. Copy the **SAS token** value (without the leading `?`).

## Back up the source database to Azure Blob Storage

Now you'll back up the source database directly to the Azure Blob Storage container using the SQL Server backup to URL feature.

1. In SSMS, connected to your **source** SQL Server, open a **New Query** window and run the following T-SQL statement:

    ```sql
    BACKUP DATABASE AdventureWorksLT
    TO URL = 'https://<storage_account_name>.blob.core.windows.net/<container_name>/AdventureWorksLT.bak'
    WITH COMPRESSION, STATS = 10;
    ```

    > **Note:** Replace `<storage_account_name>` and `<container_name>` with your actual values. The `COMPRESSION` option reduces the backup size and speeds up the transfer.

1. Verify the backup completed successfully. You should see a message indicating the database was successfully backed up.

1. You can also verify the backup file exists by navigating to your storage account container in the Azure portal.

## Restore the database on the target SQL Server on Azure VM

Now you'll restore the backup file from Azure Blob Storage to the target SQL Server running on the Azure Virtual Machine.

1. Open SSMS and connect to your **target** SQL Server on the Azure VM.

1. Open a **New Query** window and run the following T-SQL statement to create the same credential on the target server:

    ```sql
    CREATE CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/<container_name>]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<SAS token>';
    ```

    > **Note:** Use the same SAS token you generated earlier. Ensure it has not expired.

1. Run the following T-SQL statement to restore the database from the backup file:

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM URL = 'https://<storage_account_name>.blob.core.windows.net/<container_name>/AdventureWorksLT.bak'
    WITH REPLACE, STATS = 10;
    ```

    > **Note:** The `REPLACE` option overwrites an existing database with the same name. If the database does not exist on the target, you can omit this option. The restore may take a few minutes depending on the database size and network speed.

1. Verify the restore completed successfully. You should see a message indicating the database was successfully restored.

## Verify the migration

1. In SSMS, connected to the **target** SQL Server on Azure VM, expand **Databases** in **Object Explorer** and verify the **AdventureWorksLT** database appears.

1. Right-select the **AdventureWorksLT** database and select **New Query**. Run sample queries to verify the data was migrated correctly:

    ```sql
    -- Verify table count
    SELECT COUNT(*) AS TableCount
    FROM AdventureWorksLT.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE';

    -- Verify row counts for a sample table
    SELECT COUNT(*) AS RowCount
    FROM AdventureWorksLT.SalesLT.Customer;
    ```

1. Compare the results with the same queries executed against the source database to confirm data integrity.

You've successfully migrated a SQL Server database to a SQL Server running on an Azure Virtual Machine using the backup and restore method with Azure Blob Storage. This approach uses SQL Server's native backup to URL feature to transfer the database through an Azure Storage Account, making it simple and efficient for offline migrations.

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in additional costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about SQL Server on Azure Virtual Machines, see [What is SQL Server on Azure Virtual Machines?](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql-vm).
