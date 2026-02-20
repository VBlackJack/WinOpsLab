---
title: "Protection des comptes privilegies"
description: "Modele d'administration en tiers, PAW, groupe Protected Users et Credential Guard pour proteger les comptes a privileges sur Windows Server 2022."
tags:
  - securite
  - durcissement
  - comptes-privilegies
  - tier-model
  - credential-guard
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

# Protection des comptes privilegies

<span class="level-advanced">Avance</span> Â· Temps estime : 50 minutes

Les comptes privilegies sont la cible prioritaire des attaquants. Un compte Domain Admin compromis donne un acces total a l'infrastructure. La protection de ces comptes necessite une architecture de securite en couches.

---

## Le modele d'administration en tiers

!!! example "Analogie"

    Le modele en tiers fonctionne comme la securite d'une banque : le coffre-fort (Tier 0) n'est accessible qu'au directeur, les guichets (Tier 1) sont geres par les employes de la banque, et l'accueil (Tier 2) est ouvert au public. Un agent d'accueil n'a jamais la cle du coffre-fort, et le directeur ne descend pas a l'accueil pour eviter de se faire derober ses cles.

Le **Tiered Administration Model** (modele en niveaux) segmente l'administration en trois zones de confiance pour limiter la propagation d'une compromission.

```mermaid
flowchart TB
    subgraph Tier0["Tier 0 - Identite"]
        DC[Controleurs de domaine]
        ADFS[AD Federation Services]
        CA[Autorite de certification]
        PKI[Infrastructure PKI]
    end

    subgraph Tier1["Tier 1 - Serveurs"]
        SRV[Serveurs applicatifs]
        SQL[Serveurs SQL]
        FILE[Serveurs de fichiers]
        EXCH[Serveurs Exchange]
    end

    subgraph Tier2["Tier 2 - Postes de travail"]
        WKS[Postes de travail]
        MOB[Appareils mobiles]
        PRINT[Imprimantes]
    end

    Tier0 -.->|Controle| Tier1
    Tier1 -.->|Controle| Tier2
    Tier2 -.-x|Interdit| Tier0
    Tier2 -.-x|Interdit| Tier1
```

### Principes fondamentaux

| Regle | Description |
|-------|-------------|
| **Pas de connexion descendante** | Un admin Tier 0 ne se connecte jamais sur un poste Tier 2 |
| **Pas d'elevation ascendante** | Un admin Tier 2 ne peut pas administrer un serveur Tier 1 |
| **Comptes dedies par tier** | Chaque administrateur possede un compte distinct par niveau |
| **Stations d'administration dediees** | L'administration de chaque tier se fait depuis des postes specifiques |

### Comptes a creer

```powershell
# Create tiered admin accounts for a user
$tiers = @(
    @{ Name = "T0-jdupont"; Tier = "Tier 0"; OU = "OU=Tier0,OU=Admin Accounts,DC=lab,DC=local" },
    @{ Name = "T1-jdupont"; Tier = "Tier 1"; OU = "OU=Tier1,OU=Admin Accounts,DC=lab,DC=local" },
    @{ Name = "T2-jdupont"; Tier = "Tier 2"; OU = "OU=Tier2,OU=Admin Accounts,DC=lab,DC=local" }
)

foreach ($account in $tiers) {
    New-ADUser -Name $account.Name `
        -SamAccountName $account.Name `
        -Path $account.OU `
        -Description "$($account.Tier) admin account" `
        -Enabled $true `
        -AccountPassword (Read-Host -AsSecureString "Password for $($account.Name)")
}
```

Resultat :

```text
PS C:\> New-ADUser -Name "T0-jdupont" ...
PS C:\> New-ADUser -Name "T1-jdupont" ...
PS C:\> New-ADUser -Name "T2-jdupont" ...

PS C:\> Get-ADUser -Filter * -SearchBase "OU=Admin Accounts,DC=lab,DC=local" | Select-Object Name, Enabled

