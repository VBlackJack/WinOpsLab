---
title: "Lab 01 : Installation de Windows Server"
description: Exercice pratique - installer et configurer un premier serveur Windows Server 2022.
tags:
  - lab
  - fondamentaux
  - debutant
---

# Lab 01 : Installation de Windows Server

!!! abstract "Objectifs du lab"

    - [ ] Creer une machine virtuelle Hyper-V
    - [ ] Installer Windows Server 2022 (Desktop Experience)
    - [ ] Installer Windows Server 2022 (Server Core)
    - [ ] Effectuer la configuration post-installation

## Scenario

Vous etes administrateur systeme junior dans une PME. On vous demande de mettre en place
deux nouveaux serveurs : un serveur avec interface graphique pour la gestion quotidienne,
et un serveur Core pour les services reseau.

## Environnement requis

| Ressource | Specification |
|-----------|---------------|
| VM 1 (GUI) | 2 vCPU, 4 Go RAM, 60 Go disque |
| VM 2 (Core) | 2 vCPU, 2 Go RAM, 40 Go disque |
| Reseau | Switch virtuel interne |
| ISO | Windows Server 2022 Evaluation |

```mermaid
flowchart TD
    A["Creer le switch<br/>virtuel LabSwitch"] --> B["Creer VM 1<br/>SRV-GUI<br/>4 Go RAM / 60 Go"]
    A --> C["Creer VM 2<br/>SRV-CORE<br/>2 Go RAM / 40 Go"]
    B --> D["Installer Windows Server<br/>Desktop Experience"]
    C --> E["Installer Windows Server<br/>Server Core"]
    D --> F["Configuration post-installation"]
    E --> F
    F --> G["Renommer le serveur"]
    G --> H["Configurer IP statique"]
    H --> I["Configurer DNS"]
    I --> J["Activer gestion<br/>a distance"]
```

## Instructions

### Partie 1 : Creer les machines virtuelles

1. Ouvrir le Gestionnaire Hyper-V
2. Creer un switch virtuel interne nomme `LabSwitch`
3. Creer la VM 1 (GUI) avec les specifications ci-dessus
4. Creer la VM 2 (Core) avec les specifications ci-dessus

```powershell
# Create the virtual switch
New-VMSwitch -Name "LabSwitch" -SwitchType Internal

# Create VM 1 - Desktop Experience
New-VM -Name "SRV-GUI" -MemoryStartupBytes 4GB -NewVHDPath "G:\VMs\SRV-GUI.vhdx" -NewVHDSizeBytes 60GB -Generation 2 -SwitchName "LabSwitch"
Set-VM -Name "SRV-GUI" -ProcessorCount 2

# Create VM 2 - Server Core
New-VM -Name "SRV-CORE" -MemoryStartupBytes 2GB -NewVHDPath "G:\VMs\SRV-CORE.vhdx" -NewVHDSizeBytes 40GB -Generation 2 -SwitchName "LabSwitch"
Set-VM -Name "SRV-CORE" -ProcessorCount 2
```

### Partie 2 : Installer Windows Server (Desktop Experience)

1. Monter l'ISO sur la VM `SRV-GUI`
2. Demarrer la VM et suivre l'assistant d'installation
3. Choisir **Windows Server 2022 Standard (Desktop Experience)**
4. Definir le mot de passe administrateur

### Partie 3 : Installer Windows Server (Server Core)

1. Monter l'ISO sur la VM `SRV-CORE`
2. Demarrer la VM et suivre l'assistant d'installation
3. Choisir **Windows Server 2022 Standard** (sans Desktop Experience)
4. Definir le mot de passe administrateur

### Partie 4 : Configuration post-installation

Sur chaque serveur, effectuer les taches suivantes :

??? success "Solution - SRV-GUI"

    ```powershell
    # Rename the server
    Rename-Computer -NewName "SRV-GUI" -Restart

    # After restart, configure static IP
    New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.10 -PrefixLength 24 -DefaultGateway 192.168.10.1
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.10.10

    # Disable the firewall for lab purposes (not in production!)
    # Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

    # Enable Remote Desktop
    Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
    Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
    ```

??? success "Solution - SRV-CORE"

    ```powershell
    # Use sconfig for initial configuration
    sconfig

    # Or via PowerShell:
    Rename-Computer -NewName "SRV-CORE" -Restart

    # After restart
    New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.10.11 -PrefixLength 24 -DefaultGateway 192.168.10.1
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.10.10

    # Enable remote management
    Enable-PSRemoting -Force
    ```

## Verification

!!! question "Questions de validation"

    1. Quelle commande permet de verifier la version de Windows Server installee ?
    2. Quelle est la difference principale entre Desktop Experience et Server Core ?
    3. Quel outil en ligne de commande permet de configurer rapidement un Server Core ?
    4. Pourquoi est-il important de renommer le serveur avant de le joindre a un domaine ?

??? success "Reponses"

    1. `Get-ComputerInfo | Select-Object WindowsProductName, OsVersion` ou `winver`
    2. Desktop Experience inclut l'interface graphique complete (Explorateur, Server Manager, etc.).
       Server Core est une installation minimale sans GUI, plus legere et plus securisee.
    3. `sconfig` - un menu texte interactif pour les taches de configuration courantes.
    4. Le renommage apres jonction au domaine necessite des operations supplementaires sur le DC.
       Il est plus simple de renommer le serveur avant de le joindre.

## Nettoyage

Pour conserver les VMs pour les labs suivants, ne les supprimez pas.
Si vous souhaitez repartir de zero :

```powershell
# Stop and remove VMs
Stop-VM -Name "SRV-GUI", "SRV-CORE" -Force -ErrorAction SilentlyContinue
Remove-VM -Name "SRV-GUI", "SRV-CORE" -Force
Remove-Item "G:\VMs\SRV-GUI.vhdx", "G:\VMs\SRV-CORE.vhdx" -ErrorAction SilentlyContinue
```

## Prochaine etape

:material-arrow-right: [Lab 02 : Premier domaine AD DS](lab-02-ad-ds-premier-domaine.md)
