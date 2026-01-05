# CREATE-A-VM-INSTANCE-WITH-PS

# =============================================================================
# SCRIPT COMPLET : Créer une VM Azure de ZÉRO avec PowerShell
# Copier-coller prêt pour GitHub
# =============================================================================
# Remplace TOUS les <...> par tes valeurs avant exécution !
# =============================================================================

# 1. PRÉ-REQUIS (exécuter une fois)
# Install-Module -Name Az -Scope CurrentUser -Force
# Connect-AzAccount
# Set-AzContext -Subscription "<Ton-ID-Abonnement>"

# 2. VARIABLES - MODIFIER ICI 👇
$location          = "westeurope"        # Région Azure
$resourceGroupName = "RG-MaVM-$(Get-Date -Format 'yyyyMMdd-HHmm')"
$vmName            = "MaVM-$(Get-Date -Format 'yyyyMMdd-HHmm')"
$vnetName          = "VNET-$($vmName)"
$subnetName        = "Subnet-$($vmName)"
$publicIpName      = "PIP-$($vmName)"
$nsgName           = "NSG-$($vmName)"
$nicName           = "NIC-$($vmName)"
$vmSize            = "Standard_B1s"      # B1s=petite/éco, D2s_v3=plus puissant

# VM Windows ou Linux ? (true=Windows, false=Linux)
$isWindowsVM       = $true

# CREDENTIELS VM
$adminUser         = "vmadmin"
$adminPassword     = ConvertTo-SecureString "MonMotDePasse123!456" -AsPlainText -Force
$cred              = New-Object System.Management.Automation.PSCredential ($adminUser, $adminPassword)

Write-Host "🚀 Début création VM: $vmName" -ForegroundColor Green
Write-Host "📍 Région: $location" -ForegroundColor Cyan

# =============================================================================
# 3. CRÉATION RESOURCE GROUP + RÉSEAUX
# =============================================================================

# Resource Group
Write-Host "📁 Création Resource Group..." -ForegroundColor Yellow
New-AzResourceGroup -Name $resourceGroupName -Location $location

# VNet + Subnet
Write-Host "🌐 Création VNet + Subnet..." -ForegroundColor Yellow
$vnetAddressPrefix   = "10.0.0.0/16"
$subnetAddressPrefix = "10.0.1.0/24"
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix $subnetAddressPrefix
$vnet = New-AzVirtualNetwork -Name $vnetName -ResourceGroupName $resourceGroupName -Location $location -AddressPrefix $vnetAddressPrefix -Subnet $subnetConfig

# NSG (RDP 3389 + SSH 22)
Write-Host "🔒 Création NSG..." -ForegroundColor Yellow
$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroupName -Location $location -Name $nsgName
$nsg | Add-AzNetworkSecurityRuleConfig -Name "Allow-RDP" -Protocol "Tcp" -Direction "Inbound" -Priority 1000 -SourceAddressPrefix "*" -SourcePortRange "*" -DestinationAddressPrefix "*" -DestinationPortRange 3389 -Access "Allow" | Set-AzNetworkSecurityGroup
$nsg | Add-AzNetworkSecurityRuleConfig -Name "Allow-SSH" -Protocol "Tcp" -Direction "Inbound" -Priority 1001 -SourceAddressPrefix "*" -SourcePortRange "*" -DestinationAddressPrefix "*" -DestinationPortRange 22 -Access "Allow" | Set-AzNetworkSecurityGroup

# IP Publique
Write-Host "🌍 Création IP Publique..." -ForegroundColor Yellow
$publicIp = New-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroupName -Location $location -AllocationMethod Static -Sku Basic

# NIC
Write-Host "🔌 Création NIC..." -ForegroundColor Yellow
$subnet = Get-AzVirtualNetworkSubnetConfig -Name $subnetName -VirtualNetwork $vnet
$nic = New-AzNetworkInterface -Name $nicName -ResourceGroupName $resourceGroupName -Location $location -Subnet $subnet -NetworkSecurityGroup $nsg -PublicIpAddress $publicIp

# =============================================================================
# 4. CRÉATION VM
# =============================================================================

Write-Host "🖥️  Création VM..." -ForegroundColor Yellow

if ($isWindowsVM) {
    # WINDOWS SERVER 2022
    $vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize | `
        Set-AzVMOperatingSystem -Windows -ComputerName $vmName -Credential $cred -ProvisionVMAgent -EnableAutoUpdate | `
        Set-AzVMSourceImage -PublisherName "MicrosoftWindowsServer" -Offer "WindowsServer" -Skus "2022-datacenter-azure-edition" -Version "latest" | `
        Add-AzVMNetworkInterface -Id $nic.Id
} else {
    # UBUNTU 22.04
    $vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize | `
        Set-AzVMOperatingSystem -Linux -ComputerName $vmName -Credential $cred -DisablePasswordAuthentication:$false | `
        Set-AzVMSourceImage -PublisherName "Canonical" -Offer "0001-com-ubuntu-server-focal" -Skus "22_04-lts-gen2" -Version "latest" | `
        Add-AzVMNetworkInterface -Id $nic.Id
}

New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig

# =============================================================================
# 5. INFO FINALES
# =============================================================================
Write-Host "`n✅ VM CRÉÉE AVEC SUCCÈS !" -ForegroundColor Green
Write-Host "`n📋 RÉSUMÉ :" -ForegroundColor Cyan
Write-Host "   RG: $resourceGroupName"
Write-Host "   VM:  $vmName"
Write-Host "   IP Publique: $(Get-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroupName).IpAddress"
Write-Host "`n🔗 Connexion RDP/SSH avec l'IP ci-dessus" -ForegroundColor Magenta
Write-Host "   User: $adminUser"
Write-Host "   Pass: MonMotDePasse123!456 (CHANGEZ-LE !)" -ForegroundColor Red