Name          Enabled
----          -------
T0-jdupont    True
T1-jdupont    True
T2-jdupont    True
T0-mlambert   True
T1-mlambert   True
T2-mlambert   True
```

### Restrictions de connexion par GPO

Pour chaque tier, configurez les droits de connexion via GPO :

```
# Computer Configuration > Policies > Windows Settings > Security Settings
# > Local Policies > User Rights Assignment

# On Tier 0 systems (Domain Controllers):
# - "Allow log on locally"     : Tier 0 admins only
# - "Deny log on locally"      : Tier 1 and Tier 2 accounts
# - "Deny log on through RDP"  : Tier 1 and Tier 2 accounts

# On Tier 1 systems (Servers):
# - "Allow log on locally"     : Tier 1 admins only
# - "Deny log on locally"      : Tier 0 and Tier 2 accounts

# On Tier 2 systems (Workstations):
# - "Allow log on locally"     : Tier 2 admins and standard users
# - "Deny log on locally"      : Tier 0 and Tier 1 accounts
```

---

## Privileged Access Workstations (PAW)

Une **PAW** est un poste de travail dedie exclusivement a l'administration. Il ne sert ni a la navigation web, ni a la messagerie, ni a aucune autre activite quotidienne.

### Caracteristiques d'une PAW

- **OS durci** : Windows derniere version, baselines de securite appliquees
- **Acces reseau restreint** : uniquement vers les systemes cibles
- **Pas d'acces Internet** : ou via un proxy tres restrictif
- **Pas d'email ni bureautique** : aucune application utilisateur
- **BitLocker actif** : chiffrement integral du disque
- **Credential Guard actif** : protection des identifiants en memoire
- **AppLocker / WDAC** : seuls les outils d'administration sont autorises

### Segmentation reseau pour les PAWs

```mermaid
flowchart LR
    subgraph PAW_Network["Reseau PAW (isole)"]
        PAW0[PAW Tier 0]
        PAW1[PAW Tier 1]
    end

    subgraph DC_Network["Reseau DC"]
        DC1[DC01]
        DC2[DC02]
    end

    subgraph Server_Network["Reseau Serveurs"]
        SRV1[SRV-APP01]
        SRV2[SRV-SQL01]
    end

    PAW0 -->|RDP / WinRM| DC1
    PAW0 -->|RDP / WinRM| DC2
    PAW1 -->|RDP / WinRM| SRV1
    PAW1 -->|RDP / WinRM| SRV2
    PAW0 -.-x Server_Network
    PAW1 -.-x DC_Network
```

---

## Groupe Protected Users

Le groupe **Protected Users** est un groupe de securite introduit avec Windows Server 2012 R2 qui applique automatiquement des restrictions de securite aux comptes membres.

### Protections appliquees

| Protection | Detail |
|------------|--------|
| Pas de mise en cache NTLM | Les identifiants NTLM ne sont pas mis en cache |
| Pas de delegation | La delegation Kerberos est refusee |
| Pas de DES/RC4 | Seul AES est autorise pour Kerberos |
| TGT a duree reduite | Duree de vie du TGT limitee a 4 heures (au lieu de 10) |
| Pas de CredSSP | Les identifiants ne transitent pas via CredSSP |

### Configuration

```powershell
# Add an admin account to the Protected Users group
Add-ADGroupMember -Identity "Protected Users" -Members "T0-jdupont"

# Verify membership
Get-ADGroupMember -Identity "Protected Users" | Select-Object Name, SamAccountName

# Check if a specific user is in the group
$user = Get-ADUser "T0-jdupont" -Properties MemberOf
$user.MemberOf | Where-Object { $_ -match "Protected Users" }
```

Resultat :

```text
Name            SamAccountName
----            --------------
T0-jdupont      T0-jdupont
T0-mlambert     T0-mlambert

