---
title: RSAT
description: Remote Server Administration Tools - administrer Windows Server depuis un poste client.
tags:
  - fondamentaux
  - console
  - debutant
---

# RSAT (Remote Server Administration Tools)

<span class="level-beginner">Debutant</span> · Temps estime : 10 minutes

## Presentation

!!! example "Analogie"

    Imaginez que vous etes directeur d'usine et que vous avez une telecommande universelle. Au lieu
    de vous deplacer physiquement dans chaque atelier (connexion RDP sur chaque serveur), vous pilotez
    toutes les machines depuis votre bureau. RSAT, c'est cette telecommande : les memes boutons (snap-ins)
    que sur les serveurs, mais installes directement sur votre poste de travail.

Les **RSAT** (Remote Server Administration Tools) permettent d'installer les consoles d'administration (MMC snap-ins) sur un poste **Windows 10 ou 11** pour gerer des serveurs a distance, sans avoir besoin de se connecter en RDP.

!!! tip "Bonne pratique"

    Installez RSAT sur votre poste de travail plutot que de vous connecter en RDP
    sur chaque serveur. C'est plus rapide, plus securise, et centralise la gestion.

## Installation sur Windows 10 / 11

Depuis Windows 10 version 1809, les RSAT sont disponibles en tant que **Features on Demand** (fonctionnalites a la demande).

=== "PowerShell (recommande)"

    ```powershell
    # List all available RSAT features
    Get-WindowsCapability -Online | Where-Object Name -like "Rsat.*"

    # Install all RSAT tools at once
    Get-WindowsCapability -Online |
        Where-Object Name -like "Rsat.*" |
        Where-Object State -eq "NotPresent" |
        Add-WindowsCapability -Online

    # Install specific RSAT tools
    Add-WindowsCapability -Online -Name "Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0"
    Add-WindowsCapability -Online -Name "Rsat.Dns.Tools~~~~0.0.1.0"
    Add-WindowsCapability -Online -Name "Rsat.DHCP.Tools~~~~0.0.1.0"
    Add-WindowsCapability -Online -Name "Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0"
    Add-WindowsCapability -Online -Name "Rsat.ServerManager.Tools~~~~0.0.1.0"

    # Verify installed RSAT tools
    Get-WindowsCapability -Online |
        Where-Object { $_.Name -like "Rsat.*" -and $_.State -eq "Installed" }
    ```

    Resultat de `Get-WindowsCapability -Online | Where-Object Name -like "Rsat.*"` (extrait) :

    ```text
    Name  : Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
    State : Installed

    Name  : Rsat.DHCP.Tools~~~~0.0.1.0
    State : NotPresent

    Name  : Rsat.Dns.Tools~~~~0.0.1.0
    State : Installed

    Name  : Rsat.GroupPolicy.Management.Tools~~~~0.0.1.0
    State : Installed

    Name  : Rsat.ServerManager.Tools~~~~0.0.1.0
    State : Installed

    Name  : Rsat.FailoverCluster.Management.Tools~~~~0.0.1.0
    State : NotPresent
    ```

    Resultat de `Add-WindowsCapability` :

    ```text
    Path          :
    Online        : True
    RestartNeeded : False
    ```

=== "Interface graphique"

    1. **Parametres** > **Applications** > **Fonctionnalites facultatives**
    2. Cliquer sur **Ajouter une fonctionnalite**
    3. Rechercher "RSAT"
    4. Selectionner et installer les outils souhaites

## Outils RSAT disponibles

| Outil RSAT | Console | Gere |
|------------|---------|------|
| AD DS and AD LDS Tools | `dsa.msc`, `dssite.msc` | Active Directory |
| DNS Server Tools | `dnsmgmt.msc` | DNS |
| DHCP Server Tools | `dhcpmgmt.msc` | DHCP |
| Group Policy Management | `gpmc.msc` | GPO |
| Server Manager | Server Manager | Vue d'ensemble |
| Failover Clustering | `cluadmin.msc` | Clusters |
| Hyper-V Management | `virtmgmt.msc` | Hyper-V |
| BitLocker Tools | `manage-bde` | BitLocker |

## Configuration du poste d'administration

Pour gerer des serveurs depuis un poste non joint au domaine :

```powershell
# Add remote servers to the TrustedHosts list
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "SRV-DC-01,SRV-FS-01" -Force

# Or allow all servers (less secure, lab only)
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force

# Verify TrustedHosts configuration
Get-Item WSMan:\localhost\Client\TrustedHosts
```

Resultat de `Get-Item WSMan:\localhost\Client\TrustedHosts` :

```text
   WSManConfig: Microsoft.WSMan.Management\WSMan::localhost\Client

Type            Name                           SourceOfValue   Value
----            ----                           -------------   -----
System.String   TrustedHosts                                   DC-01,SRV-01
```

!!! warning "Poste hors domaine"

    Si votre poste n'est pas joint au meme domaine que les serveurs :

    - Ajoutez les serveurs a `TrustedHosts`
    - Vous devrez fournir des identifiants a chaque connexion
    - Le DNS doit pouvoir resoudre les noms des serveurs

## RSAT vs Windows Admin Center

