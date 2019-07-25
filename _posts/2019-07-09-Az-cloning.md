---
layout: post
title: Cloning an Environment
categories:
- Azure
---
>Into the hole again, we hurried along our way, into a once-glorious garden now steeped in dark decay. 
-Alice

# Introduction

I've recently come across an interesting issue. When creating a staging environment, or one for testing by clients, the strategy was to go through a lengthy procedure of firstly using ARM templates to deploy the base infrastructure. This was followed by the installation and configuration of a SQL server which itself is a somewhat lengthy process. Then the recovery of SQL database backups (based on which databases were required at the time) and finally the deployment of the current version of the application. This entire process could take up to a week to accomplish based on the size and number of databases required for testing.

The challenge was that the client required a copy of a production or QA environment, but there was no easy way to do this via the Azure portal. Azure only offered an export solution for ARM templates, this would only recreate the base infrastructure and not the data and configurations on the disks attached to the Virtual machines.

It was difficult to find anything that would care to this specific task. Although I thought that something like this would be a common problem with a simple solution. My search may not have been thorough enough and looking back on it it is quite possible that a simple solution does exists out there. However, I had decided to create my own catered solution for this challenge.

My idea was simple, I would...

* take snapshots of all the disks in a resource group,
* copy or recreate other resources, such as network security groups or virtual networks, into a new resource group, 
* create new disks from the snapshots,
* build new virtual machines around those disks,
* and finally attach the virtual machines to the other resources.

Breaking the task down into smaller steps allowed me to create simple scripts for each scenario before combining them to achieve the end goal.

# Snapshots
The goal here was to query the resource group for a list of all the disks contained within it, loop through the list, and take snapshots of every disk.

This was a simple task that required less than 10 lines of code.


Lets first declare some variables.

```
$ResourceGroup = {Resource Group Name}
$DiskName = * # We can use wildcards here or be specific.
$Location = {Preferred data center location}  
```

We now need to select the subscription we are going to use. 

```
Select-AzSubscription -Subscription {SubscriptionId}
```
Next we query Azure for a list of disks in the target resource group.

```
$Disks = Get-AzResource -ResourceGroupName $ResourceGroup -ResourceType Microsoft.Compute/disks -Name $DiskName
```
Now that we have a list of disks we loop through the list and simply take snapshots of each disk.

```
ForEach($Disk in $Disks){
    $Name = 'ss' + $Disk.Name
    New-AzSnapshot -ResourceGroupName $ResourceGroup -SnapshotName $Nam#e -Snapshot (New-AzSnapshotConfig -SourceResourceId $Disk.ResourceID -Location $Location -CreateOption copy)
}
```
Once run, this Powershell script will create snapshots of every disk in the resource group.

### Note:
If you are using striped disks, for example for SQL database storage, you will need to bring down the host VM before taking snapshots of the disks. Since snapshots of disks are taken asynchronously, when the stripped disks are put back together there will be problems syncing the disks, as data may have been changed on one disk but not the other. This will lead to corruption of entire databases.

Additionally, it is best to bring down SQL servers before taking snapshots as a difference between database disks and log drives can cause some issues when starting SQL back up.

# Other Resources

The next step was to copy or recreate the other necessary resources into a new resource group. This step was a little time consuming as some resources depend on parameters from the virtual machines they are attached to and so I was required to create chain queries to gather the necessary parameter values.

As before we need to declare some base variables so that we do not need rewrite the same thing multiple times, and in doing so we also decrease the likelihood errors due to spelling.

```
$ResourceGroupName = {Original resource group name}
$NewResourceGroup = {Name of the new resource group}
$SubnetPrefix = {Subnet prefix for the resource group}
$StorageType = {Storage type for the managed disks, premium or standard}
$VnetNameNew = {Name of the new virtual network}
$Location = {Preferred data center location}
$DiskName = * # We can use wildcards here or be specific.
```
Next we once again select our subscriptions.
```
Select-AzSubscription -Subscription {SubscriptionId}
```
Create the new resource group.
```
New-AzResourceGroup -Name $NewResourceGroup -Location $Location
```

# NSG Rules
Now that we have the new resource group we need to copy the network security groups over to it. We do this by querying the existing groups for their configurations and creating new groups in the new resource group with the same configurations.

We first get a list of all network security groups in a resource group.
```
$NSGroups = Get-AzNetworkSecurityGroup -ResourceGroupName $ResourceGroupName
```
We now loop through these each of these groups, grab the required parameters for each rule, and create new network security groups in the new resource group with the same rules.

