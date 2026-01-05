
# CREATE-A-VM-INSTANCE-WITH-PS
# Script PowerShell - Création VM Azure 

# ---------- PRÉREQUIS (une seule fois) ----------
```bash
Install-Module Az -Scope CurrentUser -Force
Connect-AzAccount
Set-AzContext -Subscription "<SUBSCRIPTION_ID>"
```
# ---------- VARIABLES ----------
```bash
$location = "westeurope"

$timestamp = Get-Date -Format "yyyyMMdd-HHmm"
$resourceGroupName = "RG-MaVM-$timestamp"
$vmName       = "MaVM-$timestamp"
$vnetName     = "VNET-$vmName"
$subnetName   = "Subnet-$vmName"
$publicIpName = "PIP-$vmName"
$nsgName      = "NSG-$vmName"
$nicName      = "NIC-$vmName"

$vmSize = "Standard_B1s"
$isWindowsVM = $true   # true = Windows | false = Linux
```
# ---------- IDENTIFIANTS VM ----------
```bash
$adminUser = "vmadmin"
$adminPassword = ConvertTo-SecureString "MonMotDePasse123!456" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ($adminUser, $adminPassword)
```
# ---------- RESOURCE GROUP ----------
```bash
New-AzResourceGroup -Name $resourceGroupName -Location $location
```
# ---------- VNET + SUBNET ----------
```bash
$subnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name $subnetName `
  -AddressPrefix "10.0.1.0/24"
```
```bash
$vnet = New-AzVirtualNetwork `
  -Name $vnetName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -AddressPrefix "10.0.0.0/16" `
  -Subnet $subnetConfig
```
# ---------- NSG ----------
```bash
$nsg = New-AzNetworkSecurityGroup `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -Name $nsgName
```
```bash
$nsg | Add-AzNetworkSecurityRuleConfig `
  -Name "Allow-RDP" `
  -Protocol Tcp `
  -Direction Inbound `
  -Priority 1000 `
  -SourceAddressPrefix "*" `
  -SourcePortRange "*" `
  -DestinationAddressPrefix "*" `
  -DestinationPortRange 3389 `
  -Access Allow | Set-AzNetworkSecurityGroup
```

``` bash
# SSH 
$nsg | Add-AzNetworkSecurityRuleConfig -Name "Allow-SSH" -Protocol Tcp -Direction Inbound -Priority 1001 -SourceAddressPrefix "*" -SourcePortRange "*" -DestinationAddressPrefix "*" -DestinationPortRange 22 -Access Allow | Set-AzNetworkSecurityGroup
````
# ---------- IP PUBLIQUE ----------
```bash
$publicIp = New-AzPublicIpAddress `
  -Name $publicIpName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -AllocationMethod Static
```
# ---------- NIC ----------
```bash
$subnet = Get-AzVirtualNetworkSubnetConfig -Name $subnetName -VirtualNetwork $vnet

$nic = New-AzNetworkInterface `
  -Name $nicName `
  -ResourceGroupName $resourceGroupName `
  -Location $location `
  -Subnet $subnet `
  -NetworkSecurityGroup $nsg `
  -PublicIpAddress $publicIp
```
# ---------- VM ----------
```bash
if ($isWindowsVM) {
    $vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize |
      Set-AzVMOperatingSystem -Windows -ComputerName $vmName -Credential $cred -ProvisionVMAgent |
      Set-AzVMSourceImage -PublisherName "MicrosoftWindowsServer" -Offer "WindowsServer" -Skus "2022-datacenter-azure-edition-core" -Version "latest" |
      Add-AzVMNetworkInterface -Id $nic.Id
} else {
    $vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize |
      Set-AzVMOperatingSystem -Linux -ComputerName $vmName -Credential $cred -DisablePasswordAuthentication:$false |
      Set-AzVMSourceImage -PublisherName "Canonical" -Offer "0001-com-ubuntu-server-jammy" -Skus "22_04-lts-gen2" -Version "latest" |
      Add-AzVMNetworkInterface -Id $nic.Id
}

New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig

```
# ---------- INFOS ----------
```bash
$ip = (Get-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroupName).IpAddress
Write-Host "✅ VM créée avec succès"
Write-Host "IP publique : $ip"

```
