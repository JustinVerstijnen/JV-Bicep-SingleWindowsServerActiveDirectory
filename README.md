# JV Bicep Single Windows Server with Active Directory deployment

This project deploys a Windows Server VM in Azure and promotes it to a new Active Directory Domain Controller.

The Active Directory installation is executed by the Azure Custom Script Extension using an inline protected PowerShell command from `main.bicep`.

## Project structure

```text
jv-bicep-single-windows-server-active-directory/
├── getting-started-with-bicep.md
├── main.bicep
└── README-Bicep-guide.txt
```

The actual Azure deployment is defined in:

```text
main.bicep
```

## What it creates

The Bicep file is deployed into an existing Azure resource group. Based on the `projectName` parameter, Bicep creates resources with this naming convention. The Azure VM resource name and Windows computer name are kept identical, including hyphens:

```text
Resource group: rg-jv-<project>   Created before deployment with Azure CLI
VM:             vm-jv-<project>
Computer name:  vm-jv-<project>
OS disk:        osdisk-jv-<project>
VNET:           vnet-jv-<project>
Subnet:         snet-jv-<project>
NIC:            nic-jv-<project>
Public IP:      pip-jv-<project>
NSG:            nsg-jv-<project>
VM extension:   install-ad-ds
```

Default AD forest name:

```text
jvlab.local
```

Default AD NetBIOS name:

```text
JVLAB
```

Default internal IP:

```text
10.10.1.4
```

Default network ranges:

```text
VNET:   10.10.0.0/16
Subnet: 10.10.1.0/24
```

Default VM size:

```text
Standard_B2ms
```

Default Windows Server image:

```text
Windows Server 2022 Datacenter Azure Edition
```

## Active Directory bootstrap

This Bicep project does not download a separate bootstrap script from GitHub.

The Active Directory Domain Services installation command is included directly in `main.bicep` and is executed by the Azure Custom Script Extension.

The extension installs the AD DS role, installs DNS, creates a new forest, sets the DSRM password, and schedules a restart of the VM.

The password is base64 encoded inside the generated PowerShell command only to avoid quoting issues. Base64 is not encryption. The command is passed through `protectedSettings` so it is not stored as normal deployment output.

## RDP source IP allow list

RDP is not opened to the entire internet by default. Configure the allowed source public IPv4 address with the `sourceIpAddress` parameter.

Enter only the IP address, without `/32`:

```powershell
sourceIpAddress="12.34.56.78"
```

The Bicep file adds `/32` automatically in the NSG rule:

```text
12.34.56.78/32
```

This template supports one RDP source IP address by default. If you want to allow multiple source IP addresses, adjust the NSG rule in `main.bicep`.

## Requirements

Install these tools on your workstation:

- Visual Studio Code
- Azure CLI
- Bicep extension for Visual Studio Code
- An Azure subscription where you are allowed to create resources

Azure CLI can install and use the Bicep CLI automatically. To check Bicep, run:

```powershell
az bicep version
```

If Bicep is not installed yet, run:

```powershell
az bicep install
```

## How to run from Visual Studio Code

1. Extract this ZIP file.
2. Open the extracted folder in Visual Studio Code.
3. Open a terminal in Visual Studio Code.
4. Sign in to Azure:

```powershell
az login
```

5. Optional: set the correct subscription:

```powershell
az account set --subscription "00000000-0000-0000-0000-000000000000"
```

6. Create the target resource group:

```powershell
az group create --name "rg-jv-biceplab" --location "westeurope"
```

7. Validate that the Bicep file can be built:

```powershell
az bicep build --file .\main.bicep
```

8. Run a what-if deployment first:

```powershell
az deployment group what-if `
  --resource-group "rg-jv-biceplab" `
  --template-file .\main.bicep `
  --parameters `
    projectName="biceplab" `
    sourceIpAddress="12.34.56.78" `
    adminUsername="jvadmin" `
    adminPassword="<strong-password>" `
    domainName="jvlab.local" `
    domainNetbiosName="JVLAB"
