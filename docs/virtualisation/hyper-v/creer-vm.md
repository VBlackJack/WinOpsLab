---
title: "Creer une machine virtuelle"
description: "Creation et configuration de VMs Hyper-V : Generation 1 vs 2, CPU, RAM dynamique, disques virtuels et services d'integration."
tags:
  - virtualisation
  - hyper-v
  - vm
  - configuration
---

# Creer une machine virtuelle

<span class="level-intermediate">Intermediaire</span> · Temps estime : 30 minutes

La creation d'une machine virtuelle Hyper-V implique plusieurs choix architecturaux : generation de la VM, allocation des ressources (CPU, RAM, stockage) et configuration des services d'integration.

---

## Generation 1 vs Generation 2

| Critere | Generation 1 | Generation 2 |
|---------|-------------|-------------|
| **Firmware** | BIOS legacy | UEFI |
| **Secure Boot** | Non | Oui |
| **Boot** | IDE, CD/DVD, PXE | SCSI, PXE via carte reseau standard |
| **Disque de boot** | IDE (max 2 To) | SCSI (max 64 To) |
| **OS invites** | Windows, Linux anciens | Windows 8+, Server 2012+, Linux UEFI |
| **TPM virtuel** | Non | Oui (vTPM) |
| **Redimensionnement VHDX online** | Non | Oui |
| **Hot Add/Remove** | Non | Memoire et adaptateurs reseau |

!!! tip "Recommandation"

    Choisissez **Generation 2** pour toute nouvelle VM avec un OS compatible. Generation 1 est reservee aux anciens systemes (Windows Server 2008, Linux ancien).

---

## Creer une VM via PowerShell

### Creation basique

```powershell
# Create a Generation 2 VM
New-VM -Name "SRV-APP01" `
    -Generation 2 `
    -MemoryStartupBytes 4GB `
    -NewVHDPath "D:\Hyper-V\Virtual Hard Disks\SRV-APP01.vhdx" `
    -NewVHDSizeBytes 60GB `
    -SwitchName "vSwitch-External"

# Set the number of virtual processors
Set-VMProcessor -VMName "SRV-APP01" -Count 4

# Attach an ISO for OS installation
Add-VMDvdDrive -VMName "SRV-APP01" -Path "D:\ISO\WindowsServer2022.iso"

# Set boot order (DVD first for installation)
$dvd = Get-VMDvdDrive -VMName "SRV-APP01"
Set-VMFirmware -VMName "SRV-APP01" -FirstBootDevice $dvd
```

### Creation complete avec parametres avances

```powershell
# Full VM creation script with all recommended settings
$vmName = "SRV-SQL01"
$vmPath = "D:\Hyper-V\Virtual Machines"
$vhdPath = "D:\Hyper-V\Virtual Hard Disks"

# Create the VM
New-VM -Name $vmName `
    -Generation 2 `
    -MemoryStartupBytes 8GB `
    -Path $vmPath `
    -NewVHDPath "$vhdPath\$vmName-OS.vhdx" `
    -NewVHDSizeBytes 80GB `
    -SwitchName "vSwitch-External"

# Configure processors
Set-VMProcessor -VMName $vmName -Count 4 -Reserve 10 -Maximum 80

# Configure memory
Set-VMMemory -VMName $vmName `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 4GB `
    -StartupBytes 8GB `
    -MaximumBytes 16GB `
    -Buffer 20

# Add a second data disk
New-VHD -Path "$vhdPath\$vmName-Data.vhdx" -SizeBytes 200GB -Dynamic
Add-VMHardDiskDrive -VMName $vmName -Path "$vhdPath\$vmName-Data.vhdx"

# Configure Secure Boot
Set-VMFirmware -VMName $vmName -EnableSecureBoot On

# Enable vTPM
Enable-VMTPM -VMName $vmName

# Configure automatic start/stop actions
Set-VM -VMName $vmName `
    -AutomaticStartAction Start `
    -AutomaticStartDelay 60 `
    -AutomaticStopAction ShutDown `
    -Notes "SQL Server production VM"
```

---

## Configuration CPU

### Allocation de processeurs virtuels

```powershell
# Set virtual processor count
Set-VMProcessor -VMName "SRV-APP01" -Count 4

# Configure resource controls
Set-VMProcessor -VMName "SRV-APP01" `
    -Count 4 `
    -Reserve 10 `          # Minimum CPU % guaranteed
    -Maximum 80 `          # Maximum CPU % allowed
    -RelativeWeight 100    # Priority relative to other VMs (1-10000)
```

### Compatibilite du processeur

```powershell
# Enable processor compatibility (for live migration between different CPU generations)
Set-VMProcessor -VMName "SRV-APP01" -CompatibilityForMigrationEnabled $true

# Verify processor configuration
Get-VMProcessor -VMName "SRV-APP01" |
    Select-Object Count, Reserve, Maximum, RelativeWeight, CompatibilityForMigrationEnabled
