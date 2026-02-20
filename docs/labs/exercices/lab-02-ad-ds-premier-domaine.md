---
title: "Lab 02 : Premier domaine AD DS"
description: Exercice pratique - creer un premier domaine Active Directory, promouvoir un controleur de domaine et verifier le fonctionnement.
tags:
  - lab
  - active-directory
  - debutant
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

# Lab 02 : Premier domaine AD DS

<span class="level-beginner">Debutant</span> · Temps estime : 60 minutes

---

!!! abstract "Objectifs du lab"

    - [ ] Installer le role AD DS sur SRV-DC01
    - [ ] Promouvoir SRV-DC01 en premier controleur de domaine
    - [ ] Creer le domaine winopslab.local
    - [ ] Verifier le fonctionnement du domaine (DNS, partages SYSVOL/NETLOGON)
    - [ ] Joindre SRV-DC02 et CLI-W11 au domaine

## Scenario

Votre responsable vous demande de mettre en place le domaine Active Directory de l'entreprise WinOpsLab. Vous devez creer une nouvelle foret, promouvoir le premier controleur de domaine, puis joindre les autres serveurs et un poste client au domaine.

## Environnement requis

| Ressource | Specification |
|-----------|---------------|
| SRV-DC01 | 2 vCPU, 4 Go RAM, IP 192.168.10.10 (configure au Lab 01) |
| SRV-DC02 | 2 vCPU, 4 Go RAM, IP 192.168.10.11 (configure au Lab 01) |
| CLI-W11 | 2 vCPU, 4 Go RAM, DHCP ou IP statique |
| Reseau | LabSwitch (switch interne) |

!!! warning "Prerequis"

    Le Lab 01 (installation et configuration initiale) doit etre termine.
    SRV-DC01 et SRV-DC02 doivent etre installes, renommes et configures avec des IP statiques.

```mermaid
graph TD
    FOREST["Foret : winopslab.local"]
    FOREST --> DOMAIN["Domaine : winopslab.local"]
    DOMAIN --> DC1["SRV-DC01<br/>192.168.10.10<br/>DC principal + DNS"]
    DOMAIN --> DC2["SRV-DC02<br/>192.168.10.11<br/>DC secondaire + DNS"]
    DOMAIN --> CLI["CLI-W11<br/>Poste client"]
    DC1 -->|"Replication AD"| DC2
    DC1 -.->|"DNS"| CLI
    DC2 -.->|"DNS"| CLI
```

## Instructions

