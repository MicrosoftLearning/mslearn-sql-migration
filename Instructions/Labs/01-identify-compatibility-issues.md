---
lab:
    title: 'Identify compatibility issues for SQL migration'
    description: In this exercise, you'll migrate a legacy SQL Server database to Azure SQL Database and identify compatibility issues.
    level: 300
    duration: 20 minutes
    islab: true
    primarytopics: 
        - Azure
        - Azure SQL Database
        - SQL Server Migration
---

# Identify compatibility issues for SQL migration

In our scenario, you've been asked to migrate a legacy SQL Server database to Azure SQL Database. Your task is to perform the migration and identify any compatibility issues that surface as deployment errors. You should also review the schema of the database and identify any features or configurations that aren't supported in Azure SQL Database.

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

1. Run the following command on the **AdventureWorksLT** database in the SQL Server instance. This creates a view that references a table in another database, which isn't supported in Azure SQL Database.

```sql
CREATE VIEW [SalesLT].[vServerPrincipals]
AS
    SELECT name, create_date FROM master.sys.server_principals;
GO
```

## Set up Azure Database Migration Service

Azure Database Migration Service (DMS) enables seamless migration of your databases to Azure. In this section, you'll create a DMS instance and set up a self-hosted integration runtime to connect to your on-premises SQL Server.

1. Open a browser and navigate to the [Azure portal](https://portal.azure.com).

1. In the Azure portal search bar, type **Azure Database Migration Services** and select it from the results.

1. Select **Start a new migration** to create a new Database Migration Service.

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

## Create SQL database

If you don't have one, use the following steps to create a SQL databse in your Azure Portall:

1. In a new browser tab, go to the [Azure portal](https://portal.azure.com) and search for **SQL databases**. Select **+ Create**.

1. Select the same **Resource group** you used for DMS, enter `AdventureWorksTarget` as the **Database name**, and under **Server**, select **Create new**. Enter a unique server name, set **Location** to the same region as DMS, select **Use SQL authentication**, and provide an admin login and password. Select **OK**.

1. On the **Compute + storage** setting, select **Configure**, and choose the **Basic** or **Free** tier for cost savings. 

1. Select **Review + create**, then **Create**. Wait for the deployment to complete.

## Migrate the database

Now you'll use Azure Database Migration Service to migrate the database to Azure SQL Database. During this process, any incompatible objects will produce deployment errors that reveal compatibility issues.

1. In your Azure Database Migration Service resource, select **+ New Migration** from the overview page.

1. On the **Select migration scenario** page, select:
    - **Source server type**: SQL Server
    - **Target server type**: Azure SQL Database
    - **Migration mode**: Offline

1. Then select **Next**.

1. On the **Source details** tab:
   - **Is your source SQL Server instance tracked in Azure**: Select **No** (unless using Azure VM or Azure Arc).
   - **Source infrastructure type**: Select **Other**.
   - **Source SQL Server instance name**: Enter the exact SQL Server instance name
     (e.g. `localhost`, `localhost\SQLEXPRESS`, or `(localdb)\MSSQLLocalDB`).

1. Then select **Next**.

1. On the **Connect to source SQL Server** tab:
   - **Source server name**: Use the same value as the SQL Server instance name.
   - **Authentication type**: Select your authentication type. Windows Authentication recommended.
   - **Username and Password**: Use an account with at least `db_owner` permissions on the source database.
   - **Connection properties**: Unselect **Encrypt connection** and **Trust server certificate**.

1. Then select **Next**.

1. On the **Select databases for migration** tab, select the *AdventureWorksLT* database.

1. Then select **Next**.

1. On the **Connect to target Azure SQL Database** tab, select your Azure subscription if not already selected, and then select the target Azure SQL Database you created earlier with the proper username and password.

1. Then select **Next**.

1. On the **Map source and target databases** tab, select the `AdventureWorksTarget` database you created earlier.

1. Then select **Next**.

1. On the **Select database tables to migrate** tab, select the tables you want to migrate. For this exercise, select all tables and make sure the option **Migrate missing schema** is selected.

1. Then select **Next**.

1. On the **Database migration summary** tab, review the migration details before proceeding.

1. Then select **Start migration**.

## Review the migration results

1. On the DMS overview page, select the migration to view its progress.

1. Wait until the migration status shows **Succeeded**. You can select the source database name to view the migration details and progress.

    > **Note:** The migration may take a few minutes to complete depending on network connectivity and the amount of data.

1. Review the column **Deployment errors**. You should see the following error for the `vServerPrincipals` view:

    > **Deployed failure: Reference to database and/or server name in 'master.sys.server_principals' is not supported in this version of SQL Server. Object element: [SalesLT].[vServerPrincipals].**

    This error confirms that cross-database references aren't supported in Azure SQL Database. The view references `master.sys.server_principals`, which exists outside the current database boundary.

1. Review the deployment errors. Each error includes details about the incompatible object and why it failed to deploy on Azure SQL Database.

    > **Note:** Even though there was a deployment error caused by a compatibility issue, the migration still completed successfully and migrated all remaining objects. Incompatible objects are skipped, but the rest of the database is migrated as expected.

## Fix the issue (optional)

1. Run the following T-SQL command on the *AdventureWorksLT* database.

    ```sql
    DROP VIEW [SalesLT].[vServerPrincipals];
    ```

1. Go back to your Azure Database Migration Service in the Azure portal.

1. Start a new migration for the *AdventureWorksLT* database with **Azure SQL Database** as the target type. Make sure you create a new target database or drop all the objects in the target database before starting the new migration.

1. You should now see that the migration completes successfully without any deployment errors.

You've learned how to identify compatibility issues in a SQL Server database when migrating to Azure SQL Database. By reviewing deployment errors, addressing incompatible objects, and making essential schema changes, you've taken an important step in ensuring a successful migration to Azure SQL Database.
