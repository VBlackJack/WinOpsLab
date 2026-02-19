---
title: Creation d'un pool de stockage
description: Creer et gerer des pools de stockage et des disques virtuels avec PowerShell et Server Manager sous Windows Server 2022.
tags:
  - stockage
  - storage-spaces
  - intermediaire
---

# Creation d'un pool de stockage

!!! info "Niveau : Intermediaire"

    Temps estime : 30 minutes

## Prerequis

Avant de creer un pool de stockage, verifiez que :

- Vous disposez d'au moins **un disque physique non initialise** (sans partition ni volume)
- Le ou les disques ne font pas partie d'un autre pool
- Le disque systeme ne sera pas inclus dans le pool

```powershell
# List available physical disks (not in a pool, not the boot disk)
Get-PhysicalDisk -CanPool $true |
    Select-Object DeviceId, FriendlyName, MediaType,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
        CanPool |
    Format-Table -AutoSize
```

!!! tip "Disques non eligibles"

    Si `Get-PhysicalDisk -CanPool $true` ne retourne aucun resultat, verifiez que les disques sont vierges (non initialises). Utilisez `Clear-Disk` pour les nettoyer.

    ```powershell
    # Clean a disk (removes all partitions and data)
    Clear-Disk -Number 1 -RemoveData -RemoveOEM -Confirm:$false
    ```

## Creation avec PowerShell

### Etape 1 : Creer le pool de stockage

```powershell
# Get the storage subsystem
$subsystem = Get-StorageSubSystem -FriendlyName "*Windows*"

# Get eligible physical disks
$disks = Get-PhysicalDisk -CanPool $true

# Create the storage pool
New-StoragePool -FriendlyName "PoolDonnees" `
    -StorageSubSystemFriendlyName $subsystem.FriendlyName `
    -PhysicalDisks $disks
```

Pour selectionner des disques specifiques :

```powershell
# Select specific disks by DeviceId
$disks = Get-PhysicalDisk -CanPool $true |
    Where-Object { $_.DeviceId -in @("1", "2", "3") }

New-StoragePool -FriendlyName "PoolDonnees" `
    -StorageSubSystemFriendlyName $subsystem.FriendlyName `
    -PhysicalDisks $disks
```

### Etape 2 : Creer un disque virtuel

```powershell
# Two-way mirror virtual disk, fixed provisioning
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Mirror" `
    -ResiliencySettingName Mirror `
    -Size 100GB `
    -ProvisioningType Fixed
```

Options de resilience :

```powershell
# Simple (no resilience)
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Simple" `
    -ResiliencySettingName Simple `
    -Size 200GB `
    -ProvisioningType Thin

# Parity (RAID 5 equivalent)
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Parity" `
    -ResiliencySettingName Parity `
    -Size 500GB `
    -ProvisioningType Fixed

# Three-way mirror (requires 5+ disks)
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Mirror3" `
    -ResiliencySettingName Mirror `
    -NumberOfDataCopies 3 `
    -Size 50GB
```

!!! info "Provisioning dynamique (Thin)"

    Avec `-ProvisioningType Thin`, la taille du disque virtuel peut depasser l'espace physique disponible dans le pool. L'espace est alloue a la demande au fur et a mesure de l'utilisation.

### Etape 3 : Initialiser et formater le volume

```powershell
# Initialize the virtual disk
$vd = Get-VirtualDisk -FriendlyName "VD-Mirror"
$vd | Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "Donnees" -Confirm:$false
```

### Pipeline complet en une commande

```powershell
# Full pipeline: pool -> virtual disk -> volume
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Apps" `
    -ResiliencySettingName Mirror `
    -Size 100GB `
    -ProvisioningType Fixed |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "Applications" -Confirm:$false
```

## Creation avec les tiers de stockage

Si votre pool contient des disques SSD et HDD, vous pouvez creer un disque virtuel avec tiers automatiques.

### Configurer les tiers

```powershell
# Verify media types are correctly detected
Get-PhysicalDisk -StoragePool (Get-StoragePool -FriendlyName "PoolDonnees") |
    Select-Object FriendlyName, MediaType,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}} |
    Format-Table -AutoSize