```
ForEach($Group in $NSGroups){

    $NSGOrigin = $Group.Name
    $NSGDest =$Group.Name

    $NSG = Get-AzNetworkSecurityGroup -Name $NSGOrigin -ResourceGroup $ResourceGroupName
    $NSGRules = Get-AzNetworkSecurityRuleConfig -NetworkSecurityGroup $NSG
    $NewNSG = New-AzNetworkSecurityGroup -Name $NSGDest -ResourceGroupName $NewResourceGroup -Location $Location

    ForEach($NSGRule in $NSGRules){
        Add-AzNetworkSecurityRuleConfig -NetworkSecurityGroup $NewNSG `
        -Name $NSGRule.Name `
        -Protocol $NSGRule.Protocol `
        -SourcePortRange $NSGRule.SourcePortRange `
        -DestinationPortRange $NSGRule.DestinationPortRange `
        -SourceAddressPrefix $NSGRule.SourceAddressPrefix `
        -DestinationAddressPrefix $NSGRule.DestinationAddressPrefix `
        -Priority $NSGRule.Priority `
        -Direction $NSGRule.Direction `
        -Access $NSGRule.Access | Out-Null

    }
    Set-AzNetworkSecurityGroup -NetworkSecurityGroup $NewNSG | Our-Null
}
```

# Virtual Networks
Next, we create virtual networks for our environment. The environment that I was working with had four types of machines; Web Server, Compute Engine, SQL Server, and a Reporting Server. Each type of machine was had its own subnet and security configurations. 

Since the subnets themselves did not have any specific settings we simple need to create one virtual network and four unique subnets.

```
$VirtualNetwork = New-AzVirtualNetwork -ResourceGroupName $NewResourceGroup `
-Location $Location `
-Name $VNetNameNew `
-AddressPrefix 10.10.0.0/16

$SubnetConfig = Add-AzVirtualNetworkSubnetConfig -Name ($SubnetPrefix + 'Web') `
-AddressPrefix 10.10.1.0/24
-VirtualNetwork $VirtualNetwork

$VirtualNetwork | Set-AzVirtualNetwork | Out-Null

$VirtualNetwork = Get-AzVirtualNetwork -ResourceGroupName $NewResourceGroup `
-Name $VNetNameNew

$SubnetConfig = Add-AzVirtualNetworkSubnetConfig -Name ($SubnetPrefix + 'Compute') `
-AddressPrefix 10.10.2.0/24
-VirtualNetwork $VirtualNetwork

$VirtualNetwork | Set-AzVirtualNetwork | Out-Null

$SubnetConfig = Add-AzVirtualNetworkSubnetConfig -Name ($SubnetPrefix + 'SQL') `
-AddressPrefix 10.10.3.0/24
-VirtualNetwork $VirtualNetwork

$VirtualNetwork | Set-AzVirtualNetwork | Out-Null

$SubnetConfig = Add-AzVirtualNetworkSubnetConfig -Name ($SubnetPrefix + 'Report') `
-AddressPrefix 10.10.4.0/24
-VirtualNetwork $VirtualNetwork

$VirtualNetwork | Set-AzVirtualNetwork | Out-Null

```

The VM type could have been put into an array and the entire process could have been done in a loop to make the code simpler. I chose to write it this way to start with to better visualize what was happening, and in the end did not end up refactoring it, leaving it as is.

# Disks and VMs
Now comes the part where we create disks from our snapshots, build VMs around those disks, and finally wire up the new VMs to the networks. The concept here is fairly straight forward:
- Get the snapshot information,
- Get the disk and VM information from source disk.
- Create new VM based on the gathered information.
- Create and attach any data disks belonging to the new VM.

Firstly we need to grab the new network security group and prepare some variable so that we can assign the correct security group to the correct virtual machine. 

```
$NSGroups = Get-AzNetworkSecurityGroup -ResourceGroupName $NewResourceGroup

