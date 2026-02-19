---
title: "Lab 07 : Cluster de basculement"
description: Exercice pratique - creer un cluster de basculement a 2 noeuds avec stockage partage et tester le basculement.
tags:
  - lab
  - cluster
  - haute-disponibilite
  - avance
---

# Lab 07 : Cluster de basculement

!!! abstract "Objectifs du lab"

    - [ ] Preparer deux serveurs pour le clustering
    - [ ] Installer la fonctionnalite Failover Clustering
    - [ ] Valider la configuration du cluster
    - [ ] Creer un cluster a 2 noeuds
    - [ ] Configurer un role sur le cluster et tester le basculement

## Scenario

L'entreprise a besoin de haute disponibilite pour un serveur de fichiers critique. Vous devez mettre en place un cluster de basculement a deux noeuds pour garantir la continuite de service.

## Environnement requis

| Ressource | Specification |
|-----------|---------------|
| SRV-DC01 | DC + DNS (reseau du lab) |
| SRV-FILE01 | Noeud 1 du cluster, joint au domaine |
| SRV-DC02 | Noeud 2 du cluster (ou creer SRV-NODE02) |
| Stockage partage | Disque iSCSI ou VHDX partage |

## Instructions

### Partie 1 : Preparer le stockage partage

Creer un disque VHDX partage accessible par les deux noeuds (simule un stockage SAN).

??? success "Solution"

    ```powershell
    # On the Hyper-V HOST: Create a shared VHDX for cluster storage
    New-VHD -Path "G:\VMs\SharedDisks\ClusterDisk.vhdx" -SizeBytes 20GB -Fixed

    # Attach the shared disk to both VMs
    Add-VMHardDiskDrive -VMName "SRV-FILE01" -Path "G:\VMs\SharedDisks\ClusterDisk.vhdx" `
        -SupportPersistentReservations
    Add-VMHardDiskDrive -VMName "SRV-DC02" -Path "G:\VMs\SharedDisks\ClusterDisk.vhdx" `
        -SupportPersistentReservations

    # On SRV-FILE01: Initialize and format the shared disk
    Get-Disk | Where-Object { $_.PartitionStyle -eq "RAW" } |
        Initialize-Disk -PartitionStyle GPT -PassThru |
        New-Partition -UseMaximumSize -DriveLetter S |
        Format-Volume -FileSystem NTFS -NewFileSystemLabel "ClusterStorage" -Confirm:$false
    ```

### Partie 2 : Installer la fonctionnalite Failover Clustering

??? success "Solution"

    ```powershell
    # On BOTH nodes (SRV-FILE01 and SRV-DC02):
    Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools -Restart

    # Verify installation on both nodes
    Invoke-Command -ComputerName SRV-FILE01, SRV-DC02 -ScriptBlock {
        Get-WindowsFeature Failover-Clustering | Select-Object Name, Installed
    }
    ```

### Partie 3 : Valider la configuration

??? success "Solution"

    ```powershell
    # Run cluster validation (from either node)
    Test-Cluster -Node SRV-FILE01, SRV-DC02 -ReportName "C:\Temp\ClusterValidation"

    # Review the report
    # The report is saved as an HTML file in C:\Temp\
    # Check for warnings and errors
    ```

!!! warning "Attention"

    La validation du cluster est une etape obligatoire. Examinez attentivement les
    avertissements et erreurs dans le rapport. Certains avertissements sont acceptables
    en lab (ex. : un seul reseau) mais pas en production.

### Partie 4 : Creer le cluster

??? success "Solution"

    ```powershell
    # Create the cluster
    New-Cluster -Name "CLUSTER-LAB" `
        -Node SRV-FILE01, SRV-DC02 `
        -StaticAddress 192.168.10.50 `
        -NoStorage

    # Add the shared disk to the cluster
    Get-ClusterAvailableDisk | Add-ClusterDisk

    # Verify cluster status
    Get-Cluster | Select-Object Name, Domain
    Get-ClusterNode | Select-Object Name, State
    Get-ClusterResource | Select-Object Name, State, OwnerNode, ResourceType
    ```

### Partie 5 : Ajouter un role de serveur de fichiers

??? success "Solution"

    ```powershell
    # Add File Server role to the cluster
    Add-ClusterFileServerRole -Name "FS-CLUSTER" `
        -Storage "Cluster Disk 1" `
        -StaticAddress 192.168.10.51

    # Create a shared folder on the cluster
    New-Item -Path "S:\Shares\Donnees" -ItemType Directory -Force
    New-SmbShare -Name "Donnees" -Path "S:\Shares\Donnees" `
        -FullAccess "WINOPSLAB\Domain Users" `
        -ScopeName "FS-CLUSTER"

    # Verify the clustered file share
    Get-SmbShare -ScopeName "FS-CLUSTER"
    ```

### Partie 6 : Tester le basculement

??? success "Solution"

    ```powershell
    # Check current owner
    Get-ClusterGroup "FS-CLUSTER" | Select-Object Name, OwnerNode, State

    # Move the role to the other node (planned failover)
    Move-ClusterGroup "FS-CLUSTER" -Node "SRV-DC02"

    # Verify the move
    Get-ClusterGroup "FS-CLUSTER" | Select-Object Name, OwnerNode, State

    # Test from CLI-W11: access the share during failover
    # \\FS-CLUSTER\Donnees should remain accessible

    # Simulate an unplanned failover (stop the active node)
    # On the Hyper-V HOST:
    Stop-VM -Name "SRV-DC02" -Force

    # After a few seconds, the role should fail over to SRV-FILE01
    # Verify from the remaining node:
    Get-ClusterGroup "FS-CLUSTER" | Select-Object Name, OwnerNode, State
    ```

## Verification

!!! question "Questions de validation"

    1. Quel est le nombre minimum de noeuds pour un cluster de basculement ?
    2. Qu'est-ce que le quorum et pourquoi est-il important ?
    3. Quelle est la difference entre un basculement planifie et non planifie ?
    4. Pourquoi faut-il valider le cluster avant de le creer ?

??? success "Reponses"

    1. Le minimum est **2 noeuds**. Cependant, pour un quorum optimal, **3 noeuds** ou
       un temoin (disque ou partage de fichiers) est recommande.
    2. Le quorum determine combien de defaillances le cluster peut tolerer. Il fonctionne
       par vote : la majorite des votes doit etre disponible pour que le cluster reste en ligne.
    3. **Planifie** : l'administrateur deplace le role (`Move-ClusterGroup`), pas de perte de
       connexion. **Non planifie** : un noeud tombe, le cluster detecte la panne et bascule
       automatiquement (quelques secondes d'indisponibilite).
    4. La validation detecte les problemes de configuration (reseau, stockage, DNS) avant la
       creation. Elle est obligatoire pour le support Microsoft.

## Nettoyage

```powershell
# Remove the cluster
Remove-Cluster -Name "CLUSTER-LAB" -Force -CleanupAD

# Uninstall Failover Clustering feature (optional)
Uninstall-WindowsFeature -Name Failover-Clustering
```

## Prochaine etape

:material-arrow-right: [Lab 08 : Infrastructure PKI](lab-08-pki.md)
