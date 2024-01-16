---
lab:
    title: 'Migrate SQL Server databases to Azure SQL Managed Instance by using Log Replay Service'
---

# Migrate SQL Server databases to Azure SQL Managed Instance by using Log Replay Service

In this exercise, you'll learn how to migrate a SQL Server database to Azure SQL Managed Instance by using Log Replay Service. 

You'll start by deploying an Azure SQL Managed Instance. Then, you'll use Log Replay Services to perform an online migration of a SQL Server database to Azure SQL Managed Instance. You'll also learn how to monitor the migration process in PowerShell.

This exercise takes approximately **45** minutes.

> **Note**: To complete this exercise, you need access to an Azure subscription to create Azure resources. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?azure-portal=true) before you begin.

## Before you start

To run this exercise, you need:

| Item | Description |
| --- | --- |
| **Target server** | An Azure SQL Managed Instance. We'll create it during this exercise.|
| **Source server** | An instance of SQL Server 2019 or a [newer version](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) installed on a server of your preference. |
| **Source database** | The lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) database to be restored on the SQL Server instance. |
| **Azure Data Studio** | Install [Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio) in the same server where the source database is located. If it's already installed, update it to make sure that you’re using the most recent version. |

## Restore a SQL Server database

Let's restore the *AdventureWorksLT* database on the SQL Server instance. This database serves as the source database for this lab exercise. You can skip these steps if the database is already restored.

1. Select the Windows Start button and type SSMS. Select **Microsoft SQL Server Management Studio 18** from the list.  

1. When SSMS opens, notice that the **Connect to Server** dialog pre-populates with the default instance name. Select **Connect**.

1. Select the **Databases** folder, and then **New Query**.

1. In the new query window, copy and paste the following T-SQL. Execute the query to restore the database.

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\LabFiles\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\LabFiles\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\LabFiles\AdventureWorksLT2019.ldf';
    ```

    > **Note**: Ensure that the database backup file name and path in the above example match your actual backup file. If they don’t, the command may fail.

1. You should see a successful message after the restore is complete.

## Deploy an Azure SQL Managed Instance

Create an Azure SQL Managed Instance by following these steps:

1. Sign in to the [Azure portal](https://portal.azure.com/learn.docs.microsoft.com?azure-portal=true), and then select **Create a resource** in the upper-left corner.
1. Search for **managed instance**, select **Azure SQL managed instance**, and then select **Create**.
1. Fill out the SQL managed instance form using the information in the following table:

    |  | Suggested value |
    |---|---|
    | **Subscription** | Your subscription. |
    | **managed instance name** | Any valid name. |
    | **managed instance admin login** | Any valid username. Don't use "serveradmin" because that's a reserved server-level role. |
    | **Password** | Any password longer than 16 characters and meeting the complexity requirements. |
    | **Time zone** | The time zone to be observed by your managed instance. |
    | **Collation** | The collation you want to use for the managed instance. If you migrate databases from SQL Server, check the source collation by using SELECT SERVERPROPERTY(N'Collation') and use that value. |
    | **Location** | The Azure region in which you want to create the managed instance. |
    | **Virtual network** | Select either "Create new virtual network" or a valid virtual network and subnet. |
    | **Enable public endpoint** | Check this option to enable a public endpoint, which then helps clients outside Azure to access the database. |
    | **Allow access from** | Select from Azure services, the internet, or no access. |
    | **Connection type** | Choose between a Proxy and a Redirect connection type. |
    | **Resource group** | A new or existing resource group. |

1. Select **Pricing tier** to size compute and storage resources, and to review the pricing tier options. 
1. When you're finished, select **Apply** to save your selection, and then select **Create** to deploy the managed instance.
1. Select the **Notifications icon** to view the status of the deployment.
1. Select **Deployment in progress** to open the managed instance window to further monitor the deployment progress.

## Create an Azure Blob Storage account and container

Create an Azure Blob Storage account in the same region as your Azure SQL Managed Instance. This is where you'll store your database backups for migration.

1. Go to the [Azure portal](https://portal.azure.com) and sign in with your account credentials.
1. In the left menu, select **All services** and search for *"Storage accounts"*. Select **Storage accounts** to open the storage accounts page.
1. On the Storage accounts page, select **+ Add** to create a new storage account.
1. In the **Basics** tab of the **Create storage account** page, select the subscription that you want to use for the storage account. Then, select the resource group that contains your Azure SQL Managed Instance.
1. Enter a unique name for the storage account. 
    
    > **Note:** The name must be between 3 and 24 characters long and can contain only lowercase letters and numbers.

1. Select the location (region) where your Azure SQL Managed Instance is located.
1. Select the performance tier for the storage account.
1. Select **BlobStorage** for the account kind for the storage account. 
1. Select **Locally-redundant storage (LRS)** for the replication option for the storage account.
1. Review and select **Review + create** to create the storage account.
1. Once the storage account is created, go to the storage account page and select the **Containers** option in the left menu. Then, select **+ Container** to create a new container. Enter a name for the container and select the public access level. 
1. Select **Create** button to create the container.

After completing these steps, you'll have an Azure Blob Storage account in the same region as your Azure SQL Managed Instance, and a container where you can store your database backups for migration.

## Back up a SQL Server database

Let's create a full backup of the *AdventureWorksLT* database on the SQL Server instance, followed by a differential and log backups with `CHECKSUM` enabled. 

1. Select the Windows Start button and type SSMS. Select **Microsoft SQL Server Management Studio 18** from the list.  
1. When SSMS opens, notice that the **Connect to Server** dialog will be prepopulated with the default instance name. Select **Connect**.
1. Select the **Databases** folder, and then **New Query**.
1. In the new query window, copy and paste the below T-SQL into it. Execute the query to restore the database.

    ```sql
    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_full.bak'
    WITH CHECKSUM;

    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_diff.dif'
    WITH DIFFERENTIAL, CHECKSUM;

    BACKUP LOG AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_log.trn'
    WITH CHECKSUM;
    ```

    > **Note**: Ensure that the file path in the above example match your actual file path. If they don’t, the command may fail.

1. You should see a successful message after the restore is complete.
1. If you're running a version of SQL Server (starting with SQL Server 2012 SP1 CU2 and SQL Server 2014), you can take backups from SQL Server directly to your Blob Storage account by using the native SQL Server `BACKUP TO URL` option. 

    ```sql
    CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/<containername>] 
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',  
    SECRET = '<SAS_TOKEN>';  
    GO
    
    -- Take a full database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_full.bak'
    WITH INIT, COMPRESSION, CHECKSUM
    GO
    
    -- Take a differential database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_diff.bak'  
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM
    GO
    
    -- Take a transactional log backup to a URL
    BACKUP LOG [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_log.trn'  
    WITH COMPRESSION, CHECKSUM
    ```

    > **Note:** If you decide to use this option, you can skip the next **Copy backup files to Azure Storage account** section.

## Copy backup files to Azure Storage account

Now let's copy the backup files to the Azure Blob Storage account that you created earlier.

1. Go to the [Azure portal](https://portal.azure.com) and sign in with your account credentials.
1. In the left-hand menu, select **Storage accounts** and then select the storage account that you created earlier.
1. In the storage account overview page, scroll down to the **Blob service** section and select **Containers**. Select the container you created earlier.
1. Select **Upload** at the top of the container page. In the **Upload blob** page, select **Folder** to select the folder containing the backup files, or select **Files** to choose individual backup files. Once you have selected the files, select **Upload** to start the upload process.

## Validate access

It's important to validate if your SQL Server and your SQL Managed Instance can access your Blob Storage account successfully. For this run a sample test query to determine if your managed instance is able to access the backup in the container.

1. Connect on your SQL Managed Instance via SSMS.
1. Open a new Query editor, and run the command.

```sql
CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/databases] 
WITH IDENTITY = 'SHARED ACCESS SIGNATURE' 
, SECRET = '<sastoken>' 

