---
title: "Windows Server Backup"
description: "Installer et utiliser Windows Server Backup sous Windows Server 2022 : sauvegardes completes, incrementales, bare metal recovery et planification."
tags:
  - haute-disponibilite
  - backup
  - windows-server-backup
  - windows-server
---
<!--
  Copyright 2026 Julien Bombled

  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at

      http://www.apache.org/licenses/LICENSE-2.0

  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License.
-->

# Windows Server Backup

<span class="level-advanced">Avance</span> Â· Temps estime : 45 minutes

## Introduction

**Windows Server Backup** (WSB) est la fonctionnalite de sauvegarde integree a Windows Server. Bien qu'elle ne remplace pas les solutions de sauvegarde d'entreprise (Veeam, Commvault, etc.), WSB est un outil fiable pour les environnements de petite et moyenne taille et pour les sauvegardes de l'etat systeme.

WSB utilise la technologie **VSS** (Volume Shadow Copy Service) pour creer des instantanes coherents, meme lorsque les fichiers sont en cours d'utilisation.

!!! example "Analogie"

    Windows Server Backup, c'est comme la boite a outils fournie avec une voiture : un cric, une roue de secours, un triangle de signalisation. C'est suffisant pour depanner en cas de crevaison sur le bord de la route. Mais si vous etes chauffeur professionnel avec 20 vehicules, vous aurez besoin d'un garage equipe (Veeam, Azure Backup...). WSB est l'outil de secours integre, fiable pour les situations courantes, mais pas un remplacement pour une solution d'entreprise.

## Installation

WSB n'est pas installe par defaut. Il doit etre ajoute comme fonctionnalite.

```powershell
# Install Windows Server Backup feature
Install-WindowsFeature -Name Windows-Server-Backup -IncludeManagementTools

# Verify installation
Get-WindowsFeature -Name Windows-Server-Backup
```

Resultat :

```text
Display Name                       Name                   Install State
------------                       ----                   -------------
[X] Windows Server Backup          Windows-Server-Backup  Installed
```

Apres l'installation, l'outil est accessible via :

- **GUI** : `wbadmin.msc` ou Server Manager > Tools > Windows Server Backup
- **CLI** : cmdlet `wbadmin` ou module PowerShell `WindowsServerBackup`

## Types de sauvegarde

| Type | Description | Espace requis | Temps |
|---|---|---|---|
| **Full (complete)** | Sauvegarde l'integralite des donnees selectionnees | Eleve | Long |
| **Incremental** | Sauvegarde uniquement les modifications depuis la derniere sauvegarde | Reduit | Court |
| **System State** | AD, registre, fichiers systeme, boot, SYSVOL | Moyen | Moyen |
| **Bare Metal Recovery** | Tout le necessaire pour restaurer sur un serveur vierge | Eleve | Long |

!!! tip "WSB et les incrementales"

    Windows Server Backup utilise par defaut des sauvegardes incrementales au niveau des blocs. Meme lorsque vous planifiez une sauvegarde "complete", WSB optimise le stockage en ne copiant que les blocs modifies.

## Sauvegarde via PowerShell

### Sauvegarde complete du serveur

```powershell
# Define the backup policy
$policy = New-WBPolicy

# Add all volumes (full server backup)
$volumes = Get-WBVolume -AllVolumes
Add-WBVolume -Policy $policy -Volume $volumes

# Include bare metal recovery
Add-WBBareMetalRecovery -Policy $policy

# Include system state
Add-WBSystemState -Policy $policy

# Set backup target (dedicated disk)
$targetDisk = Get-WBDisk | Where-Object { $_.DiskNumber -eq 1 }
$target = New-WBBackupTarget -Disk $targetDisk -Label "BackupDisk"
Add-WBBackupTarget -Policy $policy -Target $target

# Execute the backup
Start-WBBackup -Policy $policy
```

### Sauvegarde vers un partage reseau

```powershell
$policy = New-WBPolicy

# Add bare metal recovery
Add-WBBareMetalRecovery -Policy $policy
Add-WBSystemState -Policy $policy

# Configure network share as target
$cred = Get-Credential -Message "Credentials for backup share"
$target = New-WBBackupTarget -NetworkPath "\\YOURBACKUPSERVER\Backups" -Credential $cred
Add-WBBackupTarget -Policy $policy -Target $target

# Run backup
Start-WBBackup -Policy $policy
```

!!! warning "Sauvegarde reseau"

    Lors d'une sauvegarde vers un partage reseau, WSB ne conserve qu'**une seule version**. Chaque nouvelle sauvegarde ecrase la precedente. Pour conserver un historique, utilisez un disque local dedie.

### Sauvegarde de volumes specifiques

