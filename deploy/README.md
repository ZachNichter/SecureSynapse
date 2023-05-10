# SecureSynapseARM Structure

This project consists of a driver ARM template that is linked to a number of scoped ARM templates as listed:
```
* 01-Vnet: creates VNET necessary for deployment

* 02-JumpVM: creates a jumpbox VM (optional)

* 03-SynapseWS: creates a Synapse workspace with managed Vnet. No provisioned reources (SQL) are created. The workspace can be locked down on the network by setting "allowAllConnections: false"

* 04-PrivateLinkHub: creates a Synapse Private Link Hub to secure access to Synapse Studio

* 05-PrivateEndpoints: creates private endpoints for Synapse Studio (web), dev, sql, and sqlOnDemand access points
```

The resulting deployment looks like this:

![Deployed Architecture](images/deployedArchitecture.png?raw=true "Architecture")

# Prerequisites
**You must have an existing resource group** which will be used for all deployments. Note that Synapse creates a managed resource group as part of the deployment process. The managed resource group is not alway cleaned up if you delete your own resource group.

## Secure Credentials
In order to avoid storing passwords in the templates, you must pass a password secure string as a Powershell cmdlet parameter as shown below:
```powershell
$vmpassword = ConvertTo-SecureString "your-vm-password" -AsPlainText -Force

$sqlpassword = ConvertTo-SecureString "your-Synapse-password" -AsPlainText -Force

New-AzResourceGroupDeployment
    -ResourceGroupName "your-resource-group"
    -TemplateFile azuredeploy.json
    -TemplateParameterFile azuredeploy.parameters.json
    -vmAdminUsername "your-vm-admin-user"
    -vmAdminPassword $vmpassword
    -sqlAdminLogin "your-Synapse-admin-user"
    -sqlAdminPassword $sqlpassword
```

Template parameters specified on the command line will overwrite the defaults in the template.

# Example Calls

## Powershell
Clone or download the Github repo from https://github.com/vsuopys/SecureSynapse.

On your local machine, switch directory to SecureSynapseARM.

Run the following commands:

```powershell
$vmpassword = ConvertTo-SecureString "your-vm-password" -AsPlainText -Force

$sqlpassword = ConvertTo-SecureString "your-Synapse-password" -AsPlainText -Force

Connect-AzAccount

New-AzResourceGroupDeployment -ResourceGroupName "your-resource-group" -Name "your-deployment-name" -TemplateFile azuredeploy.json -TemplateParameterFile azuredeploy.parameters.json -vmAdminUsername "your-vm-admin-user" -vmAdminPassword $vmpassword -sqlAdminLogin "your-Synapse-admin-user" -sqlAdminPassword $sqlpassword -allowAllConnections "false" -createJumpVM "true"
```

# Post Deployment Requirements
```
1. The jumpbox VM is configured to use AAD authentication. You will need to add yourself to the "Virtual Machine User Login" or "Virtual Machine Administrator Login" roles for this VM or you will not be able to log in with you AAD credentials.
2. Make sure the Windows OS is fully updated. You may have to check for updates multiple times after rebooting if needed.
3. You will need to configure the jumpbox VM with Microsoft endpoint protection. On the VM, go to "Settings/Account/Access Work or School". Disconnect from Microsoft Azure AD, reboot, and re-login as the local administrator account. Go back to "Settings/Account/Access Work or School" and add a new connection to Microsoft Azure AD with your Microsoft account. This will register the VM with Microsoft endpoint protection.
4. (Optional) It would be best practice to enable just in time access through the Azure Portal for this VM. You may be prompted for VPN and disk encryption which are optional.

5. Manually create a private endpoint for the default Synapse data lake account in the "privateEndpointSubnet". This will allow Synapse Studio to see files in the storage account. I will fix this in the near future.
6. Manually create a *managed* private endpoint for the default Synapse data lake account. This will allow Synapse background processes to see files in the storage account. I will fix this in the near future.

```

# Notes and Bugs
The main ARM template (azuredeploy.json) references several linked templates that are stored in a read only Azure blob account. If you wish to modify these linked templates then you will need to change the main ARM template to point to your local version.

The Azure storage account does not get assigned MSI privileges for the Synapse workspace. For now you have to assign this manually to your Synapse workspace in the pulldown. See the below screenshot of the storage account network configuration:

![Storage ACL Fix](images/storageACLSBug.png?raw=true "Storage ACLS")

# Parameter Specifications
These values can be overridden on the command line.

```json
{
    "virtualNetwork_vnet_name": {
      "defaultValue": "vnet-customer",
      "type": "String"
    },
    "virtualNetwork_address_block": {
      "defaultValue": "172.17.0.0/16",
      "type": "string"
    },
    "virtualNetwork_appsubnet_address_block": {
      "defaultValue": "172.17.3.0/26",
      "type": "string"
    },
    "virtualNetwork_datasubnet_address_block": {
      "defaultValue": "172.17.2.0/26",
      "type": "string"
    },
    "virtualNetwork_gateway_address_block": {
      "defaultValue": "172.17.1.0/26",
      "type": "string"
    },
    "virtualNetwork_default_address_block": {
      "defaultValue": "172.17.0.0/26",
      "type": "string"
    },
    "createJumpVM": {
      "type": "String",
      "defaultValue": "false",
      "metadata": {
        "description": "Create a jumpbox VM?"
      }
    },
    "virtualMachineName": {
        "type": "String",
        "maxLength": 12,
        "defaultValue": "vm",    
        "metadata": {
          "description": "The name of the VM. Limited to 2 characters because a unique string is appendedin the template."
        }
    },
    "vmAdminUsername": {
      "type": "String",
      "defaultValue": "cloudsa",
      "metadata": {
        "description": "The admin user name of the VM"
      }
    },
    "vmAdminPassword": {
      "type": "SecureString",
      "minLength": 12,
      "defaultValue": null,
      "metadata": {
        "description": "Pass a value in on the command line. The admin password of the VM, 12 characters minimum."
      }
    },
    "createSynapseWS": {
      "type": "String",
      "defaultValue": "true",
      "metadata": {
        "description": "Create new Synapse workspace."
      }
    },
    "createNewStorageAccount": {
      "type": "String",
      "defaultValue": "true",
      "metadata": {
        "description": "Create new data lake storage account for Synapse metadata."
      }
    },
    "dataLakeAccount": {
      "type": "string",
      "defaultValue": "secsyndlac",
      "metadata": {
        "description": "Name of the data lake storage account for Synapse metadata."
      }
    },
    "dataLakeAccountContainer": {
      "type": "string",
      "defaultValue": "synapsefilesystem",
      "metadata": {
        "description": "Name of the data lake filesystem for Synapse metadata."
      }
    },
    "synapseWSName": {
      "type": "string",
      "defaultValue": "secsynapse-ws",
      "metadata": {
        "description": "Name of the Synapse workspace."
      }
    },
    "allowAllConnections": {
      "type": "String",
      "defaultValue": "true",
      "metadata": {
        "description": "Name of the data lake storage account for Synapse metadata."
      }
    },
    "sqlAdminLogin": {
      "type": "string",
      "defaultValue": "sqladminuser",
      "metadata": {
          "description": "Synapse SQL administrator account name"
      }
    },
    "sqlAdminPassword": {
      "type": "securestring",
      "defaultValue": null,
      "metadata": {
          "description": "Pass a value in on the command line."
      }
    }
}
```