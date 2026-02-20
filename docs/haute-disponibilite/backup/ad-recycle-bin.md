---
title: "Corbeille Active Directory"
description: "Activer et utiliser la corbeille Active Directory sous Windows Server 2022 : niveaux fonctionnels, restauration d'objets supprimes, tombstone et recycled."
tags:
  - haute-disponibilite
  - backup
  - active-directory
  - recycle-bin
  - windows-server
---

# Corbeille Active Directory

<span class="level-advanced">Avance</span> · Temps estime : 35 minutes

## Introduction

La **corbeille Active Directory** (AD Recycle Bin) est une fonctionnalite qui permet de restaurer des objets AD supprimes (utilisateurs, groupes, OU) sans avoir besoin d'une restauration authoritative ni d'un temps d'arret. Disponible depuis Windows Server 2008 R2, elle est devenue un indispensable de l'administration AD.

!!! warning "Prerequis critique"

    La corbeille AD necessite un **niveau fonctionnel de foret** minimum de Windows Server 2008 R2. Verifiez le niveau fonctionnel avant d'activer la fonctionnalite.

## Cycle de vie d'un objet AD supprime

### Sans la corbeille AD

Sans la corbeille, un objet supprime passe par l'etat **tombstone** : la plupart de ses attributs sont effaces, ne laissant qu'un objet minimal pendant la duree de vie du tombstone (180 jours par defaut).

### Avec la corbeille AD

Avec la corbeille, un objet supprime conserve **tous ses attributs** pendant une duree configurable, permettant une restauration complete.

```mermaid
stateDiagram-v2
    [*] --> Actif : Objet cree
    Actif --> Supprime : Suppression

    state "Sans Corbeille AD" as SC {
        Supprime --> Tombstone : Attributs effaces
        Tombstone --> PurgeDefinitive : Expiration (180 jours)
    }

    state "Avec Corbeille AD" as AC {
        Supprime --> Recycle : Attributs conserves
        Recycle --> RecycledState : Expiration deleted object lifetime
        RecycledState --> PurgeDefinitive2 : Expiration recycled object lifetime
    }

    PurgeDefinitive --> [*] : Objet detruit
    PurgeDefinitive2 --> [*] : Objet detruit
```

### Durees de retention

| Phase | Duree par defaut | Attributs conserves |
|---|---|---|
| **Deleted** (corbeille) | 180 jours | Tous les attributs |
| **Recycled** | 180 jours | Attributs minimaux (comme tombstone) |
| **Purge definitive** | - | Objet completement supprime |

```powershell
# Check the deleted object lifetime (days in Deleted state)
(Get-ADObject "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=yourdomain,DC=local" `
    -Properties "msDS-DeletedObjectLifetime")."msDS-DeletedObjectLifetime"