```powershell
$policy = New-WBPolicy

# Add only specific volumes
$volumeC = Get-WBVolume -VolumePath "C:"
$volumeD = Get-WBVolume -VolumePath "D:"
Add-WBVolume -Policy $policy -Volume $volumeC, $volumeD

# Target: dedicated backup disk
$target = New-WBBackupTarget -Disk (Get-WBDisk | Where-Object { $_.DiskNumber -eq 2 })
Add-WBBackupTarget -Policy $policy -Target $target

Start-WBBackup -Policy $policy
```

### Sauvegarde de l'etat systeme uniquement

```powershell
# Quick system state backup (AD, registry, boot files)
wbadmin start systemstatebackup -backupTarget:E: -quiet
```

## Sauvegarde via l'interface graphique

1. Ouvrir **Windows Server Backup** (`wbadmin.msc`)
2. Dans le panneau Actions, cliquer sur **Backup Once** ou **Backup Schedule**
3. Choisir les options :
    - **Full server** : tout le serveur (recommande)
    - **Custom** : selection de volumes, fichiers ou system state
4. Selectionner la destination de sauvegarde
5. Confirmer et demarrer

## Planification des sauvegardes

### Via PowerShell

```powershell
$policy = New-WBPolicy

# Add backup content
Add-WBBareMetalRecovery -Policy $policy
Add-WBSystemState -Policy $policy

# Configure backup target
$target = New-WBBackupTarget -Disk (Get-WBDisk | Where-Object { $_.DiskNumber -eq 1 })
Add-WBBackupTarget -Policy $policy -Target $target

# Schedule daily backup at 9:00 PM
Set-WBSchedule -Policy $policy -Schedule 21:00

# Apply the scheduled policy
Set-WBPolicy -Policy $policy

# Verify the scheduled policy
Get-WBPolicy
```

### Multiples horaires

```powershell
# Schedule backup twice daily
Set-WBSchedule -Policy $policy -Schedule 06:00, 18:00
```

### Verification de la planification

```powershell
# View current backup schedule
$currentPolicy = Get-WBPolicy
$currentPolicy | Format-List Schedule, BackupTargets

# View last backup status
Get-WBSummary

# View backup history
Get-WBBackupSet
```

## Bare Metal Recovery (BMR)

La sauvegarde BMR inclut tout le necessaire pour restaurer un serveur a partir de zero :

- Volumes critiques (volume systeme, volume de demarrage)
- Etat systeme (registre, Active Directory si DC, fichiers de boot)
- Configuration de demarrage

```powershell
# Verify BMR backup is included
$policy = Get-WBPolicy
$policy.BMR  # Should return True
```

```mermaid
flowchart LR
    A[Sauvegarde BMR] --> B[Volume systeme C:]
    A --> C[Partition de demarrage]
    A --> D[Etat systeme]
    A --> E[Configuration BCD]
    B --> F["Restauration complete<br/>sur serveur vierge"]
    C --> F
    D --> F
    E --> F
```

!!! danger "Testez vos restaurations"

    Une sauvegarde non testee est une sauvegarde non fiable. Planifiez des tests de restauration BMR reguliers dans un environnement isole pour valider l'integrite de vos sauvegardes.

## Surveillance et diagnostic

```powershell
# View last backup summary
Get-WBSummary | Format-List LastSuccessfulBackupTime, LastBackupResultHR, LastBackupResultDetailedHR

# View detailed backup job status
Get-WBJob -Previous 5

# Check Windows Server Backup event logs
Get-WinEvent -LogName "Microsoft-Windows-Backup" -MaxEvents 20 | Format-Table TimeCreated, Message -Wrap
```

Resultat :

```text
LastSuccessfulBackupTime      : 20/02/2026 22:00:17
LastBackupResultHR            : 0
LastBackupResultDetailedHR    : 0

Id  StartTime             EndTime               Result
--  ---------             -------               ------
 5  20/02/2026 22:00:00   20/02/2026 22:47:13   Success
 4  19/02/2026 22:00:01   19/02/2026 22:51:42   Success
 3  18/02/2026 22:00:00   18/02/2026 22:48:55   Success
 2  17/02/2026 22:00:01   17/02/2026 23:12:08   Warning
 1  16/02/2026 22:00:00   16/02/2026 22:49:21   Success

TimeCreated             Message
-----------             -------
20/02/2026 22:47:13     The backup operation completed successfully.
20/02/2026 22:00:01     Backup started with destination backup disk 'BackupDisk'.
19/02/2026 22:51:42     The backup operation completed successfully.
```

## Limitations de WSB

| Limitation | Alternative |
|---|---|
| Une seule version en sauvegarde reseau | Utiliser un disque local dedie |
| Pas de granularite fichier en planifie | Utiliser un logiciel tiers |
| Pas de deduplication native | Solutions tierces ou Azure Backup |
| Pas de sauvegarde sur bande | Solutions d'entreprise (Veeam, etc.) |
| Pas de chiffrement natif | BitLocker sur le volume de sauvegarde |

