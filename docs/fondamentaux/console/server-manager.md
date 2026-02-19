---
title: Server Manager
description: Utilisation de Server Manager pour administrer Windows Server 2022.
tags:
  - fondamentaux
  - console
  - debutant
---

# Server Manager

!!! info "Niveau : Debutant"

    Temps estime : 15 minutes

## Presentation

Server Manager est la console d'administration centrale de Windows Server Desktop Experience. Il se lance automatiquement a l'ouverture de session et permet de :

- Visualiser l'etat de sante du serveur
- Ajouter/supprimer des roles et fonctionnalites
- Gerer des serveurs distants depuis un seul point
- Lancer les consoles MMC specialisees

!!! note "Server Core"

    Server Manager n'est pas disponible sur Server Core. Utilisez PowerShell,
    RSAT depuis un poste client, ou Windows Admin Center.

## Interface principale

Le tableau de bord Server Manager se compose de plusieurs zones :

### Menu de navigation (gauche)

| Element | Description |
|---------|-------------|
| **Dashboard** | Vue d'ensemble de tous les roles et serveurs |
| **Local Server** | Proprietes et configuration du serveur local |
| **All Servers** | Liste de tous les serveurs geres |
| **Roles installes** | Un noeud par role (AD DS, DNS, DHCP, etc.) |

### Panneau Local Server

Le panneau **Local Server** affiche les proprietes essentielles en un coup d'oeil :

- Nom de l'ordinateur
- Domaine / Groupe de travail
- Adresse IP
- Etat de Windows Update
- Etat du pare-feu
- Bureau a distance
- Configuration NIC Teaming
- Fuseau horaire

!!! tip "Astuce"

    Chaque propriete est cliquable : cliquez dessus pour la modifier directement.

## Ajout de roles et fonctionnalites

### Via Server Manager

1. Cliquer sur **Manage** > **Add Roles and Features**
2. Choisir le type d'installation : **Role-based or feature-based**
3. Selectionner le serveur cible
4. Cocher les roles souhaites
5. Ajouter les fonctionnalites requises (dependances automatiques)
6. Confirmer et installer

### Via PowerShell (equivalent)

```powershell
# List all available roles and features
Get-WindowsFeature

# List only installed features
Get-WindowsFeature | Where-Object Installed

# Install a role (example: DNS Server)
Install-WindowsFeature -Name DNS -IncludeManagementTools

# Install multiple roles at once
Install-WindowsFeature -Name AD-Domain-Services, DNS, DHCP -IncludeManagementTools

# Remove a role
Uninstall-WindowsFeature -Name DHCP -Remove
```

!!! warning "Flag -IncludeManagementTools"

    N'oubliez pas `-IncludeManagementTools` pour installer les consoles de gestion
    associees au role. Sans ce flag, le role est installe mais sans outils d'administration.

## Gestion de serveurs distants

Server Manager peut gerer plusieurs serveurs depuis une seule console :

### Ajouter un serveur distant

1. Cliquer sur **Manage** > **Add Servers**
2. Rechercher par nom, IP ou dans Active Directory
3. Ajouter les serveurs souhaites

### Prerequis pour la gestion a distance

```powershell
# On the remote server, enable remote management
Enable-PSRemoting -Force
winrm quickconfig -q

# On the remote server, enable Server Manager remote management
Configure-SMRemoting.exe -Enable

# Verify WinRM connectivity from the management server
Test-WSMan -ComputerName SRV-REMOTE-01
```

### Groupes de serveurs

Creez des groupes logiques pour organiser vos serveurs :

1. Clic droit sur **All Servers** > **Create Server Group**
2. Nommer le groupe (ex: "Controleurs de domaine", "Serveurs Web")
3. Ajouter les serveurs concernes

## Notifications et alertes

Le drapeau de notification dans Server Manager signale :

| Icone | Signification |
|-------|---------------|
| :material-flag: Vert | Tout est normal |
| :material-flag: Jaune | Avertissement (mise a jour en attente, configuration requise) |
| :material-flag: Rouge | Erreur critique (service arrete, role en echec) |

!!! tip "Post-installation"

    Apres l'installation d'un role, verifiez toujours les notifications.
    Certains roles (comme AD DS) necessitent une configuration supplementaire
    post-installation, signalee par un drapeau jaune.

## Desactiver le lancement automatique

Si vous preferez que Server Manager ne se lance pas a l'ouverture de session :

=== "Via Server Manager"

    1. **Manage** > **Server Manager Properties**
    2. Cocher **Do not start Server Manager automatically at logon**

=== "Via PowerShell"

    ```powershell
    # Disable auto-start for current user
    Set-ItemProperty `
        -Path "HKCU:\Software\Microsoft\ServerManager" `
        -Name "DoNotOpenServerManagerAtLogon" `
        -Value 1
    ```

## Limites de Server Manager

- Ne peut pas tout faire (certaines configurations avancees necessitent MMC ou PowerShell)
- Interface parfois lente sur des serveurs charges
- Non disponible sur Server Core
- Progressivement remplace par **Windows Admin Center** pour la gestion moderne

## Points cles a retenir

- Server Manager est le point d'entree principal sur Desktop Experience
- Il permet d'ajouter des roles, de surveiller l'etat et de gerer des serveurs distants
- Chaque action dans Server Manager a un equivalent PowerShell
- Pour Server Core, utilisez RSAT ou Windows Admin Center

## Pour aller plus loin

- [MMC et snap-ins](mmc-snap-ins.md) - consoles de gestion specialisees
- [RSAT](rsat.md) - outils d'administration a distance
- [Windows Admin Center](../../gestion-moderne/wac/index.md) - alternative web moderne
