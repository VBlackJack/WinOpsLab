---
title: "Installation de Windows Admin Center"
description: Installer et configurer Windows Admin Center sur Windows Server 2022 - mode gateway, mode desktop, certificat et premier acces.
tags:
  - gestion-moderne
  - wac
  - intermediaire
---

# Installation de Windows Admin Center

<span class="level-intermediate">Intermediaire</span> · Temps estime : 25 minutes

## Presentation

Windows Admin Center (WAC) est l'outil de gestion graphique moderne de Microsoft pour Windows Server. Il fonctionne dans un navigateur web et remplace progressivement les consoles MMC traditionnelles (Server Manager, Hyper-V Manager, etc.).

```mermaid
graph TD
    A[Windows Admin Center] --> B[Mode Gateway]
    A --> C[Mode Desktop]
    B --> D[Serveur dedie<br/>Acces multi-utilisateurs]
    C --> E[Poste d'administration<br/>Usage individuel]
    B --> F[Port 443 par defaut]
    C --> G[Port 6516 par defaut]
```

## Modes d'installation

| Mode | Cible | Port par defaut | Usage |
|------|-------|----------------|-------|
| **Gateway** | Windows Server (Core ou GUI) | TCP 443 | Production - acces multi-utilisateurs via navigateur |
| **Desktop** | Windows 10/11 | TCP 6516 | Poste d'administration individuel |

!!! tip "Recommandation"

    En production, installez WAC en **mode Gateway** sur un serveur dedie
    (pas sur un controleur de domaine). Cela permet a toute l'equipe d'administration
    d'y acceder depuis n'importe quel navigateur.

## Prerequis

| Element | Specification |
|---------|---------------|
| OS | Windows Server 2016+ (Gateway) ou Windows 10/11 (Desktop) |
| RAM | 2 Go minimum |
| Navigateur | Microsoft Edge ou Google Chrome |
| Reseau | Connectivite vers les serveurs a gerer (WinRM, ports 5985/5986) |
| Certificat | Auto-signe (test) ou certificat d'une CA (production) |

## Installation en mode Gateway

### Telechargement

Telecharger Windows Admin Center depuis le site officiel de Microsoft.

### Installation via l'interface

1. Executer le fichier `.msi` telecharge
2. Accepter les termes de licence
3. Cocher **Use Microsoft Update** (recommande)
4. Configurer les options :
    - **Port** : 443 (ou un port personnalise)
    - **Certificat** : generer un certificat auto-signe ou selectionner un certificat existant
    - **Autoriser WAC a modifier les parametre TrustedHosts** : cocher pour les serveurs hors domaine
5. Cliquer sur **Install**

### Installation silencieuse (PowerShell)

```powershell
# Silent install with self-signed certificate on port 443
msiexec /i "WindowsAdminCenter.msi" /qn /L*v "C:\Logs\wac-install.log" `
    SME_PORT=443 `
    SSL_CERTIFICATE_OPTION=generate

# Silent install with a specific certificate thumbprint
msiexec /i "WindowsAdminCenter.msi" /qn /L*v "C:\Logs\wac-install.log" `
    SME_PORT=443 `
    SME_THUMBPRINT=<thumbprint-du-certificat>

# Silent install on a custom port
msiexec /i "WindowsAdminCenter.msi" /qn /L*v "C:\Logs\wac-install.log" `
    SME_PORT=8443 `
    SSL_CERTIFICATE_OPTION=generate
```

### Verification post-installation

```powershell
# Verify the WAC service is running
Get-Service ServerManagementGateway | Select-Object Name, Status, StartType

# Check the listening port
Get-NetTCPConnection -LocalPort 443 -State Listen -ErrorAction SilentlyContinue

# Verify the WAC URL is accessible
Test-NetConnection -ComputerName localhost -Port 443
```

## Configuration du certificat

### Certificat auto-signe (test/lab)

L'installation genere un certificat auto-signe valide 60 jours. Les navigateurs afficheront un avertissement de securite.

### Certificat d'une autorite de certification (production)