```

!!! warning "Surallocation CPU"

    Hyper-V permet d'allouer plus de vCPU que de coeurs physiques disponibles. C'est acceptable pour des charges non-critiques mais deconseille pour des VMs de production a forte charge.

---

## Memoire dynamique

La memoire dynamique ajuste automatiquement la RAM allouee a une VM en fonction de sa charge.

```mermaid
flowchart LR
    subgraph Dynamic Memory
        MIN[Minimum<br/>2 Go] --> START[Startup<br/>4 Go] --> MAX[Maximum<br/>8 Go]
    end
    BUFFER[Buffer 20%] --> Dynamic Memory
```

### Configuration

```powershell
# Enable dynamic memory
Set-VMMemory -VMName "SRV-APP01" `
    -DynamicMemoryEnabled $true `
    -MinimumBytes 2GB `
    -StartupBytes 4GB `
    -MaximumBytes 8GB `
    -Buffer 20               # Percentage of memory buffer

# View memory allocation
Get-VM -Name "SRV-APP01" | Select-Object Name, MemoryAssigned, MemoryDemand,
    MemoryStartup, DynamicMemoryEnabled
```

### Parametres

| Parametre | Description |
|-----------|-------------|
| **Minimum** | RAM minimale que la VM conserve toujours |
| **Startup** | RAM allouee au demarrage |
| **Maximum** | Plafond de RAM que la VM peut atteindre |
| **Buffer** | Pourcentage de memoire supplementaire maintenu au-dessus de la demande |

!!! tip "Quand desactiver la memoire dynamique"

    - **SQL Server** : utilise un gestionnaire de memoire interne, peut entrer en conflit avec la memoire dynamique
    - **Exchange Server** : necessaire d'avoir une allocation fixe pour la performance
    - **VMs critiques** : quand une allocation garantie est requise

---

## Services d'integration

Les services d'integration ameliorent la communication et les performances entre l'hote et la VM.

| Service | Description |
|---------|-------------|
| **Heartbeat** | L'hote verifie que la VM repond |
| **Time Synchronization** | Synchronisation de l'horloge avec l'hote |
| **Data Exchange** | Echange de paires cle-valeur via le registre |
| **Shutdown** | Arret propre de la VM depuis l'hote |
| **VSS (Volume Shadow Copy)** | Backups coherentes de la VM |
| **Guest Services** | Copie de fichiers entre l'hote et la VM |

```powershell
# View integration services status
Get-VMIntegrationService -VMName "SRV-APP01" |
    Select-Object Name, Enabled, PrimaryOperationalStatus

# Enable Guest Services (file copy between host and VM)
Enable-VMIntegrationService -VMName "SRV-APP01" -Name "Guest Service Interface"

# Disable Time Synchronization (recommended for domain-joined VMs)
Disable-VMIntegrationService -VMName "SRV-APP01" -Name "Time Synchronization"
```

!!! warning "Synchronisation du temps"

    Pour les VMs jointes a un domaine Active Directory, **desactivez** le service Time Synchronization d'Hyper-V. Les VMs doivent synchroniser leur horloge avec le controleur de domaine (hierarchie NTP du domaine), pas avec l'hote Hyper-V.

---

## Gestion du cycle de vie

```powershell
# Start a VM
Start-VM -Name "SRV-APP01"

# Stop a VM gracefully
Stop-VM -Name "SRV-APP01"

# Force stop a VM (equivalent to power off)
Stop-VM -Name "SRV-APP01" -Force -TurnOff

# Restart a VM
Restart-VM -Name "SRV-APP01"

# Save a VM state (hibernate)
Save-VM -Name "SRV-APP01"

# Resume from saved state
Start-VM -Name "SRV-APP01"

# List all VMs with their status
Get-VM | Select-Object Name, State, CPUUsage,
    @{N='MemoryGB';E={[math]::Round($_.MemoryAssigned/1GB, 2)}},
    Uptime, Status |
    Format-Table -AutoSize
```

---

## Export et import de VMs

```powershell
# Export a VM (creates a portable copy)
Export-VM -Name "SRV-APP01" -Path "E:\VM-Exports"

# Import a VM
$vmConfigPath = "E:\VM-Exports\SRV-APP01\Virtual Machines\*.vmcx"
Import-VM -Path (Get-ChildItem $vmConfigPath).FullName -Copy -GenerateNewId
```

---

## Points cles a retenir

- **Generation 2** est le choix par defaut pour les OS modernes (UEFI, Secure Boot, vTPM)
- La **memoire dynamique** optimise l'utilisation de la RAM mais n'est pas adaptee a toutes les charges
- Les **services d'integration** doivent etre configures selon le contexte (desactiver Time Sync pour les VMs en domaine AD)
- Ne pas **surallouer** les vCPU en production
- Activer **Secure Boot** et **vTPM** pour les VMs Generation 2
- Configurer les **actions automatiques** de demarrage et d'arret pour chaque VM

---

## Pour aller plus loin

- Reseaux virtuels Hyper-V (voir la page [Reseaux virtuels](reseaux-virtuels.md))
- Stockage virtuel (voir la page [Stockage virtuel](stockage-virtuel.md))
- Checkpoints (voir la page [Checkpoints](checkpoints.md))
