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
---
title: "Azure AD Connect"
description: Mettre en place l'identite hybride avec Azure AD Connect - synchronisation, modes d'authentification et architecture.
tags:
  - gestion-moderne
  - azure
  - identite
  - intermediaire
---

# Azure AD Connect

<span class="level-intermediate">Intermediaire</span> · Temps estime : 30 minutes

## Presentation

Azure AD Connect (renomme **Microsoft Entra Connect** depuis 2023) est l'outil qui synchronise les identites de votre Active Directory on-premises vers Azure AD (Microsoft Entra ID). Il permet aux utilisateurs de se connecter aux services cloud Microsoft (Microsoft 365, Azure) avec leurs identifiants Active Directory existants.

```mermaid
graph LR
    A[Active Directory<br/>on-premises] -->|Azure AD Connect| B[Azure AD<br/>Microsoft Entra ID]
    B --> C[Microsoft 365]
    B --> D[Azure Portal]
    B --> E[Applications SaaS]
    A --> F[Utilisateurs]
    F -->|Memes identifiants| C
    F -->|Memes identifiants| D
```

!!! example "Analogie"

    Azure AD Connect, c'est comme un employe de la mairie qui transcrit chaque jour le registre civil local (votre Active Directory) vers une base nationale (Azure AD). Les deux bases restent independantes, mais sont synchronisees : ajouter un habitant dans le registre local le fait apparaitre automatiquement dans la base nationale.

## Modes d'authentification

Azure AD Connect propose trois modes d'authentification :

### Password Hash Synchronization (PHS)

