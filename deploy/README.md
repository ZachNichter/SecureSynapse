#

## Crayon's DPi30 Secure Synapse Deployment Structure

This project consists of a driver ARM template that is linked to a number of scoped ARM templates as listed:

```
* 010-Vnet: creates VNET necessary for deployment

* 020-JumpVM: creates a jumpbox VM (optional)

* 030-Bastion: creates a Bastion service for public developer access to Synapse Studio through the VM

* 040-SynapseWS: creates a Synapse workspace with managed Vnet. No provisioned reources (SQL) are created. The workspace can be locked down on the network by setting "allowAllConnections: false"

* 050-KeyVault: creates a Key Vault

* 060-PrivateLinkHub: creates a Synapse Private Link Hub to secure access to Synapse Studio

* 070-PrivateEndpoints: creates private endpoints for Synapse Studio (web), dev, sql, and sqlOnDemand access points
```

The resulting deployment looks like this:

![Deployed Architecture](images/deployedArchitecture.png?raw=true "Architecture")

## Prerequisites
### Existing Resource Group
**You must have an existing resource group** which will be used for all deployments. Note that Synapse creates a managed resource group as part of the deployment process. The managed resource group is not alway cleaned up if you delete your own resource group.

### Portal Resource Permissions
#### List of resource providers used in this deployment
Microsoft.Authorization/roleAssignments
Microsoft.Compute/virtualMachines
Microsoft.Compute/virtualMachines/extensions
Microsoft.DevTestLab/schedules
Microsoft.KeyVault/vaults
Microsoft.KeyVault/vaults/privateEndpointConnections
Microsoft.Network/bastionHosts
Microsoft.Network/networkInterfaces
Microsoft.Network/networkSecurityGroups
Microsoft.Network/networkSecurityGroups/securityRules
Microsoft.Network/privateDnsZones
Microsoft.Network/privateDnsZones/virtualNetworkLinks
Microsoft.Network/privateEndpoints
Microsoft.Network/privateEndpoints/privateDnsZoneGroups
Microsoft.Network/publicIPAddresses
Microsoft.Network/virtualNetworks
Microsoft.Network/virtualNetworks/subnets
Microsoft.Resources/deployments
Microsoft.Storage/storageAccounts
Microsoft.Storage/storageAccounts/blobServices/containers
Microsoft.Synapse/privateLinkHubs
Microsoft.Synapse/workspaces

## Secure Synapse Deployment


### Virtual Network
First thing created is the virtual network to put a fence around the Synapse environment and the services needed for this solution.

### Virtual Machine Jumpbox
After the virtual network is set up and the subnets have been created, the virtual machines can be created.

### Bastion
Once the VM's are created, Bastion is setup.  By default, this template creates an Azure Bastion deployment within this solution's resource group.  Then, the template creates the network security group settings, an AzureBastionSubnet subnet, a bastion host, and a public IP address resource that's used for the bastion host.
> **NOTE:**
When you deploy Azure Bastion, it should be deployed to its own subnet (AzureBastionSubnet) within the deployment's virtual network or to a peered virtual network.

##### Deployed Bastion Architecture
* Azure Bastion is deployed in a hub virtual network or the virtual network for this deployment that contains the jumpbox subnet and virtaul machines.
* NSGs are used to protect the subnets in the virtual network.
* The NSG protecting the VM subnet allows RDP and SSH traffic from the Azure Bastion subnet.
* Bastion supports communications only through TCP port 443 from the Azure portal.

Because Azure Bastion is protected by the virtual network's NSG, your NSG needs to support the following traffic flow:


###### Inbound:
- RDP and SSH connections from the Azure Bastion subnet to the target VM subnet
- TCP port 443 access from the internet to the Azure Bastion public IP
- TCP access from Azure Gateway Manager on ports 443 or 4443

###### Outbound:
- TCP access from the Azure platform on port 443 to support diagnostic logging

> **NOTE:**
> Azure Gateway Manager manages portal connections to the Azure Bastion service.

> **NOTE:**
> If have a Hub\Spoke network architecture, the Bastion subnet should be created in the hub vNet. The article [Bastion and Network Security Groups](https://learn.microsoft.com/en-us/azure/bastion/bastion-nsg#nsg) can provide additiona information on Bastion ingress\egress traffic rules.

![Bastion Architecture](images/bastionDiagram.png?raw=true "Architecture")

#### Components
Microsoft.Network/bastionHosts - Creates the bastion host
Microsoft.Network/virtualNetworks  - Creates a virtual network
Microsoft.Network/virtualNetworks/subnets - Creates the subnet
Microsoft Network/networkSecurityGroups - Controls the network security group settings
Microsoft.Network/publicIpAddresses - Specifies the public IP address value used for the bastion host

### Synapse Workspace

### Key Vault

### Private Link Hub

### Private Endpoints


## Post Deployment Requirements
```
1. The jumpbox VM is configured to use AAD authentication. You will need to add yourself to the "Virtual Machine User Login" or "Virtual Machine Administrator Login" roles for this VM or you will not be able to log in with you AAD credentials.
2. Make sure the Windows OS is fully updated. You may have to check for updates multiple times after rebooting if needed.
3. You will need to configure the jumpbox VM with Microsoft endpoint protection. On the VM, go to "Settings/Account/Access Work or School". Disconnect from Microsoft Azure AD, reboot, and re-login as the local administrator account. Go back to "Settings/Account/Access Work or School" and add a new connection to Microsoft Azure AD with your Microsoft account. This will register the VM with Microsoft endpoint protection.
4. (Optional) It would be best practice to enable just in time access through the Azure Portal for this VM. You may be prompted for VPN and disk encryption which are optional.
5. Manually create a private endpoint for the default Synapse data lake account in the "privateEndpointSubnet". This will allow Synapse Studio to see files in the storage account. I will fix this in the near future.
6. Manually create a *managed* private endpoint for the default Synapse data lake account. This will allow Synapse background processes to see files in the storage account. I will fix this in the near future.
```

## Notes and Bugs
The main ARM template (azuredeploy.json) references several linked templates that are stored in a read only Azure blob account. If you wish to modify these linked templates then you will need to change the main ARM template to point to your local version.

The Azure storage account does not get assigned MSI privileges for the Synapse workspace. For now you have to assign this manually to your Synapse workspace in the pulldown. See the below screenshot of the storage account network configuration:

![Storage ACL Fix](images/storageACLSBug.png?raw=true "Storage ACLS")

