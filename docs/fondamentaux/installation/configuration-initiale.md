---
title: Configuration initiale
description: Etapes de configuration post-installation de Windows Server 2022.
tags:
  - fondamentaux
  - installation
  - debutant
---

# Configuration initiale

!!! info "Niveau : Debutant"

    Temps estime : 20 minutes | Lab associe : [Lab 01](../../labs/exercices/lab-01-installation.md)

## Introduction

Apres l'installation de Windows Server 2022, plusieurs etapes de configuration sont necessaires avant de mettre le serveur en production.

## Etape 1 : Renommer le serveur

Par defaut, Windows attribue un nom aleatoire (ex: `WIN-A1B2C3D4E5F6`). Il est essentiel de choisir un nom explicite.

!!! tip "Convention de nommage"

    Adoptez une convention claire, par exemple : `<ROLE>-<SITE>-<NUMERO>`

    - `DC-PARIS-01` : controleur de domaine a Paris
    - `FS-LYON-01` : serveur de fichiers a Lyon
    - `WEB-PARIS-01` : serveur web a Paris

=== "PowerShell"

    ```powershell
    # Display the current hostname
    hostname

    # Rename the server (requires restart)
    Rename-Computer -NewName "SRV-LAB-01" -Restart
    ```

=== "Server Manager"

    1. Ouvrir **Server Manager**
    2. Cliquer sur le nom du serveur dans le tableau de bord
    3. Cliquer sur **Modifier** dans la fenetre des proprietes systeme
    4. Saisir le nouveau nom et redemarrer

## Etape 2 : Configurer une adresse IP statique

Un serveur doit toujours avoir une adresse IP statique pour etre fiable.

=== "PowerShell"

    ```powershell
    # List network adapters
    Get-NetAdapter

    # Remove DHCP configuration
    Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Disabled

    # Set static IP address
    New-NetIPAddress `
        -InterfaceAlias "Ethernet" `
        -IPAddress 192.168.10.10 `
        -PrefixLength 24 `
        -DefaultGateway 192.168.10.1

    # Set DNS servers
    Set-DnsClientServerAddress `
        -InterfaceAlias "Ethernet" `
        -ServerAddresses 192.168.10.10, 8.8.8.8

    # Verify the configuration
    Get-NetIPConfiguration
    ```

=== "Interface graphique"

    1. Ouvrir **Parametres reseau** > **Modifier les options d'adaptateur**
    2. Clic droit sur la carte reseau > **Proprietes**
    3. Double-cliquer sur **Protocole Internet version 4 (TCP/IPv4)**
    4. Selectionner **Utiliser l'adresse IP suivante**
    5. Remplir adresse IP, masque, passerelle et DNS

!!! warning "Attention"

    Ne jamais laisser un serveur en DHCP en production, sauf cas tres specifiques (clusters avec reservation DHCP).

## Etape 3 : Configurer le fuseau horaire

Un horaire incorrect peut causer des problemes d'authentification Kerberos dans Active Directory.

```powershell
# Display current time zone
Get-TimeZone

# Set time zone (example: Paris)
Set-TimeZone -Id "Romance Standard Time"

# List all available time zones
Get-TimeZone -ListAvailable | Where-Object { $_.DisplayName -like "*Paris*" }

# Configure NTP server
w32tm /config /manualpeerlist:"time.windows.com" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time
w32tm /resync
```

!!! danger "Kerberos et le temps"

    Active Directory utilise Kerberos pour l'authentification. Kerberos tolere un ecart maximum
    de **5 minutes** entre le client et le serveur. Un decalage superieur bloque les connexions.

## Etape 4 : Activer le Bureau a distance (RDP)

```powershell
# Enable Remote Desktop
Set-ItemProperty `
    -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
    -Name "fDenyTSConnections" `
    -Value 0

# Enable the firewall rule for RDP
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Optional: allow only NLA (Network Level Authentication) - recommended
Set-ItemProperty `
    -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' `
    -Name "UserAuthentication" `
    -Value 1
```

!!! tip "Bonne pratique"

    Activez toujours **NLA (Network Level Authentication)** pour securiser les connexions RDP.
    NLA exige que l'utilisateur s'authentifie avant d'etablir la session graphique.

## Etape 5 : Configurer Windows Update

```powershell
# Check for available updates
Install-Module PSWindowsUpdate -Force -Scope CurrentUser
Import-Module PSWindowsUpdate
Get-WindowsUpdate

# Install all available updates
Install-WindowsUpdate -AcceptAll -AutoReboot

# Configure automatic updates via Group Policy (sconfig on Core)
# Or use sconfig option 6 on Server Core
```

=== "Server Core"

    ```powershell
    # Use the sconfig menu
    sconfig
    # Select option 6: Download and Install Updates
    ```

=== "Desktop Experience"

    1. Ouvrir **Parametres** > **Windows Update**
    2. Cliquer sur **Rechercher les mises a jour**
    3. Installer toutes les mises a jour disponibles

## Etape 6 : Configurer le pare-feu

Par defaut, le pare-feu Windows est active. Verifiez sa configuration :

```powershell
# Display firewall status for all profiles
Get-NetFirewallProfile | Select-Object Name, Enabled

# List active inbound rules
Get-NetFirewallRule -Direction Inbound -Enabled True |
    Select-Object DisplayName, Profile, Action |
    Sort-Object DisplayName
```

!!! danger "Ne jamais desactiver le pare-feu en production"

    En lab, il peut etre tentant de desactiver le pare-feu pour simplifier les tests.
    En production, creez plutot des regles specifiques pour les services necessaires.

## Etape 7 : Activer la gestion a distance

```powershell
# Enable PowerShell Remoting
Enable-PSRemoting -Force

# Enable WinRM for remote management
winrm quickconfig -q

# Verify WinRM is running
Get-Service WinRM

# Test remote connectivity from another machine
# Test-WSMan -ComputerName SRV-LAB-01
```

## Checklist de configuration initiale

- [ ] Serveur renomme selon la convention de nommage
- [ ] Adresse IP statique configuree
- [ ] DNS configure (pointe vers le futur DC ou DNS externe)
- [ ] Fuseau horaire correct
- [ ] Bureau a distance active avec NLA
- [ ] Mises a jour installees
- [ ] Pare-feu configure (regles adaptees)
- [ ] PowerShell Remoting active
- [ ] Licence activee ou periode d'evaluation verifiee

## Pour aller plus loin

- [Server Core vs GUI](server-core-vs-gui.md) - gestion sans interface graphique
- [Server Manager](../console/server-manager.md) - outil de gestion centralise
- [Lab 01 : Installation](../../labs/exercices/lab-01-installation.md) - mise en pratique