# View available storage tiers
Get-StorageTier -MediaType SSD | Select-Object FriendlyName, MediaType, ResiliencySettingName
Get-StorageTier -MediaType HDD | Select-Object FriendlyName, MediaType, ResiliencySettingName
```

### Creer un disque virtuel avec tiers

```powershell
# Get tier references
$ssdTier = Get-StorageTier -FriendlyName "*SSD*" |
    Where-Object { $_.ResiliencySettingName -eq "Mirror" }
$hddTier = Get-StorageTier -FriendlyName "*HDD*" |
    Where-Object { $_.ResiliencySettingName -eq "Parity" }

# Create tiered virtual disk (mirror on SSD, parity on HDD)
New-VirtualDisk -StoragePoolFriendlyName "PoolDonnees" `
    -FriendlyName "VD-Tiered" `
    -StorageTiers $ssdTier, $hddTier `
    -StorageTierSizes 20GB, 200GB `
    -WriteCacheSize 1GB
```

!!! tip "Write-back cache"

    Le parametre `-WriteCacheSize` configure un cache d'ecriture sur SSD. Les ecritures sont d'abord stockees sur le tier SSD rapide, puis migrees vers le tier HDD en arriere-plan. La valeur par defaut est 1 Go.

## Creation avec Server Manager

L'interface graphique offre un assistant guide pour la creation de pools et disques virtuels.

### Etape 1 : Ouvrir l'assistant

1. Ouvrir **Server Manager** > **File and Storage Services** > **Volumes** > **Storage Pools**
2. Dans la zone **PHYSICAL DISKS**, verifier les disques disponibles
3. Dans **TASKS** (barre superieure), cliquer sur **New Storage Pool...**

### Etape 2 : Creer le pool

1. Nommer le pool et ajouter une description
2. Selectionner les disques physiques a inclure
3. Definir le role de chaque disque (Auto-Select ou Hot Spare)
4. Confirmer et creer

### Etape 3 : Creer le disque virtuel

1. Dans le pool nouvellement cree, cliquer **New Virtual Disk...**
2. Choisir la disposition de stockage (Simple, Mirror, Parity)
3. Selectionner le provisioning (Fixed ou Thin)
4. Definir la taille
5. L'assistant propose ensuite de creer un volume automatiquement

## Gestion courante

### Surveiller l'etat du pool

```powershell
# Pool health overview
Get-StoragePool | Where-Object { $_.IsPrimordial -eq $false } |
    Select-Object FriendlyName, HealthStatus, OperationalStatus,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
        @{N='AllocatedGB';E={[math]::Round($_.AllocatedSize/1GB,2)}},
        @{N='FreeGB';E={[math]::Round(($_.Size - $_.AllocatedSize)/1GB,2)}} |
    Format-Table -AutoSize

# Virtual disk health
Get-VirtualDisk |
    Select-Object FriendlyName, HealthStatus, OperationalStatus,
        ResiliencySettingName,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
        @{N='UsedGB';E={[math]::Round($_.FootprintOnPool/1GB,2)}} |
    Format-Table -AutoSize

# Physical disk health
Get-PhysicalDisk -StoragePool (Get-StoragePool -FriendlyName "PoolDonnees") |
    Select-Object FriendlyName, HealthStatus, OperationalStatus, Usage,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}} |
    Format-Table -AutoSize
```

### Ajouter un disque au pool

```powershell
# Add a new physical disk to an existing pool
$newDisk = Get-PhysicalDisk -CanPool $true | Where-Object { $_.DeviceId -eq "4" }
Add-PhysicalDisk -StoragePoolFriendlyName "PoolDonnees" -PhysicalDisks $newDisk
```

### Etendre un disque virtuel

```powershell
# Extend a virtual disk (after adding disks to the pool)
Resize-VirtualDisk -FriendlyName "VD-Mirror" -Size 200GB

# Then extend the partition on the volume
$partition = Get-VirtualDisk -FriendlyName "VD-Mirror" |
    Get-Disk | Get-Partition | Where-Object { $_.Type -eq "Basic" }