| Critere | RSAT | Windows Admin Center |
|---------|:----:|:--------------------:|
| Type | Consoles MMC natives | Interface web |
| Installation | Sur chaque poste | Sur un serveur (navigateur) |
| Acces mobile | :material-close: | :material-check: |
| Fonctionnalites | Completes | En evolution |
| Extensions | :material-close: | :material-check: |

!!! tip "Recommandation"

    Utilisez les deux : RSAT pour les taches avancees, Windows Admin Center pour la gestion quotidienne.

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Thomas, administrateur systeme, vient de recevoir un nouveau laptop Windows 11
    pour remplacer son ancien poste. Il doit reconfigurer son environnement d'administration pour
    gerer les serveurs du lab : DC-01 (controleur de domaine), SRV-01 (serveur de fichiers) et
    HV-01 (Hyper-V). Son laptop n'est pas joint au domaine `lab.local`.

    **Probleme** : Apres avoir installe RSAT, il ne parvient pas a ouvrir la console Active Directory
    Users and Computers pour se connecter a DC-01. Le message d'erreur indique "The specified domain
    either does not exist or could not be contacted."

    **Diagnostic** :

    ```powershell
    # Step 1: Check if RSAT AD tools are installed
    Get-WindowsCapability -Online | Where-Object Name -like "Rsat.ActiveDirectory*"

    # Step 2: Test network connectivity to DC-01
    Test-NetConnection -ComputerName DC-01 -Port 389

    # Step 3: Check DNS resolution
    Resolve-DnsName DC-01
    ```

    ```text
    Name  : Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
    State : Installed

    ComputerName     : DC-01
    RemoteAddress    : 10.0.0.10
    RemotePort       : 389
    TcpTestSucceeded : True

    Name     Type TTL  Section IPAddress
    ----     ---- ---  ------- ---------
    DC-01    A    600  Answer  10.0.0.10
    ```

    **Analyse** : La connectivite reseau fonctionne, le DNS resout correctement. Le probleme est que
    le laptop n'est pas joint au domaine et `TrustedHosts` n'est pas configure.

    **Solution** :

    ```powershell
    # Configure TrustedHosts for the lab servers
    Set-Item WSMan:\localhost\Client\TrustedHosts -Value "DC-01,SRV-01,HV-01" -Force

    # Configure the DNS to point to DC-01 for domain resolution
    Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses 10.0.0.10, 8.8.8.8

    # Test the connection with explicit credentials
    $cred = Get-Credential -UserName "lab\Administrateur"
    Test-WSMan -ComputerName DC-01 -Credential $cred
    ```

    ```text
    wsmid           : http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd
    ProtocolVersion : http://schemas.dmtf.org/wbem/wsman/1/wsman.xsd
    ProductVendor   : Microsoft Corporation
    ProductVersion  : OS: 10.0.20348 SP: 0.0 Stack: 3.0
    ```

    **Resultat** : Apres avoir configure le DNS et TrustedHosts, Thomas peut ouvrir `dsa.msc` et
    se connecter a DC-01 en fournissant ses identifiants du domaine `lab.local`.

## Erreurs courantes

!!! danger "Erreurs courantes"

    1. **Installer RSAT sans connexion Internet** : Les fonctionnalites a la demande (Features on Demand)
       necessitent un acces a Windows Update pour le telechargement. Sans Internet, l'installation echoue
       silencieusement. En environnement deconnecte, utilisez un partage WSUS ou une source de
       fonctionnalites locale.

    2. **Oublier de configurer le DNS vers le controleur de domaine** : Si votre poste utilise un DNS
       public (8.8.8.8), il ne pourra pas resoudre les noms du domaine Active Directory. Configurez
       le DNS primaire vers le DC (ex: 10.0.0.10) pour que la resolution de `lab.local` fonctionne.

    3. **Utiliser `TrustedHosts = "*"` en production** : Autoriser toutes les machines dans TrustedHosts
       est acceptable en lab, mais c'est un risque de securite en production. Listez explicitement les
       serveurs autorises.

    4. **Ne pas verifier la version de RSAT apres une mise a jour Windows** : Les mises a jour majeures
       de Windows 10/11 peuvent desinstaller les fonctionnalites RSAT. Apres chaque mise a jour
       semestrielle, verifiez que vos outils sont toujours presents avec
       `Get-WindowsCapability -Online | Where-Object Name -like "Rsat.*"`.

    5. **Confondre RSAT et Windows Admin Center** : RSAT installe les consoles MMC classiques. Windows
       Admin Center est une interface web distincte. Les deux sont complementaires, pas interchangeables.

## Points cles a retenir

- RSAT installe les consoles d'administration sur votre poste Windows 10/11
- Depuis Windows 10 1809, l'installation se fait via les fonctionnalites a la demande
- RSAT est indispensable pour administrer des serveurs Core a distance
- Combinez RSAT + Windows Admin Center pour une gestion optimale

## Pour aller plus loin

- [MMC et snap-ins](mmc-snap-ins.md) - les consoles que RSAT installe
- [Windows Admin Center](../../gestion-moderne/wac/index.md) - alternative web moderne
- [PowerShell Remoting](../../automatisation/powershell-avance/remoting.md) - gestion en ligne de commande