ForEach($NSGroup in $NSGroups) {
    Switch -Wildcard ($NSGroup.Name){
        '*Web*' {$NSWeb = $NSGroup}
        '*Compute*' {$NSCompute = $NSGroup}
        '*SQL*' {$NSSQL = $NSGroup}
        '*Report*' {$NSReport =$NSGroup}
        Default {"Couldn't find a match for " + $NSGroup.Name}
    }
}
```
Now, we grab the snapshots we created.
```
$Snaps = Get-AzSnapshot -ResourceGroupName $ResourceGroupName -Name $DiskName
```
As we loop through them we firstly look for OS disks such that we can build virtual machines around them.
```
ForEach($Disk in $Snaps){
    If($Disk.Name -Like '*OSDisk*'){
        $SourceDiskName = ($Disk.CreationData.SourceResourceId -Split '[/]')[8]
        $SourceDisk = Get-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $SourceDiskName
        $VMName = ((Get-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $SourceDiskName).ManagedBy -Split '[/]')[8]
        $VMSize = (Get-AzVM -ResourceGroupName $ResourceGroupName -Name $VMName).HardwareProfile.VMSize
        $NICName = ((Get-AzVM -ResourceGroupName $ResourceGroupName -Name $VMName).NetworkProfile.NetworkInterfaces.Id -Split '[/]')[8]
        $IPName =((Get-AzNetworkInterface -ResourceGroupName $ResourceGroupName -Name $NICName).IPConfigurations.PublicIPAddress.Id -Split '[/]')[8]
        $VNetName = ((Get-AzNetworkInterface -ResourceGroupName $ResourceGroupName -Name $NICName).IPConfigurations.Subnet.Id -Split '[/]')[8]

        $DiskConf = New-AzDiskConfig -AccountType $SourceDisk.Sku.Name -Location $Location -CreateOption Copy -SourceResourceId $disk.Id
        $NewDisk = New-AzDisk -Disk $DiskConf -ResourceGroupName $NewResourceGroup -DiskName $SourceDiskName
        
        $VirtualMachine = New-AzVMConfig -VMName $VMName -VMSize $VMSize | Set-AzVMBootDiagnostic -Disable
        $VirtualMachine = Set-AZVMOSDisk -VM $VirtualMachine -ManagedDiskId $NewDisk.Id -CreateOption Attach -Windows

        $PublicIP = New-AzPublicIpAddress -Name $IPName -ResourceGroupName $NewResourceGroup -Location $Location -AllocationMethod Static
        
        $GetPublicIP = Get-AzPublicIpAddress -Name $IPName -ResourceGroupName $NewResourceGroup
        $VNet = Get-AzVirtualNetwork -Name $VNetNameNew -ResourceGroupName $NewResourceGroup

        Switch -Wildcard ($vmName){
                '*Web*' {$Nic = New-AzNetworkInterface -Name $NICName -ResourceGroupName $NewResourceGroup -Location $Location -SubnetId $VNet.Subnets[0].Id -PublicIPAddressId $GetPublicIP.Id -NetworkSecurityGroupId $NSWeb.Id}
                '*Compute*' {$Nic = New-AzNetworkInterface -Name $NICName -ResourceGroupName $newResourceGroup -Location $location -SubnetId $VNet.Subnets[1].Id -PublicIpAddressId $GetPublicIP.Id -NetworkSecurityGroupId $NSCompute.Id}
                '*SQL*' {$Nic = New-AzNetworkInterface -Name $NICName -ResourceGroupName $NewResourceGroup -Location $Location -SubnetId $VNet.Subnets[2].Id -PublicIPAddressId $GetPublicIP.Id -NetworkSecurityGroupId $NSSQL.Id}
                '*Report*' {$Nic = New-AzNetworkInterface -Name $NICName -ResourceGroupName $NewResourceGroup -Location $Location -SubnetId $VNet.Subnets[3].Id -PublicIPAddressId $GetPublicIp.Id -NetworkSecurityGroupId $NSReport.Id}
                Default { "Couldn't find a match for " + $VMName; exit}
        }    

        $VirtualMachine = Add-AzVMNetworkInterface -VM $VirtualMachine -Id $Nic.Id
        New-AzVM -VM $VirtualMachine -ResourceGroupName $NewResourceGroup -Location $Location

    } Else {
        $DataDiskCount++
    }
}

```
Next we attach the remaining data disks to the newly created virtual machines.

```
Write-Host ('Found ' + $dataDiskCount  + ' data disks') -ForegroundColor Green
    ForEach($Disk in $Snaps){
        If($Disk.name -NotLike '*osdisk*'){
            $SourceDiskName = ($Disk.CreationData.SourceResourceId -Split '[/]')[8]
            $SourceDisk = Get-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $SourceDiskName
            $VMName = ((Get-AzDisk -ResourceGroupName $ResourceGroupName -DiskName $SourceDiskName).ManagedBy -split '[/]')[8]
                    
            $DiskConf = New-AzDiskConfig -AccountType $SourceDisk.Sku.Name -Location $Location -CreateOption Copy -SourceResourceId $Disk.Id
            $NewDisk = New-AzDisk -Disk $DiskConf -ResourceGroupName $NewResourceGroup -DiskName $SourceDiskName
            
            If($VMName){  
                $VM = Get-AzVM -ResourceGroupName $ResourceGroupName -Name $VMName
                $GetStorageProfile = $VM.StorageProfile.DataDisks
                ForEach($Item in $GetStorageProfile){
                    If ($Item.Name -eq $SourceDiskName){
                        $Lun = $Item.Lun
                        Break
                    }
                }
            
                $GetNewDisk = Get-AzDisk -Name $SourceDiskName -ResourceGroupName $NewResourceGroup
                $VMNew = Get-AzVM -ResourceGroupName $NewResourceGroup -Name $VMName
                
                $VMNew = Add-AzVMDataDisk -CreateOption Attach -Lun $Lun -VM $VMNew -ManagedDiskId $GetNewDisk.Id
                Update-AzVM -VM $VMNew -ResourceGroupName $NewResourceGroup
            } Else {
                Write-Host ('Disk not attached to Vm ' +  $sourceDiskName) -ForegroundColor Red -BackgroundColor Yellow
            }
        }
    }
```
# Conclusion
Once this has completed we have an exact copy of the original resource group. This script was specifically catered for a client, only the resources that mattered for the environment where cloned.

We can make this better by using workflows to parallelize the process of creating VMs and attaching data disks. I'll continue this once I make the necessary changes to my local scripts.

For extra resources these scripts can be modified to also clone items such as firewalls, storage accounts etc...