CN=Protected Users,CN=Users,DC=lab,DC=local
```

!!! warning "Impacts operationnels"

    L'ajout au groupe Protected Users peut casser certaines applications :

    - **NTLM requis** : certains outils legacy ne supportent pas Kerberos seul
    - **Delegation necessaire** : les services utilisant la delegation (SQL Server, IIS) ne fonctionneront plus
    - **TGT court** : l'administrateur devra se re-authentifier toutes les 4 heures

    **Testez soigneusement avant le deploiement en production.**

---

## Windows Credential Guard

**Credential Guard** utilise la securite basee sur la virtualisation (VBS) pour isoler les secrets (hashes NTLM, tickets Kerberos TGT) dans un conteneur protege inaccessible meme par le noyau Windows.

### Prerequis

- Processeur 64 bits avec extensions de virtualisation (Intel VT-x / AMD-V)
- UEFI Secure Boot active
- TPM 2.0 recommande
- Windows Server 2016 ou superieur

### Activation via GPO

```
Computer Configuration
  > Administrative Templates
    > System
      > Device Guard
        > Turn On Virtualization Based Security : Enabled
          - Platform Security Level : Secure Boot and DMA Protection
          - Credential Guard Configuration : Enabled with UEFI lock
```

### Activation via PowerShell

```powershell
# Enable Credential Guard via registry
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Control\DeviceGuard"

# Enable Virtualization Based Security
Set-ItemProperty -Path $regPath -Name "EnableVirtualizationBasedSecurity" -Value 1 -Type DWord

# Enable Credential Guard (value 1 = enabled with UEFI lock, value 2 = enabled without lock)
Set-ItemProperty -Path $regPath -Name "RequirePlatformSecurityFeatures" -Value 3 -Type DWord

$lsaCfgPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa"
Set-ItemProperty -Path $lsaCfgPath -Name "LsaCfgFlags" -Value 1 -Type DWord
```

### Verification

```powershell
# Verify Credential Guard status
$devGuard = Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard
$devGuard | Select-Object -Property SecurityServicesRunning, VirtualizationBasedSecurityStatus

# SecurityServicesRunning should include "1" (Credential Guard)
# VirtualizationBasedSecurityStatus: 2 = Running

# Alternative: check via msinfo32
# System Summary > Virtualization-based security > Running
# > Credential Guard > Running
```

Resultat :

```text
SecurityServicesRunning          VirtualizationBasedSecurityStatus
--------------------------       --------------------------------
{0, 1}                           2
```

!!! danger "UEFI Lock"

    Avec l'option **UEFI lock**, Credential Guard ne peut pas etre desactive a distance. Il faut un acces physique au serveur pour modifier la configuration UEFI. Cela protege contre la desactivation par un attaquant, mais complique les operations de maintenance.

---

## Bonnes pratiques supplementaires

### Strategie de mots de passe pour les comptes privilegies

```powershell
# Create a Fine-Grained Password Policy for Tier 0 admins
New-ADFineGrainedPasswordPolicy -Name "Tier0-PasswordPolicy" `
    -Precedence 10 `
    -MinPasswordLength 20 `
    -PasswordHistoryCount 30 `
    -ComplexityEnabled $true `
    -MaxPasswordAge "60.00:00:00" `
    -MinPasswordAge "1.00:00:00" `
    -LockoutThreshold 5 `
    -LockoutDuration "00:30:00" `
    -LockoutObservationWindow "00:30:00" `
    -ReversibleEncryptionEnabled $false

# Apply the policy to the Tier 0 admin group
Add-ADFineGrainedPasswordPolicy -Identity "Tier0-PasswordPolicy" `
    -Subjects "Tier0-Admins"
```

### Surveiller les comptes privilegies

```powershell
# Monitor changes to privileged groups
$privilegedGroups = @("Domain Admins", "Enterprise Admins", "Schema Admins", "Administrators")

foreach ($group in $privilegedGroups) {
    $members = Get-ADGroupMember -Identity $group
    Write-Output "=== $group ($($members.Count) members) ==="
    $members | Select-Object Name, SamAccountName, objectClass | Format-Table -AutoSize
}
```

Resultat :

```text
=== Domain Admins (3 members) ===
Name            SamAccountName  objectClass
----            --------------  -----------
Administrator   Administrator   user
T0-jdupont      T0-jdupont      user
T0-mlambert     T0-mlambert     user

=== Enterprise Admins (1 members) ===
Name            SamAccountName  objectClass
----            --------------  -----------
Administrator   Administrator   user

=== Schema Admins (1 members) ===
Name            SamAccountName  objectClass
----            --------------  -----------
Administrator   Administrator   user

=== Administrators (4 members) ===
Name            SamAccountName  objectClass
----            --------------  -----------
Administrator   Administrator   user
Domain Admins   Domain Admins   group
Enterprise A... Enterprise A... group
T0-jdupont      T0-jdupont      user
```

