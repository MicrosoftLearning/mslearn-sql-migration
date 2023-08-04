---
lab:
    title: 'Migrate SQL Server databases to SQL Server on Azure Virtual Machine'
---

# Migrate SQL Server databases to SQL Server on Azure Virtual Machine

In this exercise, you'll learn how to migrate a SQL Server database to a SQL Server running on an Azure Virtual Machine using the Azure migration extension for Azure Data Studio. You'll start by installing and launching the Azure migration extension for Azure Data Studio. Then, you'll perform an online migration of a SQL Server database to a SQL Server running on an Azure Virtual Machine. You'll also learn how to monitor the migration process on the Azure portal and complete the cutover process to finalize the migration.

This exercise will take approximately **45** minutes.

> **Note**: To complete this exercise, you need access to an Azure subscription to create Azure resources. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?azure-portal=true) before you begin.

## Before you start

To run this exercise, you'll need:

| Item | Description |
| --- | --- |
| **Target Server** | A SQL Server on Azure Virtual Machine. To learn more, visit [Provision a SQL Server on an Azure Virtual Machine](https://microsoftlearning.github.io/dp-300-database-administrator/Instructions/Labs/01-provision-sql-vm.html) |
| **Source Server** | The latest [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) version installed on a server of your choice. |
| **Source Database** | The lightweight [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) database to be restored on the SQL Server 2022 instance. |

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

## Install and launch the Azure migration extension for Azure Data Studio

Before start using the Azure migration extension, you need to install [Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio). For this scenario, install Azure Data Studio in the same server where the source database is located. The extension is available in Azure Data Studio marketplace.

To install the migration extension, follow these steps:

1. Open the extensions manager in Azure Data Studio.
1. Search for ***Azure SQL Migration*** and select the extension.
1. Install the extension. Once you install it, the Azure SQL Migration extension is in the list of installed extensions.
1. Connect to a SQL Server instance in Azure Data Studio.
1. To launch the Azure migration extension, right-click on the source instance name and select **Manage** to access the dashboard and the landing page of the Azure SQL Migration extension.

## Perform an online migration of a SQL Server database to a SQL Server running on an Azure Virtual Machine

To perform a minimal downtime migration using Azure Data Studio, follow these steps:

1. Launch the Migrate to Azure SQL wizard within the extension in Azure Data Studio.

1. On **Step 1: Databases for assessment**, select the database you want to migrate, then select **Next**.
    
    > **Note**: It's recommended to collect performance data and get right-sized Azure recommendations.

1. On **Step 2: Assessment results and recommendations**, wait for the assessment to complete, then select **SQL Server on Azure Virtual Machine** as the **Azure SQL** target.

1. At the bottom of the **Step 2: Assessment results and recommendations** page, select **View/Select** to view the assessment results. Select the database to migrate. 

    > **Note**: Take a moment to review the assessment results on the right side.

1. On **Step 3: Azure SQL target**, select an Azure account and your target SQL Server on Azure Virtual Machine.

    ![Screenshot of Azure SQL target configuration on the Azure migration extension for Azure Data Studio.](../media/3-step-azure-sql-target.png)

1. On **Step 4: Azure Database Migration Service**, create a new Azure Database Migration Service using the Azure Data Studio wizard. If you have previously created one, you can reuse it. Alternatively, you can create an Azure Database Migration Service resource through the Azure portal.

    > **Note**: Make sure the subscription is registered to use the **Microsoft.DataMigration** namespace. To learn how to perform a resource provider registration, see [Register the resource provider](https://learn.microsoft.com/azure/dms/quickstart-create-data-migration-service-portal#register-the-resource-provider).

1. Back up the source database, and copy it over to a folder of your preference on the source server.

    > **Note:** The Azure Data Migration Services will orchestrate and restore the backup files automatically at the target server.

1. On Azure portal, navigate to the storage account, then the container.

1. Select **Upload**, and then select **Folder** from the drop-down menu.

1. Choose a folder you placed the full backup file, and select **Upload**.

1. On **Step 5: Data source configuration**, select the location of your database backups, either on an on-premises network share or in an Azure Blob Storage container.

1. Start the database migration and monitor the progress in Azure Data Studio. You can also track the progress in the Azure portal under the Azure Database Migration Service resource.

1. Select **Database migrations in progress** in the migration dashboard to view ongoing migrations. 

    ![Screenshot of the migration dashboard on the Azure migration extension for Azure Data Studio.](../media/3-data-migration-dashboard.png)

1. Select the database name to get further details.

    ![Screenshot of the migration details on the Azure migration extension for Azure Data Studio.](../media/3-dashboard-details.png)

## Monitor migration on Azure portal

Alternatively, you can also monitor the migration activity using Azure Database Migration Service. 

1. 
    ![Screenshot of the monitoring page in Azure Database Migration Services in Azure portal.](../media/3-dms-azure-portal.png)
    
## Complete the cutover process

1. Take a [tail log backup](https://learn.microsoft.com/sql/relational-databases/backup-restore/tail-log-backups-sql-server) for the source database.

1. On Azure portal, upload the transaction log backup to the container and folder where the full backup file is located.

1. On the Azure migration extension, select **Complete cutover** in the monitoring page.

    ![Screenshot of the migration cutover option on Azure migration extension for Azure Data Studio.](../media/3-dashboard-details-cutover.png)

1. Verify that all log backups have been restored on the target database. The **Log backups pending restore** value should be zero. This step completes the migration.

    ![Screenshot of the migration cutover option on Azure migration extension for Azure Data Studio.](../media/3-dashboard-details-cutover-extension.png)

1. The migration status property will change to **Completing**, then to **Succeeded** after the migration is completed.

    > **Note**: You can complete the cutover using similar steps with Azure Database Migration Service through the Azure portal.

You’ve learned how to migrate a SQL Server database to a SQL Server running on an Azure Virtual Machine using the Azure migration extension for Azure Data Studio. You’ve also learned how to complete the cutover process to finalize the migration. This ensures that all data has been successfully migrated and that the new database is fully operational. Once the cutover process is complete, you can start using your new SQL Server database running on an Azure Virtual Machine. 

## Clean up

When you're working in your own subscription, it's a good idea at the end of a project to identify whether you still need the resources you created. 

Leaving resources running unnecessarily can result in additional costs. You can delete resources individually or delete the entire set of resources in the [Azure portal](https://portal.azure.com?azure-portal=true).

## More information

For more information about SQL Server on Azure Virtual Machines, see [What is SQL Server on Azure Virtual Machines?](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql-vm).
