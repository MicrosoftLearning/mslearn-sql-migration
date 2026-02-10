---
lab:
    title: 'Identify compatibility issues for SQL migration'
---

# Identify compatibility issues for SQL migration

In our scenario, you've been asked to assess the readiness of a legacy SQL Server database for migration to Azure SQL Database. Your task is to perform an assessment of their legacy database and identify any potential compatibility issues or changes that need to be made before migration. You should also review the schema of the database and identify any features or configurations that aren't supported in Azure SQL Database.

This exercise will take approximately **20** minutes.

> **Note**: To complete this exercise, you need access to an Azure subscription to create Azure resources. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?azure-portal=true) before you begin.

## Before you start

To run this exercise, ensure you have the following in place before proceeding:

- You’ll need SQL Server 2019 or a later version, along with the [**AdventureWorksLT**](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) lightweight database that is compatible with your specific SQL Server instance.
- An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?azure-portal=true).
- A SQL user with read access to the source database.

## Restore a SQL Server database and run a command

1. Select the Windows Start button and type SSMS. Select **Microsoft SQL Server Management Studio** from the list.  

1. When SSMS opens, notice that the **Connect to Server** dialog will be pre-populated with the default instance name. Select **Connect**.

1. Select the **Databases** folder, and then **New Query**.

1. In the new query window, copy and paste the below T-SQL into it. Ensure that the database backup file name and path match your actual backup file. If they don’t, the command will fail. Execute the query to restore the database.

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

1. Run the following command on the **AdventureWorksLT** database in the SQL Server instance.

```sql
ALTER TABLE [SalesLT].[Customer] ADD [Next] VARCHAR(5);
```

## Set up Azure Database Migration Service

Azure Database Migration Service (DMS) enables seamless migration of your databases to Azure. In this section, you'll create a DMS instance and set up a self-hosted integration runtime to connect to your on-premises SQL Server.

1. Open a browser and navigate to the [Azure portal](https://portal.azure.com).

1. In the Azure portal search bar, type **Azure Database Migration Services** and select it from the results.

1. Select **+ Create** to create a new Database Migration Service.

1. On the **Select migration scenario and Database Migration Service** page, select the following:
    - **Source server type**: SQL Server
    - **Target server type**: Azure SQL Database
    - Select **Database Migration Service**, and then select **Select**.

1. On the **Create Migration Service** page, fill in the following details:
    - **Subscription**: Select your Azure subscription.
    - **Resource group**: Create a new resource group or select an existing one.
    - **Location**: Select the region closest to you.
    - **Service name**: Enter `AdventureWorksDMS`.

1. Select **Review + Create**, and then select **Create**. Wait for the deployment to complete.

1. Once created, navigate to your Database Migration Service resource.

1. Under **Settings**, select **Integration runtime**.

1. Select **Configure integration runtime** and copy one of the **Authentication keys** displayed.

1. Select the **Download and install integration runtime** link to download the installer.

1. Run the installer on your local machine (where SQL Server is installed or on a machine that has network access to SQL Server). Follow the installation wizard using default settings.

1. When Microsoft Integration Runtime Configuration Manager opens after installation, paste the **Authentication key** you copied earlier and select **Register**.

1. Select **Finish** to complete the registration. Wait a few moments for the status to show as **Connected** in the Azure portal.

    > **Note**: The self-hosted integration runtime enables Azure Database Migration Service to securely connect to your on-premises SQL Server instance.

## Run the compatibility assessment

The compatibility assessment helps identify potential migration issues and provides detailed guidance on how to resolve them before the migration process begins. This can save significant time and resources. 

You'll use Azure Database Migration Service in the Azure portal to run the compatibility assessment and view the results for an Azure SQL Database target.

1. In your Azure Database Migration Service resource, select **+ New Migration** from the overview page.

1. On the **Select migration scenario** page, select:
    - **Source server type**: SQL Server
    - **Target server type**: Azure SQL Database
    - **Migration mode**: Offline migration
    - Select **Select**.

1. On the **Source details** tab, enter your source SQL Server connection details:
    - **Source SQL Server instance name**: Enter the name of your SQL Server instance (e.g., `localhost` or the machine name).
    - **Authentication type**: Select the appropriate authentication.
    - **Username** and **Password**: Enter credentials with read access to the database.

1. Select **Next: Connect to source SQL Server** and verify the connection is successful.

1. On the **Select databases** tab, select the *AdventureWorksLT* database and select **Next**.

1. On the **Select target** tab, select your Azure subscription and select an Azure SQL Database as the target. If you don't have one, use the following steps to create it:

    1. In a new browser tab, go to the [Azure portal](https://portal.azure.com) and search for **SQL databases**. Select **+ Create**.
    1. Select the same **Resource group** you used for DMS, enter `AdventureWorksTarget` as the **Database name**, and under **Server**, select **Create new**. Enter a unique server name, set **Location** to the same region as DMS, select **Use SQL authentication**, and provide an admin login and password. Select **OK**.
    1. On the **Compute + storage** setting, select **Configure**, and choose the **Basic** or **Free** tier for cost savings. Select **Review + create**, then **Create**. Wait for the deployment to complete.
    1. Return to the DMS migration wizard tab and select your newly created Azure SQL Database.

    Select **Next**.

1. On the **Summary** tab, review the compatibility assessment results before proceeding with the migration.

## Review the assessment results

You are now able to review the compatibility findings generated by the Azure Database Migration Service.

1. On the **Summary** tab, review the **Assessment results** section. Select the *AdventureWorksLT* database to view detailed findings.
    
    > **Note:** We can see that the `Next` column that was added previously was flagged, as it may lead to an error in Azure SQL Database. The column name `NEXT` is a reserved keyword.

1. Review the list of compatibility issues. Each issue includes:
    - **Issue description**: What the problem is.
    - **Recommendation**: How to resolve it before migration.
    - **Affected objects**: The specific database objects impacted.

1. Cancel the migration wizard and go back to the DMS overview. Start a new migration but this time select **Azure SQL Managed Instance** as the target type.
    
    > **Note:** The `Next` column is no longer flagged for Azure SQL Managed Instance, why is that? 
    >
    >This means that the `Next` column can be safely used on Azure SQL Managed Instance because it has broader compatibility with SQL Server features.

## Fix the issue

1. Run the following T-SQL command on the *AdventureWorksLT* database.

    ```sql
    ALTER TABLE [SalesLT].[Customer] DROP COLUMN [Next];
    ```

1. Go back to your Azure Database Migration Service in the Azure portal.

1. Start a new migration for the *AdventureWorksLT* database with **Azure SQL Database** as the target type.

1. Review the assessment results on the **Summary** tab.

    > **Note:** The database is now ready to migrate. No compatibility issues are blocking the migration to Azure SQL Database.

You've learned how to assess the readiness of a SQL Server database for migration to Azure SQL Database. By addressing compatibility issues and making essential schema changes or reporting them, you've taken an important step in mitigating potential technical issues that could arise in the future on Azure SQL Database.
