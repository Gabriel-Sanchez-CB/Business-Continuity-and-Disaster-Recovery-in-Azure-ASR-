Connect-AzAccount

$resourceGroup = "BCDR-RG"
$location = "EastUS"
$vaultName = "BCDR-RecoveryVault"
$policyName = "ASR-Policy"
$fabricName = "AzureFabric"
$containerName = "AzureContainer"

New-AzResourceGroup -Name $resourceGroup -Location $location

$vault = New-AzRecoveryServicesVault -Name $vaultName -ResourceGroupName $resourceGroup -Location $location
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

New-AzRecoveryServicesAsrPolicy `
    -Name $policyName `
    -RecoveryPointRetentionInHours 24 `
    -ApplicationConsistentSnapshotFrequencyInHours 4

$fabric = New-AzRecoveryServicesAsrFabric `
    -Name $fabricName `
    -Type Azure `
    -Location $location

$container = New-AzRecoveryServicesAsrProtectionContainer `
    -Name $containerName `
    -Fabric $fabric

$policy = Get-AzRecoveryServicesAsrPolicy -Name $policyName

Get-AzVM | ForEach-Object {
    New-AzRecoveryServicesAsrReplicationProtectedItem `
        -AzureToAzure `
        -Name $_.Name `
        -ProtectionContainer $container `
        -Policy $policy `
        -VMId $_.Id `
        -RecoveryResourceGroupId (Get-AzResourceGroup -Name $resourceGroup).ResourceId
}

Start-AzRecoveryServicesAsrTestFailoverJob `
    -ReplicationProtectedItem (Get-AzRecoveryServicesAsrReplicationProtectedItem) `
    -Direction PrimaryToRecovery `
    -AzureVMNetworkId (Get-AzVirtualNetwork | Select-Object -First 1).Id