RESTORE HEADERONLY 
FROM URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<backup_file_name>.bak'
```
1. Repeat this process connected on your SQL Server instance.

## Use Log Replay Service to restore backup files

You'll use the Log Replay Service (LRS) to restore the backup files from Azure Blob Storage to your Azure SQL Managed Instance. LRS is a free service that is based on SQL Server log-shipping technology.

1. In the storage account overview page, scroll down to the **Blob service** section and select **Containers**. Select the container where the backup files are stored.
1. Select **Generate SAS** at the top of the container page. In the **Generate shared access signature** page, select the permissions that you want to grant, set the start and expiry time for the SAS token, and then select **Generate SAS and connection string**. The SAS token will be displayed in the **SAS token** field, then copy it.
1. Use PowerShell to connect to your Azure account by running the `Connect-AzAccount` cmdlet.

    ```powershell
    Login-AzAccount
    Select-AzSubscription -SubscriptionId <subscription ID>
    ```

1. Use the `Start-AzSqlInstanceDatabaseLogReplay` cmdlet to start the Log Replay Service for the database that you want to restore. You'll need to provide the resource group name, instance name, database name, storage container URI, and the SAS token that you copied earlier.

```PowerShell
Import-Module Az.Sql

Start-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "YourResourceGroupName" -InstanceName "YourInstanceName" -Name "YourDatabaseName" -StorageContainerUri "https://yourstorageaccount.blob.core.windows.net/yourcontainer" -StorageContainerSasToken "YourSasToken"
```

## Monitor the migration progress

You can use the `Get-AzSqlInstanceDatabaseLogReplay` cmdlet to monitor the progress of the Log Replay Service. This cmdlet returns information about the current status of the service, including the last log backup file that was restored.

1. Run the following PowerShell code.

```powershell
# Import the Az.Sql module
Import-Module Az.Sql

# Set the resource group name, instance name, and database name
$resourceGroupName = "YourResourceGroupName"
$instanceName = "YourInstanceName"
$databaseName = "YourDatabaseName"

# Get the log replay status
$logReplayStatus = Get-AzSqlInstanceDatabaseLogReplay -ResourceGroupName $resourceGroupName -InstanceName $instanceName -Name $databaseName

# Display the log replay status
$logReplayStatus | Format-List
```

## Performing migration cutover

After the full database backup is restored on the target instance of Azure SQL Database managed instance, the database is available for a migration cutover.

1. When you're ready to complete the online database migration, select **Start Cutover**.
1. Stop all the incoming traffic to source databases.
1. Take the tail-log backup, make the backup file available in the SMB network share, and then wait until this final transaction log backup is restored.
1. At that point, you'll see **Pending changes** set to 0.
1. Select **Confirm**, and then select **Apply**.

    ![Migration cutover screen](../media/3-migration-cutover-screen.png)

1. When the database migration status shows **Completed**, connect your applications to the new target instance of Azure SQL Database managed instance.