$maxSize = ($partition | Get-PartitionSupportedSize).SizeMax
$partition | Resize-Partition -Size $maxSize
```

### Retirer un disque du pool

```powershell
# Mark a disk as retired (data will be migrated to remaining disks)
Set-PhysicalDisk -FriendlyName "Disk3" -Usage Retired

# Remove the disk after migration completes
Remove-PhysicalDisk -StoragePoolFriendlyName "PoolDonnees" -PhysicalDisks (
    Get-PhysicalDisk -FriendlyName "Disk3"
)
```

!!! warning "Espace suffisant"

    Le retrait d'un disque ne reussit que si les disques restants ont assez d'espace pour accueillir les donnees migrees, tout en respectant le niveau de resilience.

### Reparer un disque virtuel degrade

```powershell
# Check degraded virtual disks
Get-VirtualDisk | Where-Object { $_.HealthStatus -ne "Healthy" } |
    Select-Object FriendlyName, HealthStatus, OperationalStatus

# Trigger repair (automatic if hot spare available)
Repair-VirtualDisk -FriendlyName "VD-Mirror"
```

### Supprimer un pool et ses composants

```powershell
# Remove volume, partition, virtual disk, then pool
$vd = Get-VirtualDisk -FriendlyName "VD-Mirror"
$vd | Get-Disk | Get-Partition | Where-Object { $_.Type -eq "Basic" } |
    Remove-Partition -Confirm:$false
$vd | Remove-VirtualDisk -Confirm:$false

# Remove the storage pool
Remove-StoragePool -FriendlyName "PoolDonnees" -Confirm:$false
```

## Exemples de configurations types

### Serveur de fichiers (PME)

```powershell
# 4 x 2 TB HDD -> Mirror pool for file shares
$subsystem = Get-StorageSubSystem -FriendlyName "*Windows*"
$disks = Get-PhysicalDisk -CanPool $true

New-StoragePool -FriendlyName "Pool-Fichiers" `
    -StorageSubSystemFriendlyName $subsystem.FriendlyName `
    -PhysicalDisks $disks

New-VirtualDisk -StoragePoolFriendlyName "Pool-Fichiers" `
    -FriendlyName "VD-Partages" `
    -ResiliencySettingName Mirror `
    -UseMaximumSize `
    -ProvisioningType Fixed |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -UseMaximumSize -DriveLetter F |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "Partages" -Confirm:$false
```

### Serveur Hyper-V (performances + capacite)

```powershell
# 2 x SSD + 4 x HDD -> Tiered storage for VMs
$ssdTier = Get-StorageTier -FriendlyName "*SSD*" |
    Where-Object { $_.ResiliencySettingName -eq "Mirror" }
$hddTier = Get-StorageTier -FriendlyName "*HDD*" |
    Where-Object { $_.ResiliencySettingName -eq "Mirror" }

New-VirtualDisk -StoragePoolFriendlyName "Pool-HyperV" `
    -FriendlyName "VD-VMs" `
    -StorageTiers $ssdTier, $hddTier `
    -StorageTierSizes 100GB, 500GB `
    -WriteCacheSize 2GB |
    Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -UseMaximumSize -DriveLetter V |
    Format-Volume -FileSystem ReFS -NewFileSystemLabel "VMs" -Confirm:$false
```

## Points cles a retenir

- La creation d'un pool suit trois etapes : **pool** > **disque virtuel** > **volume**
- Seuls les disques **non initialises** et sans partition peuvent etre ajoutes a un pool
- Le mode de resilience (Simple, Mirror, Parity) est choisi a la creation du disque virtuel
- Les **tiers de stockage** (SSD + HDD) optimisent automatiquement le placement des donnees
- Le **provisioning dynamique** (Thin) necessite une surveillance de l'espace du pool
- Les pools peuvent etre etendus a chaud en ajoutant de nouveaux disques physiques
- Utilisez `Get-StoragePool`, `Get-VirtualDisk`, `Get-PhysicalDisk` pour surveiller l'etat

## Pour aller plus loin

- [Concepts de Storage Spaces](concepts.md)
- [Storage Spaces Direct](storage-spaces-direct.md)
- [Volumes et partitions](../disques/volumes-et-partitions.md)
