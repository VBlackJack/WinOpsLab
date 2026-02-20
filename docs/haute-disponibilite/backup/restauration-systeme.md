---
title: "Restauration systeme"
description: "Restaurer un serveur Windows Server 2022 : Bare Metal Recovery, restauration de l'etat systeme, WinRE et procedures de recuperation."
tags:
  - haute-disponibilite
  - backup
  - restauration
  - bare-metal-recovery
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

# Restauration systeme

<span class="level-advanced">Avance</span> Â· Temps estime : 50 minutes

## Introduction

Savoir sauvegarder est essentiel, mais savoir **restaurer** est critique. Ce chapitre couvre les differentes methodes de restauration sous Windows Server 2022, depuis la restauration de fichiers individuels jusqu'a la reconstruction complete d'un serveur (Bare Metal Recovery).

!!! example "Analogie"

    La restauration systeme, c'est comme un service de sinistres pour une maison. Selon l'ampleur du degat, on intervient differemment : pour une vitre cassee (fichier supprime), un vitrier suffit ; pour un incendie partiel (systeme corrompu), on fait appel a des specialistes pour restaurer les pieces touchees ; pour une maison entierement detruite (serveur inutilisable), il faut tout reconstruire a partir des plans d'architecte et des fondations. Chaque niveau d'intervention necessite des outils et des procedures differents.

## Types de restauration

```mermaid
flowchart TD
    A[Quel type de<br/>restauration ?] --> B{Serveur demarre ?}
    B -- Oui --> C{Quoi restaurer ?}
    B -- Non --> D[Bare Metal Recovery<br/>via WinRE]
    C --> E[Fichiers/dossiers]
    C --> F[Etat systeme]
    C --> G[Volumes complets]
    E --> E1[Restauration via<br/>WSB GUI ou wbadmin]
    F --> F1[wbadmin start<br/>systemstaterecovery]
    G --> G1[wbadmin start recovery<br/>-itemType:Volume]
```

## Restauration de fichiers et dossiers

La restauration la plus courante : recuperer des fichiers supprimes ou corrompus.

### Via PowerShell (wbadmin)

```powershell
# List available backup versions
wbadmin get versions

# Restore specific files from the latest backup
wbadmin start recovery `
    -version:01/15/2025-21:00 `
    -itemType:File `
    -items:"D:\Data\ImportantFile.xlsx" `
    -recoveryTarget:"D:\Restored" `
    -quiet
```

Resultat :

```text
wbadmin 1.0 - Backup command-line tool
(C) Copyright Microsoft Corporation. All rights reserved.

Backup time: 20/02/2026 22:00:00, Backup target: Disk labeled 'BackupDisk'
Version identifier: 02/20/2026-22:00
Can recover: Volume(s), File(s), Application(s), Bare Metal Recovery, System State

Retrieving volume information...
Starting recovery of D:\Data\ImportantFile.xlsx to D:\Restored ...
Successfully recovered D:\Data\ImportantFile.xlsx to D:\Restored.
The recovery operation completed successfully.
```

### Via l'interface graphique

1. Ouvrir **Windows Server Backup** (`wbadmin.msc`)
2. Cliquer sur **Recover** dans le panneau Actions
3. Selectionner la source : serveur local ou autre emplacement
4. Choisir la date de la sauvegarde
5. Selectionner **Files and folders**
6. Naviguer dans l'arborescence et selectionner les elements
7. Choisir la destination : emplacement d'origine ou repertoire alternatif

!!! tip "Gestion des conflits"

    Lors de la restauration vers l'emplacement d'origine, WSB propose de : ecraser la version existante, creer une copie, ou ignorer le fichier. En cas de doute, restaurez vers un emplacement alternatif.

## Restauration de l'etat systeme

La restauration de l'etat systeme est critique pour les controleurs de domaine. Elle restaure :

- La base Active Directory (NTDS.dit)
- Le registre Windows
- Le dossier SYSVOL
- Les fichiers de demarrage
- Le magasin de certificats

### Restauration non-authoritative (par defaut)

La restauration **non-authoritative** restaure les donnees AD a leur etat au moment de la sauvegarde. Ensuite, la replication AD apporte les modifications recentes depuis les autres DC.

```powershell
# System state restore (requires DSRM mode for domain controllers)
wbadmin start systemstaterecovery `
    -version:01/15/2025-21:00 `
    -quiet
