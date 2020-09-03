Converting the steps from [this guide](https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-manually-configure-prerequisites-tutorial) to powershell.
```
#############################################################################################################################################################################################
#Convert this articel to powershell https://docs.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/availability-group-manually-configure-prerequisites-tutorial
#############################################################################################################################################################################################
#Step 1 - Resouce Group
$Location = "AustraliaEast"
$ResourceGroupName = "sql-ha-rg"

New-AzResourceGroup -Name $ResourceGroupName -Location $Location

#Step 2 - Network and Subnet
# 2 Subnets are required
# Admin for Domain Controllrt
# SQL for SQL Servers
$VNetName = "autoHAVNET"
$VNetAddressPrefix = "10.0.0.0/16"
$VNet = New-AzVirtualNetwork -Name $VNetName -ResourceGroupName $ResourceGroupName -Location $Location -AddressPrefix $VNetAddressPrefix

## Subnet 1
$SubnetName_1 = "admin"
$VNetSubnetAddressPrefix_1 = "10.0.0.0/28"
$SubnetConfig_1 = Add-AzVirtualNetworkSubnetConfig -Name $SubnetName_1 -AddressPrefix $VNetSubnetAddressPrefix_1 -VirtualNetwork $VNet | Set-AzVirtualNetwork

## Subnet 2
$SubnetName_2 = "sqlsubnet"
$VNetSubnetAddressPrefix_2 = "10.0.1.0/26"
$SubnetConfig_2 = Add-AzVirtualNetworkSubnetConfig -Name $SubnetName_2 -AddressPrefix $VNetSubnetAddressPrefix_2 -VirtualNetwork $VNet | Set-AzVirtualNetwork


#Step 3- Create Availability Set
# 1 for domain controller
$AvailabilitySet_1  = "adavailabilityset"
$AvailabilitySet_1 = New-AzAvailabilitySet -ResourceGroupName $ResourceGroupName -Name $AvailabilitySet_1  -Location $Location -PlatformFaultDomainCount 3 -PlatformUpdateDomainCount 5

#2 for sql
$AvailabilitySet_2  = "sqlavailabilityset"
$AvailabilitySet_2 = New-AzAvailabilitySet -ResourceGroupName $ResourceGroupName -Name $AvailabilitySet_2  -Location $Location -PlatformFaultDomainCount 3 -PlatformUpdateDomainCount 3

#Step 4 - Create 2 Domain Controllers
$Credential = Get-Credential -Message "Type the name and password of the local administrator account."
$VMSize = "Standard_DS1_v2"
$StorageName = "sqlhargstorage1"
$StorageSku = "Standard_LRS"
$StorageAccount = New-AzStorageAccount -ResourceGroupName $ResourceGroupName -Name $StorageName -SkuName $StorageSku -Kind "Storage" -Location $Location


for ($i=1; $i -le 2; $i++)
{
    $DCName = "ad-dc-$i"
    $VirtualMachine = New-AzVMConfig -VMName $DCName -VMSize $VMSize -AvailabilitySetId $AvailabilitySet_1.Id

    $VirtualMachine = Set-AzVMOperatingSystem -VM $VirtualMachine -Windows -ComputerName $DCName -Credential $Credential -ProvisionVMAgent -EnableAutoUpdate #-TimeZone = $TimeZone
    $VNet = Get-AzVirtualNetwork -Name $VNetName
    $Interface = New-AzNetworkInterface -Name "ad-dc-$i-nic" -ResourceGroupName $ResourceGroupName -Location $Location -SubnetId $VNet.Subnets[0].Id 
    $VirtualMachine = Add-AzVMNetworkInterface -VM $VirtualMachine -Id $Interface.Id

    $OSDiskName = $DCName + "_OSDisk"
    $OSDiskUr = $StorageAccount.PrimaryEndpoints.Blob.ToString() + "vhds/" + $OSDiskName + ".vhd"
    $VirtualMachine = Set-AzVMOSDisk -VM $VirtualMachine -Name $OSDiskName -VhdUri $OSDiskUr -Caching ReadOnly -CreateOption FromImage

    #Image
        $VirtualMachine = Set-AzVMSourceImage `
        -VM $VirtualMachine `
        -PublisherName "MicrosoftWindowsServer" `
        -Offer "WindowsServer" `
        -Skus "2016-Datacenter" `
        -Version "latest"

    New-AzVM -ResourceGroupName $ResourceGroupName -Location $Location -VM $VirtualMachine
}
# Step 5 - 2 SQL Servers
for ($i=1; $i -le 2; $i++)
{
    $DCName = "sqlserver-$i"
    $VirtualMachine = New-AzVMConfig -VMName $DCName -VMSize $VMSize -AvailabilitySetId $AvailabilitySet_2.Id

    $VirtualMachine = Set-AzVMOperatingSystem -VM $VirtualMachine -Windows -ComputerName $DCName -Credential $Credential -ProvisionVMAgent -EnableAutoUpdate #-TimeZone = $TimeZone
    $VNet = Get-AzVirtualNetwork -Name $VNetName
    $Interface = New-AzNetworkInterface -Name "sqlserver-$i-nic" -ResourceGroupName $ResourceGroupName -Location $Location -SubnetId $VNet.Subnets[1].Id 
    $VirtualMachine = Add-AzVMNetworkInterface -VM $VirtualMachine -Id $Interface.Id

    $OSDiskName = $DCName + "_OSDisk"
    $OSDiskUr = $StorageAccount.PrimaryEndpoints.Blob.ToString() + "vhds/" + $OSDiskName + ".vhd"
    $VirtualMachine = Set-AzVMOSDisk -VM $VirtualMachine -Name $OSDiskName -VhdUri $OSDiskUr -Caching ReadOnly -CreateOption FromImage

    #Image
        $VirtualMachine = Set-AzVMSourceImage `
        -VM $VirtualMachine `
        -PublisherName "MicrosoftSQLServer" `
        -Offer "SQL2017-WS2016" `
        -Skus "SQLDEV" `
        -Version "latest"

    New-AzVM -ResourceGroupName $ResourceGroupName -Location $Location -VM $VirtualMachine
}
```
