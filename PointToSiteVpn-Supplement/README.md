# Point-to-Site VPN: Supplement to Microsoft Learning Course AZ-300

This Lab is a supplement to Microsoft Learning courses [AZ-300T02](https://docs.microsoft.com/en-us/learn/certifications/courses/az-300t02) (Module 3).

This Lab adds a Point-to-Site VPN to the network infrastructure build by AZ-300 Module 3's Lab [Configuring VNet peering and service chaining](https://github.com/MicrosoftLearning/AZ-300-MicrosoftAzureArchitectTechnologies/blob/master/Instructions/AZ-300T02_Lab_Mod03_Configuring%20VNet%20peering%20and%20service%20chaining.md).

## Existing Network Infrastructure

Hub-Vnet, Spoke-Vnet, Peering

Route table, Custom Route

VM1, VM2, VM3

## Supplemental Network Infrastructure

Gateway, PC

## Overview
There are five exercises in this Lab:

| | |
|-|-|
| Exercise 1 | Define variables
| Exercise 2 | Verify existing infrastructure
| Exercise 3 | Create virtual gateway
| Exersise 4 | Add VPN to PC and test
| Exersise 5 | Clean up

Most tasks are written in Azure CLI. Few tasks in exercise 3 use PowerShell cmdlets. **In principal PowerShell is required as Shell**, either CloudShell or local PowerShell. Azure CLI version 2.5.0 or above is required.

## Exercise 1: Define variables

```powershell
$HubRg="az3000401-LabRG"
$HubVnet="az3000401-vnet"

$SpokeRg="az3000402-LabRG"
$SpokeVnet="az3000402-vnet"

$Vm1="az3000401-vm1"
$Vm2="az3000401-vm2"
$Vm3="az3000402-vm1"

$GatewayName="AdatumGateway"
$GatewaySubnetPrefix="10.0.3.0/27"
$GatewayAddressPool="192.168.0.0/24"

$RootCertificateName="AdatumRootCertificate"
$ClientCertificateName="AdatumClientCertificate"
```


## Exercise 2: Verify existing infrastructure
### Task 1: Verify hub and spoke virtual networks

```powershell
az network vnet show `
    --name $HubVnet --resource-group $HubRg `
    --query '{name:name,location:location,addressSpace:addressSpace.addressPrefixes[0],provisioningState:provisioningState}' `
    --output table

$Location=az network vnet show `
    --name $HubVnet --resource-group $HubRg `
    --query "location" `
    --output tsv
    
az network vnet show `
    --name $SpokeVnet --resource-group $SpokeRg `
    --query '{name:name,location:location,addressSpace:addressSpace.addressPrefixes[0],provisioningState:provisioningState}' `
    --output table
 ```

### Task 2: Verify peering

```powershell
$HubPeeringToSpoke=az network vnet show `
    --name $HubVnet --resource-group $HubRg `
    --query 'virtualNetworkPeerings[0].name' `
    --output tsv

az network vnet peering show `
    --name $HubPeeringToSpoke --vnet-name $HubVnet --resource-group $HubRg `
    --query '{name:name,peeringState:peeringState, allowGatewayTransit:allowGatewayTransit,useRemoteGateways:useRemoteGateways}' `
    --output table
    
$SpokePeeringToHub=az network vnet show `
    --name $SpokeVnet --resource-group $SpokeRg `
    --query 'virtualNetworkPeerings[0].name' `
    --output tsv
    
az network vnet peering show `
    --name $SpokePeeringToHub --vnet-name $SpokeVnet --resource-group $SpokeRg `
    --query '{name:name,peeringState:peeringState,allowGatewayTransit:allowGatewayTransit,useRemoteGateways:useRemoteGateways}' `
    --output table
```

### Task 3: Verify virtual machines

```powershell
az vm list `
    --resource-group $HubRg --show-details `
    --query "[].{Name:name,State:powerState,privateIps:privateIps,Location:location,RG:resourceGroup,osType:storageProfile.osDisk.osType}" `
    --output table

az vm list `
    --resource-group $SpokeRg --show-details `
    --query "[].{Name:name,State:powerState,privateIps:privateIps,Location:location,RG:resourceGroup,osType:storageProfile.osDisk.osType}" `
    --output table
```

### Task 4: Verify custom route

```powershell
$RouteTable=az network route-table list --resource-group $SpokeRg --query "[].name" --output tsv

az network route-table show `
    --name $RouteTable --resource-group $SpokeRg `
    --query '{RouteTable:name,Route:routes[0].name,Prefix:routes[0].addressPrefix,NextHop:routes[0].nextHopIpAddress}' `
    --output table
```


## Exercise 3: Create virtual gateway
### Task 1: Create gateway subnet

```powershell
az network vnet subnet create `
    --name GatewaySubnet --address-prefix $GatewaySubnetPrefix `
    --vnet-name $HubVnet --resource-group $HubRg

az network vnet subnet list `
    --vnet-name $HubVnet --resource-group $HubRg --output table
```

### Task 2: Create gateway public ip

```powershell
$GatewayPip="$GatewayName-Pip"
az network public-ip create `
    --name $GatewayPip --resource-group $HubRg `
    --allocation-method dynamic --location $Location

az network public-ip list `
    --resource-group $HubRg `
    --query "[].{Name:name,Address:ipAddress,Method:publicIpAllocationMethod}" `
    --output table
```

### Task 3: Create gateway

```powershell
az network vnet-gateway create `
    --name $GatewayName --vnet $HubVnet `
    --resource-group $HubRg --location $Location `
    --address-prefixes $GatewayAddressPool `
    --public-ip-addresses $GatewayPip `
    --vpn-type RouteBased --sku Basic `
    --no-wait

az network vnet-gateway list `
    --resource-group $HubRg `
    --query '[].{name:name,provisioningState:provisioningState,resourceGroup:resourceGroup,location:location,vpnType:vpnType}' `
    --output table
```

### Task 4: Configure *Allow gateway transit* on hub
[docs](https://docs.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-peering-gateway-transit)

```powershell
az network vnet peering update `
    --name $HubPeeringToSpoke --vnet-name $HubVnet --resource-group $HubRg `
    --set allowGatewayTransit=true

az network vnet peering show `
    --name $HubPeeringToSpoke --vnet-name $HubVnet --resource-group $HubRg `
    --query '{name:name,peeringState:peeringState,allowGatewayTransit:allowGatewayTransit,useRemoteGateways:useRemoteGateways}' `
    --output table
```


### Task 5: Configure *Use remote gateway* on spoke

```powershell
az network vnet peering update `
    --name $SpokePeeringToHub --vnet-name $SpokeVnet --resource-group $SpokeRg `
    --set useRemoteGateways=true

az network vnet peering show `
    --name $SpokePeeringToHub --vnet-name $SpokeVnet --resource-group $SpokeRg `
    --query '{name:name,peeringState:peeringState,allowGatewayTransit:allowGatewayTransit,useRemoteGateways:useRemoteGateways}' `
    --output table
```

### Task 6: Configure *Virtual network gateway route propagation*
Warum? Damit man von von VM3 aus den PC anpingen kann (ping 192.168.0.2).
[docs](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#custom-routes)

```powershell
az network route-table update `
    --name $RouteTable --resource-group $SpokeRg `
    --set 'disableBgpRoutePropagation=false' 

az network route-table show `
    --name $RouteTable --resource-group $SpokeRg `
    --query '{name:name,disableBgpRoutePropagation:disableBgpRoutePropagation}' `
    --output table
```

### Task 7: Create root and client certificates

```powershell
$RootCertificate = New-SelfSignedCertificate `
    -FriendlyName $RootCertificateName `
    -Subject "CN=$RootCertificateName" `
    -Type Custom `
    -KeySpec Signature `
    -KeyExportPolicy Exportable `
    -HashAlgorithm sha256 `
    -KeyLength 2048 `
    -KeyUsageProperty Sign `
    -KeyUsage CertSign `
    -CertStoreLocation 'Cert:\CurrentUser\My'

$ClientCertificate = New-SelfSignedCertificate `
    -FriendlyName $ClientCertificateName `
    -Subject "CN=$ClientCertificateName" `
    -Type Custom `
    -KeySpec Signature `
    -KeyExportPolicy Exportable `
    -HashAlgorithm sha256 `
    -KeyLength 2048 `
    -Signer $RootCertificate `
    -TextExtension @("2.5.29.37={text}1.3.6.1.5.5.7.3.2") `
    -CertStoreLocation 'Cert:\CurrentUser\My'

Get-ChildItem Cert:\CurrentUser\My | 
    where {$_.Subject -eq "CN=$RootCertificateName" -or $_.Subject -eq "CN=$ClientCertificateName"} | 
    ft Subject, Issuer, Thumbprint
```

### Task 8: Copy root certificate to gateway configuration

```powershell
$RootCertPublicData=[System.Convert]::ToBase64String($RootCertificate.RawData)

az network vnet-gateway root-cert create `
    --name $RootCertificateName `
    --public-cert-data $RootCertPublicData `
    --gateway-name $GatewayName  --resource-group $HubRg
```




## Exercise 4: Add VPN to PC and Test
### Task 1: Download and install VPN client on PC

```powershell
$Uri=az network vnet-gateway vpn-client generate `
    --processor-architecture Amd64 `
    --name $GatewayName --resource-group $HubRg `
    --output tsv

$VpnZipPath="$env:HOMEPATH\Downloads"

Invoke-RestMethod -Uri $Uri -OutFile $VpnZipPath\VpnClient.zip

Expand-Archive -Path $VpnZipPath\VpnClient.zip -DestinationPath $VpnZipPath\VpnClient

$VpnZipPath\VpnClient\WindowsAmd64\VpnClientSetupAmd64.exe

cmd.exe /C "start ms-settings:network-vpn"
```

### Task 2: Test VPN connection

```powershell
Get-NetIPConfiguration | where InterfaceAlias -eq $HubVnet
Test-NetConnection 10.0.0.4 -Traceroute
Test-NetConnection 10.0.1.4 -Traceroute
Test-NetConnection 10.0.4.4 -Traceroute
```

### Task 3. Dissociate public IP addresses
Public IP addresses of VMs are not needed any more.

```powershell
# VM --> NIC Id
$Vm1NicId=az vm nic list --vm-name $Vm1 --resource-group $HubRg --query "[0].id" --output tsv
$Vm2NicId=az vm nic list --vm-name $Vm2 --resource-group $HubRg --query "[0].id" --output tsv
$Vm3NicId=az vm nic list --vm-name $Vm3 --resource-group $SpokeRg --query "[0].id" --output tsv

# NIC Id --> IpConfig Id
$Vm1IpconfigId=az network nic show --ids $Vm1NicId --query "ipConfigurations[0].id" --output tsv
$Vm2IpconfigId=az network nic show --ids $Vm2NicId --query "ipConfigurations[0].id" --output tsv
$Vm3IpconfigId=az network nic show --ids $Vm3NicId --query "ipConfigurations[0].id" --output tsv

# IpConfig Id --> PublicIp Id
$Vm1PublicIpId=az network nic ip-config show --ids $Vm1IpconfigId --query "publicIpAddress.id" --output tsv
$Vm2PublicIpId=az network nic ip-config show --ids $Vm2IpconfigId --query "publicIpAddress.id" --output tsv
$Vm3PublicIpId=az network nic ip-config show --ids $Vm3IpconfigId --query "publicIpAddress.id" --output tsv

# Ipconfig ändern (public IP dissoziieren)
az network nic ip-config update --ids $Vm1IpconfigId --remove publicIpAddress
az network nic ip-config update --ids $Vm2IpconfigId --remove publicIpAddress
az network nic ip-config update --ids $Vm3IpconfigId --remove publicIpAddress

# Die public IPs sind nicht mehr mit den VMs assoziiert ...
az vm list-ip-addresses --output table

# ... aber als IP sind sie noch da
az network public-ip list --output table

# Public IP löschen
az network public-ip delete --ids $Vm1PublicIpId
az network public-ip delete --ids $Vm2PublicIpId
az network public-ip delete --ids $Vm3PublicIpId
```




## Exersise 5: Clean up
### Task 1: Remove Azure resources

```powershell
az resource list --resource-group $HubRg --query "[].{Name:name, Type:type}" --output table
az group delete --name $HubRg --yes --no-wait

az resource list --resource-group $SpokeRg --query "[].{Name:name, Type:type}" --output table
az group delete --name $SpokeRg --yes --no-wait
```

### Task 2: Remove VPN aon PC

```powershell
cmd.exe /C "start ms-settings:network-vpn"
Remove-Item -Path $ClientCertificate.PSPath
Remove-Item -Path $RootCertificate.PSPath
Remove-Item -Path $VpnZipPath\VpnClient.zip
Remove-Item -Path $VpnZipPath\VpnClient -Recurse
```