```

!!! warning "Mode DSRM pour les DC"

    Sur un controleur de domaine, la restauration de l'etat systeme doit etre effectuee en **mode DSRM** (Directory Services Restore Mode). Redemarrez le serveur en mode DSRM via `bcdedit /set safeboot dsrepair` puis redemarrez.

```powershell
# Boot into DSRM mode (requires reboot)
bcdedit /set safeboot dsrepair
Restart-Computer -Force

# After system state restore, return to normal boot
bcdedit /deletevalue safeboot
Restart-Computer -Force
```

### Restauration authoritative

La restauration **authoritative** force la replication des donnees restaurees vers tous les autres DC. Utilisee pour recuperer des objets supprimes de maniere definitive.

```powershell
# Step 1: Boot into DSRM and restore system state (non-authoritative)
wbadmin start systemstaterecovery -version:01/15/2025-21:00 -quiet

# Step 2: Mark specific objects as authoritative using ntdsutil
ntdsutil
# > activate instance ntds
# > authoritative restore
# > restore subtree "OU=Users,DC=yourdomain,DC=local"
# > quit
# > quit

# Step 3: Reboot normally
bcdedit /deletevalue safeboot
Restart-Computer -Force
```

!!! danger "Utilisation de la restauration authoritative"

    La restauration authoritative ne doit etre utilisee que dans des situations specifiques (suppression massive d'objets AD). Elle ecrase les modifications recentes sur les autres DC pour les objets concernes. Privilegiez la corbeille AD pour les suppressions recentes.

## Bare Metal Recovery (BMR)

Le BMR permet de restaurer un serveur **a partir de zero**, sur un materiel vierge ou un nouveau materiel (en general compatible).

### Prerequis

- Support d'installation Windows Server 2022 (ISO/USB)
- Acces a la sauvegarde BMR (disque local ou partage reseau)
- Pilotes de stockage si le nouveau materiel differe de l'original

### Procedure de restauration BMR

```mermaid
flowchart TD
    A[Demarrer sur le media<br/>d'installation Windows Server] --> B[Choisir la langue et le clavier]
    B --> C["Cliquer sur 'Repair your computer'"]
    C --> D[Troubleshoot]
    D --> E[System Image Recovery]
    E --> F{Source de la sauvegarde ?}
    F -- Disque local --> G[Selection automatique<br/>du dernier backup]
    F -- Reseau --> H[Specifier le chemin UNC<br/>et les identifiants]
    G --> I[Confirmer les options<br/>de restauration]
    H --> I
    I --> J{Formater et<br/>repartitionner ?}
    J -- Oui --> K[Restauration complete]
    J -- Non --> L[Conservation du<br/>partitionnement existant]
    K --> M[Redemarrage du serveur]
    L --> M
```

### Etapes detaillees

1. **Demarrer sur le media d'installation** Windows Server 2022
2. Ecran de selection de la langue : cliquer sur **Suivant**
3. Cliquer sur **Repair your computer** (en bas a gauche)
4. Choisir **Troubleshoot** > **System Image Recovery**
5. Selectionner le systeme d'exploitation cible
6. L'assistant detecte automatiquement les sauvegardes disponibles
7. Selectionner le jeu de sauvegarde a restaurer
8. Options avancees :
    - **Format and repartition disks** : recree les partitions (recommande sur nouveau materiel)
    - **Exclude disks** : exclure des disques de la restauration
9. Confirmer et lancer la restauration
10. Redemarrer a la fin du processus

!!! warning "Materiel different"

    Si le materiel cible differe de l'original (controleur de stockage different), il peut etre necessaire d'injecter des pilotes via l'option **Install drivers** dans WinRE avant la restauration.

## Windows Recovery Environment (WinRE)

WinRE est l'environnement de recuperation integre a Windows. Il demarre automatiquement apres plusieurs echecs de demarrage ou peut etre lance manuellement.

### Acceder a WinRE

```powershell
# Check if WinRE is configured
reagentc /info

# Boot into WinRE (requires reboot)
reagentc /boottore
Restart-Computer -Force
```

Resultat :

```text
Windows Recovery Environment (Windows RE) and system reset configuration
Information:

    Windows RE status:         Enabled
    Windows RE location:       \\?\GLOBALROOT\device\harddisk0\partition4\Recovery\WindowsRE
    Boot Configuration Data (BCD) identifier: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    Recovery image location:
    Recovery image index:      0
    Custom image location:
    Custom image index:        0
```

### Outils disponibles dans WinRE

| Outil | Description |
|---|---|
| **System Image Recovery** | Restauration BMR |
| **Startup Repair** | Reparation automatique du demarrage |
| **Command Prompt** | Acces a la ligne de commande |
| **UEFI Firmware Settings** | Acces au BIOS/UEFI |
| **Uninstall Updates** | Desinstaller des mises a jour problematiques |

### Reparation du demarrage via la ligne de commande

```powershell
# Rebuild BCD (Boot Configuration Data)
bootrec /rebuildbcd

# Fix the MBR
bootrec /fixmbr

# Fix the boot sector
bootrec /fixboot

# Scan for Windows installations
bootrec /scanos

# Check and repair system files
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows

# Check disk integrity
chkdsk C: /r
```

## Restauration a partir d'un autre serveur

Si le serveur d'origine n'est plus disponible, il est possible de restaurer des fichiers depuis un catalogue de sauvegarde sur un autre serveur.

```powershell
# Import a backup catalog from an external disk
wbadmin restore catalog -backupTarget:E: -quiet

# List available backups from the imported catalog
wbadmin get versions -backupTarget:E:

# Restore files from the imported backup
wbadmin start recovery `
    -version:01/15/2025-21:00 `
    -backupTarget:E: `
    -itemType:File `
    -items:"D:\Data" `
    -recoveryTarget:"C:\RestoredData" `
    -quiet
```

## Checklist de restauration

Avant de restaurer :

- [ ] Identifier le type de restauration necessaire (fichier, system state, BMR)
- [ ] Verifier la disponibilite et l'integrite du jeu de sauvegarde
- [ ] Documenter l'etat actuel du serveur avant la restauration
- [ ] Prevenir les utilisateurs de l'interruption de service
- [ ] Preparer le media d'installation Windows Server (pour BMR)
- [ ] Verifier la compatibilite materielle (pour BMR sur nouveau materiel)

Apres la restauration :

- [ ] Verifier le fonctionnement du systeme d'exploitation
- [ ] Controler les services critiques
- [ ] Verifier la replication AD (si controleur de domaine)
- [ ] Appliquer les mises a jour manquantes
- [ ] Documenter l'operation effectuee

!!! example "Scenario pratique"

    **Contexte :** Valerie administre `DC-01` (controleur de domaine unique du domaine `lab.local`). Un matin, elle est alertee : apres une mise a jour nocturne, le serveur ne demarre plus et affiche un ecran bleu. Les utilisateurs ne peuvent plus s'authentifier ni acceder aux ressources du domaine.

    **Probleme :** DC-01 ne demarre pas. La derniere sauvegarde BMR date de la veille a 22h00, stockee sur un disque USB externe (`BackupDisk`).

    **Solution :** Valerie procede a une restauration BMR depuis le media d'installation Windows Server.

    **Etape 1 :** Demarrer depuis le DVD/USB d'installation Windows Server 2022
    - Ecran de selection de la langue : cliquer **Suivant**
    - Cliquer **Repair your computer** (bas gauche)
    - Choisir **Troubleshoot** > **System Image Recovery**

    **Etape 2 :** L'assistant detecte la sauvegarde sur `BackupDisk` :

    ```text
    Select a system image backup
    The following system image was automatically selected because it is the latest available:
      Location  : Disk labeled 'BackupDisk'
      Date       : 20/02/2026, 22:47:11
      Computer   : DC-01
    ```

    **Etape 3 :** Valerie confirme les options (Format and repartition disks : Non, car le disque systeme est intact) et lance la restauration.

    ```text
    Restoring disk...
    Windows is restoring your computer. This may take a while.
    Estimated time: 47 minutes.
    [===========>          ] 55% complete
    ```

    **Etape 4 :** Apres redemarrage, Valerie verifie les services AD :

    ```powershell
    # Verify AD DS service
    Get-Service ADWS, NTDS, Netlogon, DNS | Format-Table Name, Status

    # Check replication
    repadmin /showrepl
    ```

    ```text
    Name     Status
    ----     ------
    ADWS     Running
    NTDS     Running
    Netlogon Running
    DNS      Running

    Replication Summary Start Time: 2026-02-20 10:15:32
    ...
    Source DSA          Last Success Time    Delta
    -------             -----------------    -----
    DC-01               2026-02-20 10:15:30  0d.00h:00m:30s (no error)
    ```

    La restauration complete a pris 58 minutes â€” en dessous du RTO de 2 heures defini avec la direction.

!!! danger "Erreurs courantes"

    **Restauration system state DC sans mode DSRM** â€” Sur un controleur de domaine, tenter une restauration de l'etat systeme depuis Windows normal (pas DSRM) echoue car le service NTDS est en cours d'utilisation. Redemarrez toujours en mode DSRM (`bcdedit /set safeboot dsrepair`) avant la restauration.

    **Oublier de desactiver le mode DSRM apres restauration** â€” Apres la restauration, si on oublie `bcdedit /deletevalue safeboot`, le serveur redemarre en mode DSRM a chaque fois et ne rejoint pas le domaine normalement.

    **Restauration authoritative par erreur** â€” La restauration authoritative est irreversible sur le plan de la replication : elle ecrase les donnees des autres DC pour les objets concernes. Ne l'utilisez que si des objets ont ete deliberement supprimes et ne peuvent pas etre recuperes via la corbeille AD.

    **Materiel incompatible lors du BMR** â€” Si le nouveau serveur a un controleur de stockage different, Windows peut ne pas trouver les disques apres restauration. Injectez les pilotes depuis WinRE avant de lancer la restauration (`Troubleshoot > Advanced Options > Install drivers`).

    **Sauvegarde reseau inaccessible lors du BMR** â€” WinRE doit pouvoir acceder au partage reseau pendant la restauration. Verifiez que les identifiants de connexion (stockes a part, pas seulement dans WSB) sont disponibles et que le reseau est configure dans WinRE.

## Points cles a retenir

- La restauration de fichiers est la plus simple et la plus courante
- La restauration de l'etat systeme sur un DC necessite le mode **DSRM**
- La restauration **authoritative** force la replication depuis le DC restaure (a utiliser avec precaution)
- Le **BMR** permet la restauration complete sur un serveur vierge
- WinRE est l'environnement de recuperation accessible meme si Windows ne demarre plus
- Testez vos procedures de restauration **avant** d'en avoir besoin

## Pour aller plus loin

- Windows Server Backup : [Windows Server Backup](windows-server-backup.md)
- Corbeille AD : [AD Recycle Bin](ad-recycle-bin.md)
- Strategie de sauvegarde : [Strategie de sauvegarde](strategie-sauvegarde.md)