# Check tombstone lifetime
(Get-ADObject "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=yourdomain,DC=local" `
    -Properties tombstoneLifetime).tombstoneLifetime
```

## Verification du niveau fonctionnel

```powershell
# Check forest functional level
(Get-ADForest).ForestMode

# Check domain functional level
(Get-ADDomain).DomainMode
```

Le niveau fonctionnel minimum requis est **Windows2008R2Forest** pour la foret.

!!! danger "Elevation du niveau fonctionnel"

    L'elevation du niveau fonctionnel de la foret est une operation **irreversible**. Assurez-vous que TOUS les controleurs de domaine de la foret executent au minimum Windows Server 2008 R2 avant de proceder.

```powershell
# Raise forest functional level (IRREVERSIBLE)
Set-ADForestMode -Identity "yourdomain.local" -ForestMode Windows2016Forest -Confirm
```

## Activer la corbeille AD

### Via PowerShell

```powershell
# Enable AD Recycle Bin (requires Enterprise Admin)
Enable-ADOptionalFeature `
    -Identity "Recycle Bin Feature" `
    -Scope ForestOrConfigurationSet `
    -Target "yourdomain.local" `
    -Confirm

# Verify activation
Get-ADOptionalFeature -Filter 'Name -like "Recycle*"' | Format-List Name, EnabledScopes
```

### Via Active Directory Administrative Center (ADAC)

1. Ouvrir **Active Directory Administrative Center** (`dsac.exe`)
2. Selectionner le domaine dans le panneau de navigation
3. Dans le panneau Tasks, cliquer sur **Enable Recycle Bin**
4. Confirmer l'activation
5. Actualiser la console (F5)

!!! warning "Activation irreversible"

    Une fois activee, la corbeille AD ne peut **pas etre desactivee**. Cette operation est definitive. Cela ne pose generalement aucun probleme car la fonctionnalite n'a pas d'effets negatifs.

## Restaurer des objets supprimes

### Via PowerShell

```powershell
# List all deleted objects
Get-ADObject -Filter 'IsDeleted -eq $true' -IncludeDeletedObjects -Properties * |
    Select-Object Name, ObjectClass, WhenChanged, IsDeleted |
    Sort-Object WhenChanged -Descending

# Find a specific deleted user
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*jdupont*"' `
    -IncludeDeletedObjects -Properties *

# Restore a deleted user
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*jdupont*"' `
    -IncludeDeletedObjects | Restore-ADObject

# Restore a deleted user to a different OU
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*jdupont*"' `
    -IncludeDeletedObjects | Restore-ADObject -TargetPath "OU=Users,DC=yourdomain,DC=local"
```

### Restaurer une OU et son contenu

La restauration d'une OU supprimee doit se faire dans l'ordre : d'abord l'OU parent, puis son contenu.

```powershell
# Step 1: Restore the OU itself
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*OU-Marketing*" -and ObjectClass -eq "organizationalUnit"' `
    -IncludeDeletedObjects | Restore-ADObject

# Step 2: Restore all objects that were in the OU
Get-ADObject -Filter 'IsDeleted -eq $true -and LastKnownParent -like "*OU=OU-Marketing*"' `
    -IncludeDeletedObjects | Restore-ADObject
```

!!! tip "Ordre de restauration"

    Si l'OU parente a ete supprimee avec tout son contenu, restaurez d'abord le conteneur (OU), puis les objets enfants. Sinon, les objets enfants n'auront plus de conteneur cible.

### Via Active Directory Administrative Center

1. Ouvrir **ADAC** (`dsac.exe`)
2. Cliquer sur le conteneur **Deleted Objects** dans le domaine
3. Rechercher l'objet supprime
4. Clic droit > **Restore** (restaurer a l'emplacement d'origine) ou **Restore To** (choisir un emplacement)

## Restauration avancee

### Restaurer un groupe avec ses membres

Lorsqu'un groupe est supprime, les liens de membership ne sont pas automatiquement restaures.

```powershell
# Restore the group
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*GRP-Finance*" -and ObjectClass -eq "group"' `
    -IncludeDeletedObjects | Restore-ADObject

# After restoring, manually verify and re-add members if needed
Get-ADGroupMember -Identity "GRP-Finance"
```

!!! warning "Attributs linked-value"

    Les attributs de type linked-value (comme la membership de groupe) sont generalement conserves dans la corbeille, mais il est recommande de verifier apres restauration. Si des membres manquent, il faut les re-ajouter manuellement.

### Verifier les attributs d'un objet supprime

```powershell
# Examine all attributes of a deleted object before restoring
Get-ADObject -Filter 'IsDeleted -eq $true -and Name -like "*jdupont*"' `
    -IncludeDeletedObjects -Properties * | Format-List *
```

## Modifier la duree de retention

```powershell
# Set deleted object lifetime to 365 days
Set-ADObject "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=yourdomain,DC=local" `
    -Replace @{ "msDS-DeletedObjectLifetime" = 365 }

# Set tombstone (recycled object) lifetime to 365 days
Set-ADObject "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=yourdomain,DC=local" `
    -Replace @{ tombstoneLifetime = 365 }
```

## Protection contre la suppression accidentelle

Pour prevenir les suppressions, activez la protection sur les OU critiques :

```powershell
# Enable accidental deletion protection on an OU
Set-ADOrganizationalUnit -Identity "OU=Users,DC=yourdomain,DC=local" `
    -ProtectedFromAccidentalDeletion $true

# Enable protection on all OUs
Get-ADOrganizationalUnit -Filter * | Set-ADOrganizationalUnit -ProtectedFromAccidentalDeletion $true

# Verify protection status
Get-ADOrganizationalUnit -Filter * -Properties ProtectedFromAccidentalDeletion |
    Select-Object Name, ProtectedFromAccidentalDeletion
```

## Points cles a retenir

- La corbeille AD conserve **tous les attributs** des objets supprimes (contrairement au tombstone)
- L'activation est **irreversible** et necessite le niveau fonctionnel de foret **2008 R2 minimum**
- La duree de retention par defaut est de **180 jours** (configurable)
- Restaurez d'abord les conteneurs (OU) avant les objets enfants
- Activez la protection contre la suppression accidentelle sur toutes les OU critiques
- La corbeille AD ne remplace pas les sauvegardes : elle ne protege pas contre la corruption ou les modifications

## Pour aller plus loin

- Restauration systeme : [Restauration systeme](restauration-systeme.md)
- Windows Server Backup : [Windows Server Backup](windows-server-backup.md)
- Documentation Microsoft : Active Directory Recycle Bin Step-by-Step Guide