```powershell
# Request a certificate from an enterprise CA
$certParams = @{
    Subject           = "CN=wac.winopslab.local"
    DnsName           = "wac.winopslab.local", "SRV-WAC"
    CertStoreLocation = "Cert:\LocalMachine\My"
    KeyExportPolicy   = "Exportable"
    KeyLength         = 2048
    KeyAlgorithm      = "RSA"
    HashAlgorithm     = "SHA256"
    NotAfter          = (Get-Date).AddYears(2)
    KeyUsage          = "DigitalSignature", "KeyEncipherment"
    Template          = "WebServer"
}
$cert = Get-Certificate @certParams

# Get the thumbprint
$cert.Certificate.Thumbprint
```

Pour changer le certificat apres l'installation :

```powershell
# Update WAC to use a new certificate
# Method 1: Reinstall with the new thumbprint
msiexec /i "WindowsAdminCenter.msi" /qn SME_PORT=443 SME_THUMBPRINT=<new-thumbprint>

# Method 2: Update via netsh (advanced)
netsh http delete sslcert ipport=0.0.0.0:443
netsh http add sslcert ipport=0.0.0.0:443 certhash=<new-thumbprint> appid='{00000000-0000-0000-0000-000000000000}'
```

## Configuration du pare-feu

```powershell
# WAC port (should be opened automatically during install)
New-NetFirewallRule -DisplayName "Windows Admin Center" `
    -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# WinRM ports for managing remote servers
New-NetFirewallRule -DisplayName "WinRM HTTP" `
    -Direction Inbound -Protocol TCP -LocalPort 5985 -Action Allow
New-NetFirewallRule -DisplayName "WinRM HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 5986 -Action Allow
```

## Premier acces

1. Ouvrir un navigateur (Edge ou Chrome) sur un poste d'administration
2. Naviguer vers `https://SRV-WAC.winopslab.local` (ou l'IP du serveur)
3. Si certificat auto-signe : accepter l'avertissement de securite
4. S'authentifier avec un compte administrateur du domaine
5. La page d'accueil affiche la liste des connexions disponibles

### Navigateurs supportes

| Navigateur | Support |
|------------|---------|
| Microsoft Edge | :material-check: Recommande |
| Google Chrome | :material-check: Supporte |
| Mozilla Firefox | :material-check: Supporte (certaines limitations) |
| Internet Explorer | :material-close: Non supporte |

## Controle d'acces

### Roles d'acces WAC

| Role | Permissions |
|------|------------|
| **Gateway Administrators** | Gestion complete de WAC (utilisateurs, extensions, parametres) |
| **Gateway Users** | Acces aux connexions et outils (gestion des serveurs) |

```powershell
# Configure access via WAC Settings > Access
# Or via PowerShell on the WAC server:

# Add a group as Gateway Users
Add-LocalGroupMember -Group "Windows Admin Center Gateway Users" -Member "WINOPSLAB\AdminsIT"

# Add a group as Gateway Administrators
Add-LocalGroupMember -Group "Windows Admin Center Gateway Administrators" -Member "WINOPSLAB\AdminsSenior"
```

### Integration avec Active Directory

En environnement de domaine, WAC utilise l'authentification Kerberos. Les groupes AD peuvent etre ajoutes directement aux roles WAC.

## Mise a jour de WAC

```powershell
# Check current version
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\ServerManagementGateway" |
    Select-Object Version

# Update: download the new MSI and run the installer
# WAC preserves settings and connections during the upgrade
msiexec /i "WindowsAdminCenter-NEW.msi" /qn /L*v "C:\Logs\wac-upgrade.log" SME_PORT=443
```

## Points cles a retenir

- Windows Admin Center est l'outil de gestion moderne basee sur navigateur pour Windows Server
- Le mode **Gateway** (sur un serveur dedie, port 443) est recommande en production
- Un certificat d'une CA interne est fortement recommande pour eviter les avertissements de securite
- WAC utilise **WinRM** (ports 5985/5986) pour communiquer avec les serveurs geres
- Les roles **Gateway Administrators** et **Gateway Users** controlent l'acces a l'interface
- La mise a jour se fait simplement en relancant l'installeur avec la nouvelle version

## Pour aller plus loin

- [Gestion des serveurs via WAC](gestion-serveurs.md) pour decouvrir les outils disponibles
- [Extensions WAC](extensions.md) pour etendre les fonctionnalites
