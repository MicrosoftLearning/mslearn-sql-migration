
# Migrate SQL Server databases to Azure SQL Database

In this exercise, you'll learn how to migrate specific tables from a SQL Server database to Azure SQL Database using Azure CLI and the Azure Database Migration Service (DMS). 

> **Important**: To avoid impacting migration duration—which can be affected by network and connectivity constraints—we will migrate only a few tables (*Customer*, *ProductCategory*, *Product*, and *Address*) as an example. The same process can be used to migrate the entire schema when needed.

This exercise takes approximately **45** minutes.

> **Note**: To complete this exercise, you need access to an Azure subscription to create Azure resources. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?azure-portal=true) before you begin.

## Before you start

To run this exercise, you need:

| Item | Description |
| --- | --- |
| **Target server** | An Azure SQL Database server. We'll create it during this exercise.|
| **Target database** | A database on Azure SQL Database server. We'll create it during this exercise.|
| **Source server** | An instance of SQL Server 2019 or a [newer version](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) installed on a server of your preference. |
| **Source database** | The lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) database to be restored on the SQL Server instance. |
| **Azure CLI** | Install the [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (version 2.51.0 or later). After installation, install the **datamigration** extension by running ``az extension add --name datamigration``. |
| **SSMS** | Install [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) to run T-SQL scripts against the source and target databases. |
| **Microsoft.DataMigration** resource provider | Make sure the subscription is registered to use the **Microsoft.DataMigration** namespace. To learn how to perform a resource provider registration, see [Register the resource provider](https://learn.microsoft.com/azure/dms/quickstart-create-data-migration-service-portal#register-the-resource-provider). |
| **Microsoft Integration runtime** | Install [Microsoft Integration Runtime](https://aka.ms/sql-migration-shir-download). |

## Restore a SQL Server database

Let's restore the *AdventureWorksLT* database on the SQL Server instance. This database will serve as the source database for this lab exercise. You can skip these steps if the database is already restored.

1. Select the Windows Start button and type SSMS. Select **Microsoft SQL Server Management Studio 18** from the list.  

1. When SSMS opens, notice that the **Connect to Server** dialog will be pre-populated with the default instance name. Select **Connect**.

1. Select the **Databases** folder, and then **New Query**.

1. In the new query window, copy and paste the below T-SQL into it. Ensure that the database backup file name and path match your actual backup file. If they don’t, the command will fail. Execute the query to restore the database.

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\<FolderName>\AdventureWorksLT2025.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2025_Data' 
            TO 'C:\<FolderName>\AdventureWorksLT2025.mdf',
          MOVE 'AdventureWorksLT2025_Log'
            TO 'C:\<FolderName>\AdventureWorksLT2025.ldf';
    ```

    > **Note**: Make sure you have the lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) backup file on the SQL Server machine before running the T-SQL command.

1. You should see a successful message after the restore is complete.

## Register Microsoft.DataMigration namespace

Skip these steps if the **Microsoft.DataMigration** namespace is already registered on your subscription.

1. From the Azure portal, search for **Subscription** in the search box at the top, then select **Subscriptions**. Select your subscription on the **Subscriptions** blade.

1. On your subscription page, select **Resource providers** under **Settings**. Search for **Microsoft.DataMigration** in the search box at the top, then select **Microsoft.DataMigration**. 

    > **Note**: If the **Resource Provider Details** side bar opens, you can close it.

1. Select **Register**.

## Provision an Azure SQL Database

Let's set up an Azure SQL Database that will serve as our target environment.

1. From the Azure portal, search for **SQL databases** in the search box at the top, then select **SQL databases**.

1. On the **SQL databases** blade, select **+ Create**.

1. On the **Create SQL Database** page, select the following options.

    - **Subscription:** &lt;Your subscription&gt;
    - **Resource group:** &lt;Your resource group&gt;
    - **Database name:** AdventureWorksLT
    - **Server:** Select the **Create new** link. Provide the server details on the **Create SQL Database Server** page.
        - **Server name:** &lt;Choose a server name&gt;. Server name must be globally unique.
        - **Location:** &lt;Your region, same as your resource group&gt;
        - **Authentication method:** Use SQL authentication
        - **Server admin login:** sqladmin
        - **Password:** &lt;Your password&gt;
        - **Confirm password:** &lt;Your password&gt;
        
            **Note:** Make note of this server name, and credentials. You'll use it in subsequent tasks.

    - **Want to use SQL elastic pool?** No
    - **Workload environment:** Production

1. On **Compute + storage**, select **Configure database**. On the **Configure** page, for **Service tier** dropdown, select **Basic**, and then **Apply**.

1. For the **Backup storage redundancy** option, keep the default value: **Geo-redundant backup storage**. Select **Review + Create**.

1. Review the settings, then select **Create**.

1. Once the deployment is complete, select **Go to resource**.

## Enable access to an Azure SQL Database

Let's enable access to Azure SQL Database so we can connect to it through the client tools.

1. From the **SQL database** page, select the **Overview** section, and then select the link for the server name in the top section:

1. On the **SQL server** blade, select **Networking** under the **Security** section.

1. On the **Public access** tab, select **Selected networks**. 

1. In **Firewall rules**, select **+ Add your client IPv4 address**.

1. On **Exceptions**, check the **Allow Azure services and resources to access this server** property. 

1. Select **Save**.

## Create the target schema

Let's manually create the schema for the target tables before beginning the data migration.

1. Open SSMS and connect to your Azure SQL Database. In the **Connect to Server** dialog, enter `<server>.database.windows.net` as the server name, select **SQL Server Authentication**, and enter the admin credentials you set up earlier.

1. Copy and paste the following T-SQL script on a new **New Query** window to create the schema for the tables we'll migrate:

    ```sql
        USE [AdventureWorksLT]
        GO
        
        CREATE SCHEMA [SalesLT]
        GO
        
        CREATE TYPE [dbo].[Name] FROM [nvarchar](50) NULL
        GO
        
        CREATE TYPE [dbo].[NameStyle] FROM [bit] NOT NULL
        GO
        
        CREATE TYPE [dbo].[Phone] FROM [nvarchar](25) NULL
        GO
        
        CREATE TABLE [SalesLT].[Address](
        	[AddressID] [int] IDENTITY(1,1) NOT FOR REPLICATION NOT NULL,
        	[AddressLine1] [nvarchar](60) NOT NULL,
        	[AddressLine2] [nvarchar](60) NULL,
        	[City] [nvarchar](30) NOT NULL,
        	[StateProvince] [dbo].[Name] NOT NULL,
        	[CountryRegion] [dbo].[Name] NOT NULL,
        	[PostalCode] [nvarchar](15) NOT NULL,
        	[rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
        	[ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Address_AddressID] PRIMARY KEY CLUSTERED 
        (
        	[AddressID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Address_rowguid] UNIQUE NONCLUSTERED 
        (
        	[rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[Customer](
        	[CustomerID] [int] IDENTITY(1,1) NOT FOR REPLICATION NOT NULL,
        	[NameStyle] [dbo].[NameStyle] NOT NULL,
        	[Title] [nvarchar](8) NULL,
        	[FirstName] [dbo].[Name] NOT NULL,
        	[MiddleName] [dbo].[Name] NULL,
        	[LastName] [dbo].[Name] NOT NULL,
        	[Suffix] [nvarchar](10) NULL,
        	[CompanyName] [nvarchar](128) NULL,
        	[SalesPerson] [nvarchar](256) NULL,
        	[EmailAddress] [nvarchar](50) NULL,
        	[Phone] [dbo].[Phone] NULL,
        	[PasswordHash] [varchar](128) NOT NULL,
        	[PasswordSalt] [varchar](10) NOT NULL,
        	[rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
        	[ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Customer_CustomerID] PRIMARY KEY CLUSTERED 
        (
        	[CustomerID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Customer_rowguid] UNIQUE NONCLUSTERED 
        (
        	[rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[Product](
        	[ProductID] [int] IDENTITY(1,1) NOT NULL,
        	[Name] [dbo].[Name] NOT NULL,
        	[ProductNumber] [nvarchar](25) NOT NULL,
        	[Color] [nvarchar](15) NULL,
        	[StandardCost] [money] NOT NULL,
        	[ListPrice] [money] NOT NULL,
        	[Size] [nvarchar](5) NULL,
        	[Weight] [decimal](8, 2) NULL,
        	[ProductCategoryID] [int] NULL,
        	[ProductModelID] [int] NULL,
        	[SellStartDate] [datetime] NOT NULL,
        	[SellEndDate] [datetime] NULL,
        	[DiscontinuedDate] [datetime] NULL,
        	[ThumbNailPhoto] [varbinary](max) NULL,
        	[ThumbnailPhotoFileName] [nvarchar](50) NULL,
        	[rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
        	[ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Product_ProductID] PRIMARY KEY CLUSTERED 
        (
        	[ProductID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_Name] UNIQUE NONCLUSTERED 
        (
        	[Name] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_ProductNumber] UNIQUE NONCLUSTERED 
        (
        	[ProductNumber] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_rowguid] UNIQUE NONCLUSTERED 
        (
        	[rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[ProductCategory](
        	[ProductCategoryID] [int] IDENTITY(1,1) NOT NULL,
        	[ParentProductCategoryID] [int] NULL,
        	[Name] [dbo].[Name] NOT NULL,
        	[rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
        	[ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_ProductCategory_ProductCategoryID] PRIMARY KEY CLUSTERED 
        (
        	[ProductCategoryID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_ProductCategory_Name] UNIQUE NONCLUSTERED 
        (
        	[Name] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_ProductCategory_rowguid] UNIQUE NONCLUSTERED 
        (
        	[rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        ALTER TABLE [SalesLT].[Address] ADD  CONSTRAINT [DF_Address_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Address] ADD  CONSTRAINT [DF_Address_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_NameStyle]  DEFAULT ((0)) FOR [NameStyle]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Product] ADD  CONSTRAINT [DF_Product_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Product] ADD  CONSTRAINT [DF_Product_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] ADD  CONSTRAINT [DF_ProductCategory_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] ADD  CONSTRAINT [DF_ProductCategory_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH CHECK ADD  CONSTRAINT [FK_Product_ProductCategory_ProductCategoryID] FOREIGN KEY([ProductCategoryID])
        REFERENCES [SalesLT].[ProductCategory] ([ProductCategoryID])
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [FK_Product_ProductCategory_ProductCategoryID]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory]  WITH CHECK ADD  CONSTRAINT [FK_ProductCategory_ProductCategory_ParentProductCategoryID_ProductCategoryID] FOREIGN KEY([ParentProductCategoryID])
        REFERENCES [SalesLT].[ProductCategory] ([ProductCategoryID])
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] CHECK CONSTRAINT [FK_ProductCategory_ProductCategory_ParentProductCategoryID_ProductCategoryID]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_ListPrice] CHECK  (([ListPrice]>=(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_ListPrice]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_SellEndDate] CHECK  (([SellEndDate]>=[SellStartDate] OR [SellEndDate] IS NULL))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_SellEndDate]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_StandardCost] CHECK  (([StandardCost]>=(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_StandardCost]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_Weight] CHECK  (([Weight]>(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_Weight]
        GO
        
    ```

1. Execute the script by pressing **F5** or selecting **Run**.

1. Verify the tables have been created successfully by running:

    ```sql
    SELECT TABLE_SCHEMA, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'SalesLT';
    ```

## Perform an offline migration using Azure CLI (DMS)

We're now ready to migrate the data. Follow these steps to perform an offline migration using Azure CLI and the Azure Database Migration Service.

### Log in to Azure and set your subscription

1. Open a command prompt or PowerShell window and log in to Azure:

    ```bash
    az login
    ```

1. If you have multiple subscriptions, select the appropriate one:

    ```bash
    az account set --subscription "<Your subscription name or ID>"
    ```

### Create an Azure Database Migration Service instance

1. Create an Azure Database Migration Service instance if you don't already have one. Replace the placeholder values with your own:

    ```bash
    az datamigration sql-service create \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service" \
        --location "<Your region>"
    ```

1. Wait for the service to be created. You can verify the status by running:

    ```bash
    az datamigration sql-service show \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

### Register the self-hosted Integration Runtime

1. Retrieve the authentication keys for the DMS service:

    ```bash
    az datamigration sql-service list-auth-key \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

1. Copy one of the authentication keys returned.

1. Open the **Microsoft Integration Runtime Configuration Manager** on the machine where you installed the Integration Runtime, and register the node by pasting the authentication key. Select **Register** and then **Finish**.

1. Verify the Integration Runtime is connected to the DMS service by checking the node status:

    ```bash
    az datamigration sql-service list-integration-runtime-metric \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

    > **Note:** The node should have a status of **Online** before proceeding.

### Start the database migration

1. Start the offline migration with the following command. Replace the placeholder values with your environment-specific details:

    ```bash
    az datamigration sql-db create \
        --resource-group "<Your resource group>" \
        --sqldb-instance-name "<Target Azure SQL Server name>" \
        --target-db-name "AdventureWorksLT" \
        --migration-service "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.DataMigration/sqlMigrationServices/DMS-Migration-Service" \
        --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Sql/servers/<Target Azure SQL Server name>" \
        --source-database-name "AdventureWorksLT" \
        --source-sql-connection authentication="SqlAuthentication" data-source="<Source SQL Server IP or hostname>" encrypt-connection=true password="<Source SQL password>" trust-server-certificate=true user-name="<Source SQL username>" \
        --target-sql-connection authentication="SqlAuthentication" data-source="<Target server>.database.windows.net" encrypt-connection=true password="<Target SQL password>" user-name="sqladmin" \
        --table-list "[SalesLT].[Address]" "[SalesLT].[Customer]" "[SalesLT].[Product]" "[SalesLT].[ProductCategory]" \
        --migration-scope "SelectedTables"
    ```

    > **Note:** The `--table-list` parameter specifies the four tables we want to migrate: *Address*, *Customer*, *Product*, and *ProductCategory*. The schema was already created manually, so only data will be migrated.

### Monitor the migration

1. Monitor the migration status by running:

    ```bash
    az datamigration sql-db show \
        --resource-group "<Your resource group>" \
        --sqldb-instance-name "<Target Azure SQL Server name>" \
        --target-db-name "AdventureWorksLT"
    ```

1. Repeat the above command until the **migrationStatus** shows **Succeeded**. You can also check the status in the Azure portal by navigating to the Azure SQL Database resource and selecting **Migration** under **Data management**.

    > **Note:** The migration may take a few minutes to complete depending on network connectivity and the amount of data.

### Verify data migration

1. After the migration status is **Succeeded**, connect to your target Azure SQL Database using SSMS and run the following query to verify data migration success:

    ```sql
    -- Check row counts for migrated data
    SELECT 'Customer' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Customer]
    UNION ALL
    SELECT 'ProductCategory' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[ProductCategory]
    UNION ALL
    SELECT 'Product' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Product]
    UNION ALL
    SELECT 'Address' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Address];
    ```

You've successfully performed a selective migration of specific tables from a SQL Server database to Azure SQL Database using Azure CLI and the Azure Database Migration Service (DMS). You also learned how to monitor the migration process via the command line.

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in extra costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about Azure SQL Database, see [What is Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview).
