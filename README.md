# Crayon DPi30 Secure Synapse Deployment Accelerator

This project deploys a secure data lakehouse solution. Microsoft design principles, recommended technology choices and security best practices are all implemented with this solution. We primarily focus on the security considerations and key technical decisions.


## Before you deploy
You must have an **existing** resource group which will be used for all deployments.


### Deploying to Azure

Before you hit the "Deploy to Azure" button, make sure you review the details about the services deployed by this accelerator.

[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#blade/Microsoft_Azure_CreateUIDef/CustomDeploymentBlade/uri/https%3A%2F%2Fraw.githubusercontent.com%2FZachNichter%2FSecureSynapse%2Fmain%2Fdeploy%2Fazuredeploy.json/uiFormDefinitionUri/https%3A%2F%2Fraw.githubusercontent.com%2FZachNichter%2FSecureSynapse%2Fmain%2Fdeploy%2FcreateUiDefinition.json)

> **Note:** The "Deploy to Azure" button above will redirect you to the Azure Portal with a reference to the resulting ARM template file.

> **Important:** This deployment accelerator is meant to be executed under no interference from Azure Policies that deny certain configurations as they might prevent the its successful completion. Please use a sandbox environment if you need to validate the deployment resulting configuration before you run it against other environments under Azure Policies.

> **Important:** This deployment accelerator implements some service features that may still be in Public Preview. Please consider those before you plan a production deployment.

### Required Resource Providers

The target subscription for the deployment accelerator needs to have the following resource providers enabled before the deployment execution:

* Microsoft.Authorization/roleAssignments
* Microsoft.Compute/virtualMachines
* Microsoft.Compute/virtualMachines/extensions
* Microsoft.DevTestLab/schedules
* Microsoft.KeyVault/vaults
* Microsoft.KeyVault/vaults/privateEndpointConnections
* Microsoft.Network/bastionHosts
* Microsoft.Network/networkInterfaces
* Microsoft.Network/networkSecurityGroups
* Microsoft.Network/networkSecurityGroups/securityRules
* Microsoft.Network/privateDnsZones
* Microsoft.Network/privateDnsZones/virtualNetworkLinks
* Microsoft.Network/privateEndpoints
* Microsoft.Network/privateEndpoints/privateDnsZoneGroups
* Microsoft.Network/publicIPAddresses
* Microsoft.Network/virtualNetworks
* Microsoft.Network/virtualNetworks/subnets
* Microsoft.Resources/deployments
* Microsoft.Storage/storageAccounts
* Microsoft.Storage/storageAccounts/blobServices/containers
* Microsoft.Synapse/privateLinkHubs
* Microsoft.Synapse/workspaces



# Considerations
These considerations implement the pillars of the Azure Well-Architected Framework, which is a set of guiding tenets that can be used to improve the quality of a workload. For more information, see Microsoft Azure Well-Architected Framework.

## Reliability
Reliability ensures your application can meet the commitments you make to your customers. For more information, see Overview of the reliability pillar.

Event Hubs offers 90-day data retention on the premium and dedicated tiers. For failover scenarios, you can set up a secondary namespace in the paired region and activate it during failover.

Azure Synapse Spark pool jobs are recycled every seven days as nodes are taken down for maintenance. Consider this activity as you work through the service level agreements (SLAs) tied to the system. This limitation isn't an issue for many scenarios where recovery time objective (RTO) is around 15 minutes.

## Cost optimization
Cost optimization is about looking at ways to reduce unnecessary expenses and improve operational efficiencies. For more information, see [Overview of the cost optimization pillar](https://learn.microsoft.com/en-us/azure/architecture/framework/cost/overview).

### Azure Synapse
Azure Synapse Analytics serverless architecture allows you to scale your compute and storage levels independently.

Compute resources are charged based on usage. You can pause dedicated SQL pools when you're not using them in your development or test environments. You can schedule a script to pause the pool as needed, or you can pause the pool manually through the portal.

Storage resources are billed per terabyte, so your costs will increase as you ingest more data. Consider object lifecycle management through tiers on Azure Data Lake Storage. As data ages, you can move data from a hot tier, where you need to access recent data for analytics, to a cold storage tier that is priced much lower. The cold storage tier is a cost-effective option for long-term retention.


### Azure Pipelines
Pricing details for pipelines in Azure Synapse can be found under the Data Integration tab on the [Azure Synapse pricing page](https://azure.microsoft.com/pricing/details/synapse-analytics). There are three main components that influence the price of a pipeline:

1. Data pipeline activities and integration runtime hours
2. Data flows cluster size and execution
3. Operation charges

The price varies depending on the components or activities, frequency, and number of integration runtime units.

For the sample dataset, the standard Azure-hosted integration runtime, copy data activity for the core of the pipeline, is triggered on a daily schedule for all of the entities (tables) in the source database. The scenario contains no data flows. There are no operational costs for Synapse Pipelines when there are fewer than 1 million pipeline operations a month.

### Azure Synapse dedicated pool and storage
Pricing details for Azure Synapse dedicated pool can be found under the Data Warehousing tab on the [Azure Synapse pricing page](https://azure.microsoft.com/pricing/details/synapse-analytics). Under the Dedicated consumption model, customers are billed per DWU units provisioned, per hour of uptime. Another contributing factor is data storage costs: size of your data at rest + snapshots + geo-redundancy, if any.

For the sample dataset, you can provision 500DWU, which guarantees a good experience for analytical load. You can keep compute up and running over business hours of reporting. If taken into production, reserved data warehouse capacity is an attractive option for cost management. Different techniques should be used to maximize cost/performance metrics, which are covered in the previous sections.

### Blob storage
Consider using the Azure Storage reserved capacity feature to lower storage costs. With this model, you get a discount if you reserve fixed storage capacity for one or three years. For more information, see [Optimize costs for Blob storage with reserved capacity](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-reserved-capacity).

## Operational excellence
Operational excellence covers the operations processes that deploy an application and keep it running in production. For more information, see [Overview of the operational excellence pillar](https://learn.microsoft.com/en-us/azure/architecture/framework/devops/overview).

### Performance efficiency
Performance efficiency is the ability of your workload to scale to meet the demands placed on it by users in an efficient manner. For more information, see [Performance efficiency pillar overview](https://learn.microsoft.com/en-us/azure/architecture/framework/scalability/overview).

This section provides details on sizing decisions to accommodate this dataset.

You can set up Azure Synapse Spark pools with small, medium, or large virtual machine (VM) SKUs, based on the workload. You can also configure autoscale on Azure Synapse Spark pools to account for spiky workloads. If you need more compute resources, the clusters automatically scale up to meet the demand, and scale down after processing is complete.

Use best practices for designing tables in the dedicated SQL pool. Associated performance and scalability limits apply, based on the tier that the SQL pool is running on.
#### Azure Synapse provisioned pool
There's a range of [data warehouse configurations](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-manage-compute-overview) to choose from.


To see the performance benefits of scaling out, especially for larger data warehouse units, use at least a 1-TB dataset. To find the best number of data warehouse units for your dedicated SQL pool, try scaling up and down. Run a few queries with different numbers of data warehouse units after loading your data. Since scaling is quick, you can try various performance levels in an hour or less.

#### Find the best number of data warehouse units
For a dedicated SQL pool in development, begin by selecting a smaller number of data warehouse units. A good starting point is DW400c or DW200c. Monitor your application performance, observing the number of data warehouse units selected compared to the performance you observe. Assume a linear scale, and determine how much you need to increase or decrease the data warehouse units. Continue making adjustments until you reach an optimum performance level for your business requirements.

#### Scaling Synapse SQL pool
* [Scale compute for Synapse SQL pool with the Azure portal](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/quickstart-scale-compute-portal)
* [Scale compute for dedicated SQL pool with Azure PowerShell](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/quickstart-scale-compute-powershell)
* [Scale compute for dedicated SQL pool in Azure Synapse Analytics using T-SQL](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/quickstart-scale-compute-tsql)
* [Pausing, monitoring, and automation](https://learn.microsoft.com/en-us/azure/synapse-analytics/sql-data-warehouse/sql-data-warehouse-manage-compute-overview)

#### Azure Pipelines
For scalability and performance optimization features of pipelines in Azure Synapse and the copy activity used, refer to the [Copy activity performance and scalability guide](https://learn.microsoft.com/en-us/azure/data-factory/copy-activity-performance).

## Contributors
This article is maintained by Crayon. It was originally written by the following contributors.

#### Principal author:

Zach Nichter | Sr. Solution Architect

#### Other contributors:

Jon Roulston | Technical Solutions Sales Executive
Jason Wilson | Technical Data Solutions Sales Executive
