---
title: 'Azure Active Directory Domain Services: Troubleshooting Network Security Group configuration | Microsoft Docs'
description: Troubleshooting NSG configuration for Azure AD Domain Services
services: active-directory-ds
documentationcenter: ''
author: eringreenlee
manager:
editor:

ms.assetid: 95f970a7-5867-4108-a87e-471fa0910b8c
ms.service: active-directory-ds
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: article
ms.date: 02/12/2018
ms.author: ergreenl

---
# Troubleshoot invalid networking configuration for your managed domain
This article helps you troubleshoot and resolve network-related configuration errors that result in the following alert message:

## Alert AADDS104: Network error
**Alert message:**
 *Microsoft is unable to reach the domain controllers for this managed domain. This may happen if a network security group (NSG) configured on your virtual network blocks access to the managed domain. Another possible reason is if there is a user-defined route that blocks incoming traffic from the internet.*

Invalid NSG configurations are the most common cause of network errors for Azure AD Domain Services. The Network Security Group (NSG) configured for your virtual network must allow access to [specific ports](active-directory-ds-networking.md#ports-required-for-azure-ad-domain-services). If these ports are blocked, Microsoft cannot monitor or update your managed domain. Additionally, synchronization between your Azure AD directory and your managed domain is impacted. While creating your NSG, keep these ports open to avoid interruption in service.


## Sample NSG
The following table depicts a sample NSG that would keep your managed domain secure while allowing Microsoft to monitor, manage, and update information.

![sample NSG](.\media\active-directory-domain-services-alerts\default-nsg.png)

>[!NOTE]
> Azure AD Domain Services requires unrestricted outbound access from the virtual network. We recommend not to create any additional NSG rule that restricts outbound access for the virtual network.

## Add a rule to a Network Security Group using the Azure portal
If you do not want to use PowerShell, you can manually add single rules to NSGs using the Azure portal. To create rules in your Network security group, complete the following steps:

1. Navigate to the [Network security groups](https://portal.azure.com/#blade/HubsExtension/Resources/resourceType/Microsoft.Network%2FNetworkSecurityGroups) page in the Azure portal
2. From the table, choose the NSG associated with the subnet in which your managed domain is enabled.
3. Under **Settings** in the left-hand panel, click either **Inbound security rules** or **Outbound security rules**.
4. Create the rule by clicking **Add** and filling in the information. Click **OK**.
5. Verify your rule has been created by locating it in the rules table.


## Create an NSG for Azure AD Domain Services using PowerShell
This NSG is configured to allow inbound traffic to the ports required by Azure AD Domain Services, while denying any other unwanted inbound access.

**Pre-requisite: Install and configure Azure PowerShell**
Follow the instructions to [install the Azure PowerShell module and connect to your Azure subscription](https://docs.microsoft.com/powershell/azure/install-azurerm-ps?toc=%2fazure%2factive-directory-domain-services%2ftoc.json).

>[!TIP]
> We recommend using the latest version of the Azure PowerShell module. If you already have an older version of the Azure PowerShell module installed, update to the latest version.
>

Use the following steps to create a new NSG using PowerShell. 
1. Log in to your Azure subscription.

  ```PowerShell
  # Log in to your Azure subscription.
  Login-AzureRmAccount
  ```

2. Create an NSG with three rules. The following script defines three rules for the NSG that allow access to the ports needed to run Azure AD Domain Services. Then, the script creates a new NSG that contains those rules. Use the same format to add additional rules that allow other inbound traffic, if required by workloads deployed in the virtual network.

  ```PowerShell
  # Allow inbound HTTPS traffic to enable synchronization to your managed domain.
  $SyncRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowSyncWithAzureAD `
  -Description "Allow synchronization with Azure AD" `
  -Access Allow -Protocol Tcp -Direction Inbound -Priority 101 `
  -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * `
  -DestinationPortRange 443

  # Allow management of your domain over port 5986 (PowerShell Remoting)
  $PSRemotingRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowPSRemoting `
  -Description "Allow management of domain through port 5986" `
  -Access Allow -Protocol Tcp -Direction Inbound -Priority 102 `
  -SourceAddressPrefix 52.180.183.8, 23.101.0.70, 52.225.184.198, 52.179.126.223, `
  13.74.249.156, 52.187.117.83, 52.161.13.95, 104.40.156.18, 104.40.87.209, `
  52.180.179.108, 52.175.18.134, 52.138.68.41, 104.41.159.212, 52.169.218.0, `
  52.187.120.237, 52.161.110.169, 52.174.189.149, 13.64.151.161 `
  -SourcePortRange * -DestinationAddressPrefix * `
  -DestinationPortRange 5986

  # Allow management of your domain over port 3389 (remote desktop).
  $RemoteDesktopRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowRD `
  -Description "Allow management of domain through port 3389" `
  -Access Allow -Protocol Tcp -Direction Inbound -Priority 103 `
  -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * `
  -DestinationPortRange 3389

  # Create the NSG with the 3 rules above
  $Nsg = New-AzureRmNetworkSecurityGroup -ResourceGroupName $ResourceGroup -Location $Location `
  -Name "AAD-DomainServices-NSG" -SecurityRules $SyncRule,$PSRemotingRule,$RemoteDesktopRule
  ```

3. Lastly, associate the NSG with the vnet and subnet of choice.

  ```PowerShell
  # Find vnet and subnet
  $Vnet = Get-AzureRmVirtualNetwork -ResourceGroupName $ResourceGroup -Name $VnetName
  $Subnet = Get-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $Vnet -Name $SubnetName

  # Set the nsg to the subnet and save the changes
  $Subnet.NetworkSecurityGroup = $Nsg
  Set-AzureRmVirtualNetwork -VirtualNetwork $Vnet
  ```

## Full script to create and apply an NSG for Azure AD Domain Services
>[!TIP]
> We recommend using the latest version of the Azure PowerShell module. If you already have an older version of the Azure PowerShell module installed, update to the latest version.
>

```PowerShell
# Change the following values to match your deployment
$ResourceGroup = "ResourceGroupName"
$Location = "westus"
$VnetName = "exampleVnet"
$SubnetName = "exampleSubnet"

# Log in to your Azure subscription.
Login-AzureRmAccount

# Allow inbound HTTPS traffic to enable synchronization to your managed domain.
$SyncRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowSyncWithAzureAD `
-Description "Allow synchronization with Azure AD" `
-Access Allow -Protocol Tcp -Direction Inbound -Priority 101 `
-SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * `
-DestinationPortRange 443

# Allow management of your domain over port 5986 (PowerShell Remoting)
$PSRemotingRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowPSRemoting `
-Description "Allow management of domain through port 5986" `
-Access Allow -Protocol Tcp -Direction Inbound -Priority 102 `
-SourceAddressPrefix 52.180.183.8, 23.101.0.70, 52.225.184.198, 52.179.126.223, `
13.74.249.156, 52.187.117.83, 52.161.13.95, 104.40.156.18, 104.40.87.209, `
52.180.179.108, 52.175.18.134, 52.138.68.41, 104.41.159.212, 52.169.218.0, `
52.187.120.237, 52.161.110.169, 52.174.189.149, 13.64.151.161 `
-SourcePortRange * -DestinationAddressPrefix * `
-DestinationPortRange 5986

# Allow management of your domain over port 3389 (remote desktop).
$RemoteDesktopRule = New-AzureRmNetworkSecurityRuleConfig -Name AllowRD `
-Description "Allow management of domain through port 3389" `
-Access Allow -Protocol Tcp -Direction Inbound -Priority 103 `
-SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * `
-DestinationPortRange 3389

# Create the NSG with the 3 rules above
$Nsg = New-AzureRmNetworkSecurityGroup -ResourceGroupName $ResourceGroup -Location $Location `
-Name "AAD-DomainServices-NSG" -SecurityRules $SyncRule,$PSRemotingRule,$RemoteDesktopRule

# Find vnet and subnet
$Vnet = Get-AzureRmVirtualNetwork -ResourceGroupName $ResourceGroup -Name $VnetName
$Subnet = Get-AzureRmVirtualNetworkSubnetConfig -VirtualNetwork $Vnet -Name $SubnetName

# Set the nsg to the subnet and save the changes
$Subnet.NetworkSecurityGroup = $Nsg
Set-AzureRmVirtualNetwork -VirtualNetwork $Vnet
```

> [!NOTE]
> This default NSG does not lock down access to the port used for Secure LDAP. To lock down Secure LDAP access over the internet, see [this article](active-directory-ds-troubleshoot-ldaps.md).
>

## Need help?
Contact the Azure Active Directory Domain Services product team to [share feedback or for support](active-directory-ds-contact-us.md).