![Azure AD Connect PHS](../../diagrams/azure-ad-connect.drawio#Azure_AD_Connect_PHS)

| Caracteristique | Valeur |
|-----------------|--------|
| Complexite | Faible |
| Dependance on-premises | Aucune (auth cloud) |
| Haute disponibilite | Native (Azure AD) |
| Recommandation | Mode par defaut recommande |

Le hash du mot de passe est synchronise vers Azure AD. L'authentification se fait entierement dans le cloud.

```mermaid
graph LR
    A[Utilisateur] -->|Identifiants| B[Azure AD]
    B -->|Valide le hash| B
    C[AD on-premises] -->|Sync hash| B
```

### Pass-through Authentication (PTA)

| Caracteristique | Valeur |
|-----------------|--------|
| Complexite | Moyenne |
| Dependance on-premises | Oui (agents PTA) |
| Haute disponibilite | Necessite plusieurs agents |
| Recommandation | Quand PHS n'est pas acceptable (conformite) |

Azure AD redirige la validation du mot de passe vers un agent PTA on-premises.

```mermaid
graph LR
    A[Utilisateur] -->|Identifiants| B[Azure AD]
    B -->|Valide via agent| C[Agent PTA]
    C -->|Verifie dans AD| D[AD on-premises]
```

### Federation (AD FS)

| Caracteristique | Valeur |
|-----------------|--------|
| Complexite | Elevee |
| Dependance on-premises | Forte (ferme AD FS) |
| Haute disponibilite | Necessite une infrastructure AD FS HA |
| Recommandation | Scenarios avances (smartcard, MFA tiers, etc.) |

L'authentification est entierement geree par AD FS on-premises.

!!! tip "Recommandation Microsoft"

    Microsoft recommande **Password Hash Sync** comme mode par defaut. Il offre
    la meilleure resilience et ne depend pas de l'infrastructure on-premises pour
    l'authentification. Activez-le meme en complement de PTA ou AD FS comme
    solution de secours.

## Prerequis

### Infrastructure

| Element | Requis |
|---------|--------|
| AD DS | Foret Active Directory fonctionnelle |
| Niveau fonctionnel | Windows Server 2003+ |
| Serveur AAD Connect | Windows Server 2016+ (dedie, joint au domaine) |
| SQL | SQL Server Express (inclus) ou SQL Server existant |
| .NET Framework | 4.7.2+ |

### Reseau

| Endpoint | Port | Description |
|----------|------|-------------|
| `*.microsoftonline.com` | 443 | Authentification Azure AD |
| `*.windows.net` | 443 | Azure AD Graph |
| `*.msappproxy.net` | 443 | Proxy d'application (PTA) |
| `*.servicebus.windows.net` | 443 | Service Bus (PTA) |

### Compte requis

| Compte | Permissions |
|--------|-------------|
| **Administrateur Global Azure AD** | Configuration initiale |
| **Administrateur d'entreprise AD** | Configuration du connecteur AD |
| **Compte de service ADSync** | Cree automatiquement lors de l'installation |

## Installation

### Installation Express

Le mode Express convient a la plupart des environnements mono-foret :

1. Telecharger Azure AD Connect depuis le centre de telechargement Microsoft
2. Executer `AzureADConnect.msi`
3. Accepter les termes de licence
4. Cliquer sur **Configuration Express**
5. Saisir les identifiants de l'administrateur global Azure AD
6. Saisir les identifiants de l'administrateur d'entreprise AD
7. Cliquer sur **Installer**

Le mode Express configure automatiquement :

- Password Hash Synchronization
- Synchronisation de tous les utilisateurs et groupes
- Seamless Single Sign-On (SSO)
- Cycle de synchronisation toutes les 30 minutes

### Installation personnalisee

Pour les environnements complexes (multi-foret, filtrage, PTA, federation) :

1. Selectionner **Personnaliser** au lieu de Configuration Express
2. Options disponibles :
    - Choix du mode d'authentification (PHS, PTA, AD FS)
    - Filtrage par OU, domaine ou attribut
    - Configuration du serveur SQL
    - Writeback des attributs (mot de passe, appareils, groupes)
    - Mappage d'attributs personnalise

## Filtrage de la synchronisation

### Filtrage par OU

Synchroniser uniquement certaines unites d'organisation :

1. Azure AD Connect > **Configuration** > **Personnaliser les options de synchronisation**
2. Decocher les OU a exclure (ex. : exclure les OU de test, comptes de service)

### Filtrage par attribut

```powershell
# Set a custom attribute to control sync
# Example: only sync users where extensionAttribute1 = "SyncToAzure"
Set-ADUser -Identity "j.bombled" -Replace @{extensionAttribute1="SyncToAzure"}
```

## Gestion du cycle de synchronisation

```powershell
# Import the ADSync module
Import-Module ADSync

# Check synchronization status
Get-ADSyncScheduler

# Force a full synchronization cycle
Start-ADSyncSyncCycle -PolicyType Initial

# Force a delta synchronization (changes only)
Start-ADSyncSyncCycle -PolicyType Delta

# Disable the scheduler (for maintenance)
Set-ADSyncScheduler -SyncCycleEnabled $false

# Re-enable the scheduler
Set-ADSyncScheduler -SyncCycleEnabled $true

# Change sync interval (minimum 30 minutes)
Set-ADSyncScheduler -CustomizedSyncCycleInterval 00:30:00
```

Resultat :

```text
# Get-ADSyncScheduler
AllowedSyncCycleInterval      : 00:30:00
CurrentlyEffectiveSyncCycleInterval : 00:30:00
CustomizedSyncCycleInterval   : 00:00:00
NextSyncCyclePolicyType       : Delta
NextSyncCycleStartTimeInUTC   : 2026-02-20T08:47:23.142Z
PurgeRunHistoryInterval       : 7.00:00:00
SyncCycleEnabled              : True
MaintenanceEnabled            : True
StagingModeEnabled            : False
SchedulerSuspended            : False

# Start-ADSyncSyncCycle -PolicyType Delta
Result
------
Success
```

## Seamless SSO

Le Seamless Single Sign-On permet aux utilisateurs sur des postes joints au domaine de se connecter automatiquement aux services cloud sans ressaisir leur mot de passe.

### Prerequis

- Azure AD Connect avec PHS ou PTA
- Postes de travail joints au domaine AD
- La zone Intranet doit inclure l'URL Azure AD

### Activation

Le Seamless SSO est active automatiquement avec l'installation Express. Pour l'activer manuellement :

1. Relancer l'assistant Azure AD Connect
2. **Modifier la connexion utilisateur**
3. Cocher **Activer l'authentification unique**

## Password Writeback

Le writeback de mot de passe permet aux utilisateurs de reinitialiser leur mot de passe dans Azure AD et de le synchroniser vers l'AD on-premises (Azure AD Self-Service Password Reset).

### Activation

1. Relancer l'assistant Azure AD Connect
2. **Fonctionnalites facultatives**
3. Cocher **Ecriture diferee de mot de passe**

### Prerequis

- Licence Azure AD Premium P1 ou P2
- Le compte de service ADSync doit avoir les permissions de reinitialisation de mot de passe dans AD

## Surveillance et depannage

```powershell
# Check synchronization errors
Get-ADSyncCSObject -ConnectorName "winopslab.local - AAD" |
    Where-Object { $_.HasSyncError }

# View sync statistics
Get-ADSyncRunProfileResult -ConnectorName "winopslab.local" -RunProfileName "Delta Synchronization" |
    Select-Object -First 5

# Check connector space
Get-ADSyncConnectorStatistics -ConnectorName "winopslab.local"

# Azure AD Connect Health (portal)
# Navigate to: portal.azure.com > Azure AD Connect Health
```

Resultat :

```text
# Get-ADSyncRunProfileResult (extrait)
StartDate               EndDate                 Result      NumberOfChangesImported NumberOfChangesExported
---------               -------                 ------      ----------------------- -----------------------
2026-02-20 08:17:23     2026-02-20 08:17:31     success     12                      8
2026-02-20 07:47:19     2026-02-20 07:47:27     success     3                       2
2026-02-20 07:17:16     2026-02-20 07:17:24     success     0                       0

# Get-ADSyncConnectorStatistics
ConnectorName                   : lab.local
NumberOfObjects                 : 847
NumberOfConnectors              : 847
NumberOfDisconnectors           : 0
NumberOfPendingExports          : 0
```

### Erreurs courantes

| Erreur | Cause probable | Solution |
|--------|---------------|----------|
| **IdentitySynchronizationError** | Attribut en double (UPN, proxyAddress) | Corriger les doublons dans l'AD on-premises |
| **InvalidSoftMatch** | Objet existant dans Azure AD sans correspondance | Verifier le mappage des attributs |
| **LargeObject** | Attribut depassant la limite de taille Azure AD | Reduire la taille de l'attribut |
| **DataValidationFailed** | Format d'attribut invalide | Corriger le format (ex. : UPN valide) |

!!! example "Scenario pratique"

    **Context :** David, administrateur systeme, vient de creer 15 comptes utilisateurs dans l'AD on-premises de `lab.local` pour une nouvelle equipe. Ces utilisateurs doivent pouvoir acceder a Microsoft 365 (Teams, Exchange Online) dans la journee.

    **Etape 1 : Verifier que les comptes sont correctement configures pour la sync**

    ```powershell
    # Verifier le UPN des nouveaux comptes
    Get-ADUser -Filter { Department -eq "Nouvelle Equipe" } |
        Select-Object SamAccountName, UserPrincipalName, Enabled
    ```

    David constate que deux comptes ont un UPN avec le domaine `lab.local` au lieu du domaine verifie `winopslab.com`. Il corrige :

    ```powershell
    Get-ADUser -Filter { UserPrincipalName -like "*@lab.local" } |
        Set-ADUser -UserPrincipalName { $_.SamAccountName + "@winopslab.com" }
    ```

    **Etape 2 : Forcer une synchronisation delta**

    ```powershell
    Import-Module ADSync
    Start-ADSyncSyncCycle -PolicyType Delta
    ```

    **Etape 3 : Verifier dans Azure AD**

    David ouvre le portail Azure > Azure Active Directory > Utilisateurs. Les 15 nouveaux comptes sont bien visibles avec le statut "Synchronise depuis l'annuaire local". Il assigne les licences Microsoft 365 E3 depuis le portail.

    **Resultat :** Les 15 utilisateurs peuvent se connecter a Teams et Outlook Online dans les minutes suivant la synchronisation, avec leurs identifiants Active Directory habituels.

!!! danger "Erreurs courantes"

    **Installer Azure AD Connect sur un controleur de domaine.** Bien que techniquement possible, cette configuration est deconseillee. Le compte de service ADSync obtient des droits etendus sur l'AD ; un serveur dedie limite l'exposition en cas de compromission.

    **Ignorer les erreurs de synchronisation dans le portail Azure AD Connect Health.** Les erreurs IdentitySynchronizationError (UPN en double, attribut proxyAddress conflict) bloquent silencieusement certains comptes. Verifier le portail de sante hebdomadairement.

    **Modifier directement un objet Azure AD qui est synchronise depuis l'AD on-premises.** Les modifications cote Azure AD (portal) sont ecrasees au cycle de sync suivant. Toujours modifier l'objet dans l'AD on-premises, jamais directement dans Azure AD pour les comptes synchronises.

    **Oublier le Password Writeback lors de l'activation du Self-Service Password Reset.** Sans Password Writeback active dans Azure AD Connect, les utilisateurs qui reinitalisent leur mot de passe via SSPR ne peuvent plus se connecter on-premises (le mot de passe n'est pas repercute dans l'AD local).

    **Ne pas activer Password Hash Sync comme fallback.** Meme si PTA ou AD FS est utilise, activer PHS en complement garantit que les utilisateurs peuvent s'authentifier si l'infrastructure on-premises est indisponible.

## Points cles a retenir

- Azure AD Connect synchronise les identites AD on-premises vers Azure AD (Microsoft Entra ID)
- **Password Hash Sync** est le mode recommande : simple, resilient, pas de dependance on-premises
- Le mode **Express** couvre la majorite des besoins (mono-foret, PHS, Seamless SSO)
- Le cycle de synchronisation par defaut est de **30 minutes** (delta sync)
- Le **Password Writeback** permet le Self-Service Password Reset depuis le cloud vers l'AD
- Azure AD Connect doit etre installe sur un serveur dedie, joint au domaine

## Pour aller plus loin

- [Azure Arc](azure-arc.md) pour la gestion hybride des serveurs
- [Azure Monitor](azure-monitor.md) pour la surveillance des serveurs
- [Azure Backup](azure-backup.md) pour la sauvegarde cloud
