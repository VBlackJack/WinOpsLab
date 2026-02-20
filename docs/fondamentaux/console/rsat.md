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

## Points cles a retenir

- RSAT installe les consoles d'administration sur votre poste Windows 10/11
- Depuis Windows 10 1809, l'installation se fait via les fonctionnalites a la demande
- RSAT est indispensable pour administrer des serveurs Core a distance
- Combinez RSAT + Windows Admin Center pour une gestion optimale

## Pour aller plus loin

- [MMC et snap-ins](mmc-snap-ins.md) - les consoles que RSAT installe
- [Windows Admin Center](../../gestion-moderne/wac/index.md) - alternative web moderne
- [PowerShell Remoting](../../automatisation/powershell-avance/remoting.md) - gestion en ligne de commande