!!! example "Analogie"

    Creer un domaine Active Directory, c'est comme ouvrir un hotel avec un registre central :
    le premier controleur de domaine est le receptionniste principal qui tient le registre
    (l'annuaire). Le second DC est son collegue de nuit avec un double du registre (replication).
    Sans IP statique configuree avant la promotion, c'est comme si le receptionniste changeait
    de bureau a chaque poste â€” personne ne saurait ou le trouver.

### Partie 1 : Installer le role AD DS sur SRV-DC01

1. Se connecter a SRV-DC01
2. Installer le role Active Directory Domain Services
3. Ne pas encore lancer la promotion (etape suivante)

??? success "Solution"

    ```powershell
    # Install AD DS role with management tools
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

    # Verify installation
    Get-WindowsFeature AD-Domain-Services
    ```

### Partie 2 : Promouvoir SRV-DC01 en controleur de domaine

1. Creer une nouvelle foret avec le domaine `winopslab.local`
2. Niveau fonctionnel foret et domaine : Windows Server 2016
3. Installer le DNS integre a AD
4. Definir le mot de passe DSRM

??? success "Solution"

    ```powershell
    # Promote to first domain controller in a new forest
    Install-ADDSForest `
        -DomainName "winopslab.local" `
        -DomainNetBIOSName "WINOPSLAB" `
        -ForestMode "WinThreshold" `
        -DomainMode "WinThreshold" `
        -InstallDns:$true `
        -DatabasePath "C:\Windows\NTDS" `
        -LogPath "C:\Windows\NTDS" `
        -SysvolPath "C:\Windows\SYSVOL" `
        -SafeModeAdministratorPassword (ConvertTo-SecureString "Dsrm-P@ss2026!" -AsPlainText -Force) `
        -NoRebootOnCompletion:$false `
        -Force:$true

    # The server will restart automatically
    ```

### Partie 3 : Verifier le domaine

Apres le redemarrage, verifier que le domaine est fonctionnel.

??? success "Solution"

    ```powershell
    # Verify domain controller status
    Get-ADDomainController | Select-Object Name, Domain, Forest, Site,
        IPv4Address, OperatingSystem

    # Verify domain
    Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode,
        PDCEmulator, InfrastructureMaster, RIDMaster

    # Verify forest
    Get-ADForest | Select-Object Name, ForestMode, SchemaMaster, DomainNamingMaster

    # Verify DNS zones
    Get-DnsServerZone | Select-Object ZoneName, ZoneType, IsDsIntegrated

    # Verify SYSVOL and NETLOGON shares
    Get-SmbShare | Where-Object { $_.Name -in "SYSVOL", "NETLOGON" }

    # Verify SRV records for DC location
    Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.winopslab.local" -Type SRV
    ```

    Resultat attendu apres verification du domaine :

    ```text
    # Get-ADDomainController | Select-Object Name, Domain, IPv4Address
    Name      Domain            IPv4Address
    ----      ------            -----------
    SRV-DC01  winopslab.local   192.168.10.10

    # Get-ADDomain | Select-Object DNSRoot, PDCEmulator
    DNSRoot         PDCEmulator
    -------         -----------
    winopslab.local SRV-DC01.winopslab.local

    # Get-SmbShare | Where-Object { $_.Name -in "SYSVOL","NETLOGON" }
    Name     Path
    ----     ----
    NETLOGON C:\Windows\SYSVOL\sysvol\winopslab.local\SCRIPTS
    SYSVOL   C:\Windows\SYSVOL\sysvol
    ```

### Partie 4 : Joindre SRV-DC02 au domaine et promouvoir en DC

1. Configurer le DNS de SRV-DC02 pour pointer vers SRV-DC01 (192.168.10.10)
2. Joindre SRV-DC02 au domaine winopslab.local
3. Installer AD DS et promouvoir en DC supplementaire

??? success "Solution"

    ```powershell
    # On SRV-DC02: Set DNS to point to SRV-DC01
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
        -ServerAddresses 192.168.10.10

    # Join the domain
    Add-Computer -DomainName "winopslab.local" `
        -Credential (Get-Credential WINOPSLAB\Administrateur) -Restart

    # After restart, install AD DS
    Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

    # Promote as additional domain controller
    Install-ADDSDomainController `
        -DomainName "winopslab.local" `
        -InstallDns:$true `
        -Credential (Get-Credential WINOPSLAB\Administrateur) `
        -DatabasePath "C:\Windows\NTDS" `
        -LogPath "C:\Windows\NTDS" `
        -SysvolPath "C:\Windows\SYSVOL" `
        -SafeModeAdministratorPassword (ConvertTo-SecureString "Dsrm-P@ss2026!" -AsPlainText -Force) `
        -NoRebootOnCompletion:$false `
        -Force:$true
    ```

### Partie 5 : Joindre CLI-W11 au domaine

1. Configurer le DNS du client pour pointer vers SRV-DC01
2. Joindre le poste au domaine winopslab.local

??? success "Solution"

    ```powershell
    # On CLI-W11: Set DNS
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
        -ServerAddresses 192.168.10.10, 192.168.10.11

    # Join the domain
    Add-Computer -DomainName "winopslab.local" `
        -Credential (Get-Credential WINOPSLAB\Administrateur) -Restart
    ```

## Verification

!!! question "Questions de validation"

    1. Quelle commande permet de lister tous les controleurs de domaine ?
    2. Quels sont les 5 roles FSMO et sur quel serveur sont-ils ?
    3. Pourquoi le DNS de SRV-DC01 doit-il pointer vers 127.0.0.1 en premier ?
    4. Que contient le partage SYSVOL ?

??? success "Reponses"

    1. `Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, Site`
    2. Les 5 roles FSMO sont : Schema Master, Domain Naming Master, PDC Emulator,
       RID Master, Infrastructure Master. Tous sont sur SRV-DC01 (premier DC).
       Verification : `netdom query fsmo`
    3. Le premier DC est son propre serveur DNS. Il doit pointer vers lui-meme en premier
       pour resoudre les enregistrements SRV necessaires au fonctionnement d'AD DS.
    4. SYSVOL contient les scripts de connexion et les fichiers de strategies de groupe (GPO).
       Il est replique entre tous les controleurs de domaine.

!!! warning "Pieges frequents dans ce lab"

    1. **IP statique non configuree avant la promotion** : si SRV-DC01 est encore en DHCP
       au moment de `Install-ADDSForest`, le DNS integre a AD sera lie a une adresse qui peut
       changer. La promotion peut reussir mais le DNS sera inaccessible apres un redemarrage.
       Toujours configurer l'IP statique ET le DNS (127.0.0.1 ou l'IP du serveur lui-meme) avant.

    2. **DNS de SRV-DC02 pointant vers lui-meme** : avant de joindre SRV-DC02 au domaine,
       son DNS doit pointer vers SRV-DC01 (192.168.10.10), pas vers lui-meme. Un DNS pointant
       vers 127.0.0.1 ne peut pas resoudre winopslab.local car la zone n'existe pas encore
       localement.

    3. **Mot de passe DSRM oublie ou trop simple** : le mot de passe DSRM est independant
       du mot de passe Administrateur du domaine. Il doit respecter la politique de complexite.
       Notez-le dans votre documentation de lab â€” impossible a recuperer sans reinitialisation.

    4. **Joindre SRV-DC02 sans redemarrer avant la promotion** : apres `Add-Computer -Restart`,
       attendre le redemarrage complet et la reconnexion avec le compte de domaine avant
       d'executer `Install-ADDSDomainController`. Une promotion lancee sans redemarrage
       produit des erreurs de replication.

    5. **Ne pas verifier les enregistrements SRV** : apres la promotion, un domaine AD
       fonctionnel doit publier ses enregistrements `_ldap._tcp.dc._msdcs.winopslab.local`
       dans le DNS. S'ils sont absents, les clients et serveurs ne pourront pas localiser
       le controleur de domaine meme si le ping reussit.

## Nettoyage

Conservez l'environnement pour les labs suivants. Si vous devez recommencer :

```powershell
# On SRV-DC02: Demote the domain controller
Uninstall-ADDSDomainController -DemoteOperationMasterRole -Force

# On SRV-DC01: Remove the forest (destructive!)
Uninstall-ADDSDomainController -LastDomainControllerInDomain `
    -RemoveApplicationPartitions -Force
```

## Prochaine etape

:material-arrow-right: [Lab 03 : DNS et DHCP](lab-03-dns-dhcp.md)

