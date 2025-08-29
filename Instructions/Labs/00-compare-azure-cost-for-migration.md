---
lab:
    title: 'Compare Azure costs for migration'
---

# Compare Azure costs for migration

The Azure Pricing Calculator is a useful tool that you can use during a data platform modernization project to estimate the costs of different Azure services and migration approaches.

In your global retailer, the data platform modernization project is expected to realize significant savings, but the board of directors has asked you to estimate the costs of different Azure migration options as precisely as possible.

Here, you'll calculate the estimated costs of migrating to Azure by using the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/).

This exercise will take approximately **30** minutes.

## Calculate the estimated Azure costs

1. Open a new browser tab and navigate to [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/).
1. The Azure Pricing Calculator helps you estimate costs for Azure services. We'll calculate the cost of migrating your database workload to Azure.

## Add the database workload

1. In the **Products** section, locate and select **Azure SQL Database** (you can use the search or browse the **Databases** category).
1. In the **Azure SQL Database** configuration panel that appears further down the page, enter these values:

    | Property | Value |
    | --- | --- |
    | Region | **East US** (or your preferred region) |
    | Type | **Single Database** |
    | Purchase model | **vCore** |
    | Service tier | **General Purpose** |
    | Compute tier | **Provisioned** |
    | Hardware Type | **Standard-series (Gen5)** |
    | Instance | **4 vCore** |
    | Disaster Recovery | **Primary or Geo replica** |
    | Compute | **Locally Redundant** |
    | Storage | **32 GB** |
    | Backup storage | **RA-GRS** |

1. Review the monthly cost estimate displayed for the SQL Database.

## Add a Virtual Machine for comparison

1. Back in the **Products** section, locate and select **Virtual Machines**.
1. In the **Virtual Machines** configuration panel, enter these values:

    | Property | Value |
    | --- | --- |
    | Region | **East US** (same as database) |
    | Operating System | **Windows** |
    | Type | **SQL Server** |
    | Tier | **Standard** |
    | Instance | **D4s v3** (4 vCPUs, 16 GB RAM) |
    | Virtual Machines | **1** |
    | License | **SQL Standard** |

1. Expand **Managed Disks**, and add:
   - Tier: **Premium SSD**, **128 GB**

## Add storage for backups

1. In the **Products** section, locate and select **Storage Accounts**.
1. In the **Storage Accounts** configuration panel, enter these values:

    | Property | Value |
    | --- | --- |
    | Region | **East US** |
    | Type | **Block Blob Storage** |
    | Performance | **Standard** |
    | Storage Account Type | **General Purpose V2** |
    | File Structure | **Flat Namespace** |
    | Access tier | **Hot** |
    | Redundancy | **LRS** |
    | Capacity | **1 TB** |

## Review and compare costs

1. Review the total estimated monthly costs for all the services you've added:
   - **SQL Database**: Managed database service costs
   - **Virtual Machines**: Infrastructure costs for SQL Server on VM
   - **Storage Accounts**: Backup and additional storage costs

1. Consider the following questions as you review the estimates:
   - How do the costs compare between SQL Database (PaaS) and SQL Server on VM (IaaS)?
   - What are the trade-offs between managed services vs. infrastructure services?
   - How might these costs scale with your workload requirements?

1. You can save this estimate by selecting **Save** or **Export** to share with stakeholders.

## Explore cost optimization options

1. In each service configuration, explore different options to see how they impact costs:
   - **SQL Database**: Try different service tiers (Basic, Standard, Premium) and compute sizes
   - **Virtual Machines**: Compare different VM sizes and storage options
   - **Storage**: Compare different redundancy options and access tiers

1. Note how changing these options affects your overall monthly estimate.

You've used the Azure Pricing Calculator to estimate costs for migrating Adatum Corporation's Accounting server and its associated databases to different Azure services. This gives you a foundation for comparing infrastructure-as-a-service (IaaS) vs. platform-as-a-service (PaaS) options and planning your migration budget.