---

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Marc, responsable securite, decouvre lors d'un test de penetration que l'auditeur a pu extraire les credentials d'un compte Domain Admin depuis la memoire d'un serveur applicatif (SRV-APP01, 10.0.0.20). L'attaquant a utilise Mimikatz pour effectuer un Pass-the-Hash.

    **Diagnostic** :

    ```powershell
    # Check if the compromised admin is in Protected Users
    Get-ADGroupMember -Identity "Protected Users" | Select-Object Name
    ```

    Resultat :

    ```text
    Name
    ----
    (vide - aucun membre)
    ```

    ```powershell
    # Check Credential Guard status on the server
    Invoke-Command -ComputerName "SRV-APP01" -ScriptBlock {
        (Get-CimInstance -ClassName Win32_DeviceGuard -Namespace root\Microsoft\Windows\DeviceGuard).VirtualizationBasedSecurityStatus
    }
    ```

    Resultat :

    ```text
    0
    ```

    **Resolution** :

    1. Les comptes Tier 0 n'etaient pas dans le groupe Protected Users
    2. Credential Guard n'etait pas active sur les serveurs
    3. Le compte Domain Admin avait ouvert une session RDP sur un serveur Tier 1 (violation du modele en tiers)

    ```powershell
    # 1. Add Tier 0 accounts to Protected Users
    Add-ADGroupMember -Identity "Protected Users" -Members "T0-jdupont", "T0-mlambert"

    # 2. Verify no Tier 0 sessions on Tier 1 servers
    $tier1Servers = Get-ADComputer -Filter * -SearchBase "OU=Servers,DC=lab,DC=local" | Select-Object -ExpandProperty Name
    foreach ($srv in $tier1Servers) {
        $sessions = query user /server:$srv 2>$null
        Write-Output "=== $srv ===" ; $sessions
    }
    ```

    Marc deploie ensuite Credential Guard via GPO sur tous les serveurs et met en place les restrictions de connexion par tier.

---

!!! danger "Erreurs courantes"

    1. **Utiliser un seul compte admin pour tous les niveaux** : un meme compte ne doit jamais se connecter a des systemes de tiers differents. Si un poste Tier 2 est compromis et que votre compte Tier 0 y a une session, toute l'infrastructure est en danger.

    2. **Ajouter des comptes de service au groupe Protected Users** : les services utilisant la delegation Kerberos ou NTLM cesseront de fonctionner. Le groupe Protected Users est reserve aux comptes interactifs d'administration.

    3. **Activer Credential Guard avec UEFI Lock sans plan de recuperation** : avec le verrouillage UEFI, Credential Guard ne peut etre desactive qu'avec un acces physique au serveur. Prevoyez une procedure documentee pour les cas de maintenance.

    4. **Ne pas tester l'impact de Protected Users avant le deploiement** : certains outils d'administration legacy ne supportent que NTLM. Testez dans un environnement de pre-production avec des comptes pilotes avant de generaliser.

---

## Points cles a retenir

- Le **modele en tiers** segmente l'administration pour limiter la propagation des compromissions
- Chaque administrateur doit disposer de **comptes distincts par niveau** (Tier 0, 1, 2)
- Les **PAWs** (Privileged Access Workstations) fournissent un environnement d'administration isole et durci
- Le groupe **Protected Users** applique des restrictions de securite automatiques aux comptes membres
- **Credential Guard** protege les identifiants en memoire grace a la virtualisation (VBS)
- Ces mesures doivent etre **combinees** pour une defense en profondeur efficace

---

## Pour aller plus loin

- Microsoft : Securing Privileged Access (SPA) roadmap
- Microsoft : Privileged Access Workstations deployment guide
- ANSSI : Recommandations pour l'administration securisee des SI
- LAPS pour la gestion des mots de passe locaux (voir la page [LAPS](laps.md))