```

9. Apply the deployment:

```powershell
az deployment group create `
  --resource-group "rg-jv-biceplab" `
  --template-file .\main.bicep `
  --parameters `
    projectName="biceplab" `
    sourceIpAddress="12.34.56.78" `
    adminUsername="jvadmin" `
    adminPassword="<strong-password>" `
    domainName="jvlab.local" `
    domainNetbiosName="JVLAB"
```

After the deployment, the VM restarts to complete the Active Directory Domain Services installation. Wait several minutes before testing the new domain controller.

## Useful parameters

The most important parameters in `main.bicep` are:

| Parameter | Default value | Description |
| --- | --- | --- |
| `projectName` | `biceplab` | Short project name used in the Azure resource names. Keep it short because the Windows computer name has a 15 character limit. |
| `location` | Resource group location | Azure region where the resources will be deployed. |
| `adminUsername` | `jvadmin` | Local administrator username for the VM. |
| `adminPassword` | No default | Local administrator password and Active Directory DSRM password. |
| `sourceIpAddress` | No default | Public IPv4 address allowed to connect with RDP. Enter without `/32`. |
| `vmSize` | `Standard_B2ms` | Windows Server VM size. |
| `vnetAddressPrefix` | `10.10.0.0/16` | Virtual network address space. |
| `subnetPrefix` | `10.10.1.0/24` | Subnet address space. |
| `vmPrivateIpAddress` | `10.10.1.4` | Static private IP address for the domain controller. |
| `domainName` | `jvlab.local` | Active Directory domain name to create. |
| `domainNetbiosName` | `JVLAB` | Active Directory NetBIOS name to create. |

## Outputs

After deployment, Azure CLI returns these outputs from `main.bicep`:

```text
resourceGroupName
virtualMachineName
privateIPAddress
publicIPAddress
rdpCommand
activeDirectoryDomain
postDeploymentNote
```

The `rdpCommand` output can be used to start an RDP connection to the public IP address:

```powershell
mstsc /v:<public-ip-address>
```

## Logs on the deployed server

After deployment, check the Custom Script Extension status in the Azure Portal.

On the VM, Custom Script Extension logs can usually be found under:

```text
C:\WindowsAzure\Logs\Plugins\Microsoft.Compute.CustomScriptExtension\
```

The extension package files can usually be found under:

```text
C:\Packages\Plugins\Microsoft.Compute.CustomScriptExtension\
```

## Updating the lab environment

If you change the Bicep file or deployment parameters, run what-if again before applying the change:

```powershell
az deployment group what-if `
  --resource-group "rg-jv-biceplab" `
  --template-file .\main.bicep `
  --parameters `
    projectName="biceplab" `
    sourceIpAddress="12.34.56.78" `
    adminUsername="jvadmin" `
    adminPassword="<strong-password>" `
    domainName="jvlab.local" `
    domainNetbiosName="JVLAB"
```

Then apply the deployment again:

```powershell
az deployment group create `
  --resource-group "rg-jv-biceplab" `
  --template-file .\main.bicep `
  --parameters `
    projectName="biceplab" `
    sourceIpAddress="12.34.56.78" `
    adminUsername="jvadmin" `
    adminPassword="<strong-password>" `
    domainName="jvlab.local" `
    domainNetbiosName="JVLAB"
```

Azure Resource Manager uses incremental mode by default for resource group deployments. This means resources in the template are created or updated, but resources that are missing from the template are not automatically deleted from the resource group.

## Destroy the lab environment

When you are done testing, delete the complete resource group:

```powershell
az group delete --name "rg-jv-biceplab" --yes --no-wait
```

This removes the VM, OS disk, NIC, NSG, VNET, public IP address, and other resources inside the resource group.

## Security note

This project is a lab template, not a production-ready domain controller design.

RDP is exposed through a public IP address, but restricted to the configured `sourceIpAddress` parameter. For production usage, use a secure management method such as Azure Bastion, VPN, Just-in-Time VM access, or a private management network.

Do not commit real passwords, generated ARM templates, scripts with secrets, screenshots with secrets, or deployment command examples containing real passwords to Git.

For production usage, use a secure approach such as Azure Key Vault, secure pipeline variables, protected deployment processes, and a design with proper backup, monitoring, patching, recovery, and domain controller redundancy.