!!! example "Scenario pratique"

    **Contexte :** Pierre administre `DC-01`, le seul controleur de domaine d'une PME. Il doit mettre en place une sauvegarde quotidienne de l'etat systeme pour pouvoir restaurer Active Directory en cas de corruption ou de suppression accidentelle d'objets critiques.

    **Probleme :** DC-01 n'a pas de disque de sauvegarde dedie. Pierre dispose d'un partage reseau sur `SRV-01` (`\\SRV-01\Backups`) et d'un disque externe USB branche le weekend par le responsable informatique.

    **Solution :** Pierre configure une sauvegarde quotidienne de l'etat systeme vers le partage reseau :

    ```powershell
    # Step 1: Install WSB if not already present
    Install-WindowsFeature -Name Windows-Server-Backup -IncludeManagementTools
    ```

    ```text
    Success Restart Needed Exit Code Feature Result
    ------- -------------- --------- --------------
    True    No             Success   {Windows Server Backup}
    ```

    ```powershell
    # Step 2: Configure the backup policy
    $policy = New-WBPolicy
    Add-WBSystemState -Policy $policy
    Add-WBBareMetalRecovery -Policy $policy

    $cred = Get-Credential -Message "Credentials for \\SRV-01\Backups"
    $target = New-WBBackupTarget -NetworkPath "\\SRV-01\Backups" -Credential $cred
    Add-WBBackupTarget -Policy $policy -Target $target

    Set-WBSchedule -Policy $policy -Schedule 23:00
    Set-WBPolicy -Policy $policy

    # Step 3: Trigger a manual backup immediately to test
    Start-WBBackup -Policy $policy
    ```

    ```text
    Retrieving volume information...
    Starting a backup of volumes: Windows (C:) ...
    Backup of volume Windows (C:) completed successfully.
    Summary of backup:
      Backup of DC-01 completed successfully.
      Backup time: 02/20/2026 23:15:42 - 02/20/2026 23:47:11
      Backup destination: Network share
      Status: Completed successfully
    ```

    ```powershell
    # Step 4: Verify the backup
    Get-WBSummary
    ```

    ```text
    LastSuccessfulBackupTime : 20/02/2026 23:47:11
    LastBackupResultHR       : 0
    NextBackupTime           : 21/02/2026 23:00:00
    NumberOfDisks            : 0
    NumberOfNetworkPaths     : 1
    ```

    Pierre note dans son runbook que la sauvegarde reseau n'offre qu'une seule version. Chaque nuit, la sauvegarde precedente est ecrasee. Pour une retention de 7 jours, il devra passer a un disque local dedie ou a Azure Backup.

!!! danger "Erreurs courantes"

    **Sauvegarde reseau avec une seule version** â€” Lors de la sauvegarde vers un partage reseau (`\\serveur\partage`), WSB ecrase la version precedente a chaque execution. Il n'y a pas de retention automatique. Pour garder un historique, utilisez un disque local dedie ou une solution tierce.

    **Oublier `Add-WBBareMetalRecovery`** â€” Sans BMR, la sauvegarde ne peut pas etre utilisee pour une restauration a partir de zero. Sur un controleur de domaine, incluez toujours `Add-WBBareMetalRecovery` et `Add-WBSystemState`.

    **Sauvegarde systeme sur le meme volume** â€” Windows Server Backup ne permet pas de sauvegarder le volume systeme sur lui-meme. Le disque cible doit etre different du disque systeme.

    **Ne pas verifier le resultat apres la premiere planification** â€” Apres avoir configure `Set-WBPolicy`, verifiez avec `Get-WBPolicy` et attendez la premiere execution planifiee pour confirmer le succes via `Get-WBSummary`.

    **WSB pour des besoins d'entreprise** â€” WSB ne supporte pas la deduplication, le chiffrement natif, la sauvegarde sur bande, ni la granularite applicative (ex. elements Exchange). Pour ces besoins, evaluez Veeam Backup & Replication ou Azure Backup.

## Points cles a retenir

- WSB est un outil fiable pour les environnements de petite taille et les sauvegardes system state
- Toujours inclure **Bare Metal Recovery** dans les sauvegardes planifiees
- Les sauvegardes vers un partage reseau n'offrent qu'**une seule version**
- La planification se fait via `Set-WBSchedule` et `Set-WBPolicy`
- Testez regulierement vos restaurations dans un environnement de test
- Pour des besoins avances (deduplication, chiffrement, bandes), envisagez une solution tierce

## Pour aller plus loin

- Strategie de sauvegarde : [Strategie de sauvegarde](strategie-sauvegarde.md)
- Restauration systeme : [Restauration systeme](restauration-systeme.md)
- Documentation Microsoft : Windows Server Backup Feature Overview

