---
title: Server Core vs Desktop Experience
description: Comparaison entre l'installation Server Core et Desktop Experience de Windows Server 2022.
tags:
  - fondamentaux
  - installation
  - debutant
---

# Server Core vs Desktop Experience

<span class="level-beginner">Debutant</span> · Temps estime : 15 minutes

## Les deux modes d'installation

!!! example "Analogie"

    Imaginez deux cuisines de restaurant. **Server Core**, c'est la cuisine professionnelle : pas de
    decoration, pas de television, juste les ustensiles essentiels. Le chef travaille vite et efficacement.
    **Desktop Experience**, c'est une cuisine ouverte avec ecran, musique et eclairage decoratif :
    plus agreable pour apprendre, mais plus de choses a entretenir et a nettoyer.

Lors de l'installation de Windows Server 2022, vous choisissez entre deux experiences :

```mermaid
graph TD
    A[Installation Windows Server 2022] --> B[Server Core]
    A --> C[Desktop Experience]
    B --> D[Ligne de commande uniquement<br/>PowerShell + sconfig]
    C --> E[Interface graphique complete<br/>Explorateur, Server Manager, MMC]
```

## Comparatif detaille

| Critere | Server Core | Desktop Experience |
|---------|:-----------:|:------------------:|
| Interface graphique | :material-close: | :material-check: |
| Empreinte disque | ~6 Go | ~10 Go |
| Empreinte RAM | ~512 Mo | ~1.5 Go |
| Surface d'attaque | Reduite | Plus large |
| Mises a jour | Moins frequentes | Plus frequentes |
| Redemarrages | Moins nombreux | Plus nombreux |
| Gestion locale | PowerShell, sconfig | GUI + PowerShell |
| Gestion a distance | WAC, RSAT, PowerShell | WAC, RSAT, PowerShell, RDP |
| Roles supportes | La plupart | Tous |

## Server Core

### Avantages

- **Securite renforcee** : moins de composants = moins de vulnerabilites
- **Performance** : moins de ressources consommees par l'OS
- **Stabilite** : moins de mises a jour = moins de redemarrages
- **Recommandation Microsoft** : c'est le mode par defaut depuis Windows Server 2016

### Outils de gestion disponibles

#### sconfig

L'outil `sconfig` fournit un menu texte pour les taches courantes :

```
===============================================================================
                         Server Configuration
===============================================================================

 1) Domain/Workgroup:                    Workgroup: WORKGROUP
 2) Computer Name:                       SRV-CORE-01
 3) Add Local Administrator
 4) Configure Remote Management          Enabled
 5) Windows Update Settings:             Manual
 6) Download and Install Updates
 7) Remote Desktop:                      Disabled
 8) Network Settings
 9) Date and Time
10) Telemetry settings
11) Windows Activation
12) Log Off User
13) Restart Server
14) Shut Down Server
15) Exit to Command Line
```

#### PowerShell

Toute l'administration se fait en PowerShell :

```powershell
# System information
Get-ComputerInfo | Select-Object WindowsProductName, OsVersion

# Network configuration
Get-NetIPConfiguration

# Installed roles and features
Get-WindowsFeature | Where-Object Installed

# Service management
Get-Service | Where-Object Status -eq "Running"

# Event logs
Get-EventLog -LogName System -Newest 20
```

Resultat de `Get-ComputerInfo` :

```text
WindowsProductName                    OsVersion
------------------                    ---------
Windows Server 2022 Datacenter        10.0.20348
```

Resultat de `Get-WindowsFeature | Where-Object Installed` (extrait) :

```text
Display Name                                    Name                Install State
------------                                    ----                -------------
[X] File and Storage Services                   FileAndStorage-Se.. Installed
    [X] Storage Services                        Storage-Services    Installed
[X] .NET Framework 4.8 Features                 NET-Framework-45..  Installed
[X] PowerShell                                  PowerShellRoot      Installed
    [X] Windows PowerShell 5.1                  PowerShell          Installed
[X] WoW64 Support                               WoW64-Support       Installed
```

#### Gestion a distance

!!! tip "Best practice"

    Gerez les serveurs Core a distance avec :

    - **Windows Admin Center** (WAC) : interface web moderne
    - **RSAT** : outils d'administration depuis un poste Windows 10/11
    - **PowerShell Remoting** : automatisation et scripts

```powershell
# Connect to a remote Core server from another machine
Enter-PSSession -ComputerName SRV-CORE-01 -Credential (Get-Credential)

# Run a command on a remote Core server
Invoke-Command -ComputerName SRV-CORE-01 -ScriptBlock {
    Get-WindowsFeature | Where-Object Installed
}
```

Resultat de `Enter-PSSession` :

```text
[SRV-CORE-01]: PS C:\Users\Administrator\Documents>
```

Resultat de `Invoke-Command` (extrait) :

```text
Display Name                           Name              Install State PSComputerName
------------                           ----              ------------- --------------
[X] DNS Server                         DNS               Installed     SRV-CORE-01
[X] File and Storage Services          FileAndStorage-Se..Installed     SRV-CORE-01
    [X] Storage Services               Storage-Services  Installed     SRV-CORE-01
[X] PowerShell                         PowerShellRoot    Installed     SRV-CORE-01
```

### Roles non disponibles en Server Core

Quelques roles necessitent Desktop Experience :

- Fax Server
- MultiPoint Services
- Windows Deployment Services (WDS) - interface graphique uniquement

!!! note "Remarque"

    Tous les roles principaux (AD DS, DNS, DHCP, Hyper-V, File Server, IIS, etc.)
    sont disponibles en Server Core.

## Desktop Experience

### Avantages

- **Apprentissage** : plus facile pour les debutants
- **Depannage visuel** : acces direct aux outils graphiques
- **Applications tierces** : certaines applications necessitent une interface graphique
- **Compatibilite** : tous les roles et fonctionnalites sont disponibles

### Quand choisir Desktop Experience

- Serveur d'administration central (jump server)
- Environnement de lab et d'apprentissage
- Applications qui exigent une interface graphique
- Transition progressive depuis des environnements anciens

## Recommandations par role

| Role serveur | Recommandation | Raison |
|-------------|:--------------:|--------|
| Controleur de domaine (DC) | Core | Securite, pas besoin de GUI |
| Serveur DNS / DHCP | Core | Services reseau simples |
| Serveur de fichiers | Core | Gere facilement via RSAT/WAC |
| Hyper-V | Core | Performance maximale pour la virtualisation |
| IIS / Web | Core | Securite, moins de surface d'attaque |
| Serveur d'administration | GUI | Acces aux outils graphiques |
| SQL Server | GUI | Certains outils SQL necessitent une GUI |
| Lab / Formation | GUI | Facilite l'apprentissage |

## Conversion entre les modes

!!! danger "Important"

    Depuis Windows Server 2019, il n'est **plus possible** de convertir
    entre Server Core et Desktop Experience apres l'installation.
    Sous Windows Server 2016 et anterieur, la conversion etait possible
    avec `Install-WindowsFeature` / `Uninstall-WindowsFeature`.

    Si vous devez changer de mode, vous devez **reinstaller** le serveur.

## Strategie hybride recommandee

Pour un environnement de production, combinez les deux :

```mermaid
graph TD
    A[Poste admin Windows 11<br/>avec RSAT + WAC] --> B[DC-01<br/>Server Core]
    A --> C[DC-02<br/>Server Core]
    A --> D[FS-01<br/>Server Core]
    A --> E[HV-01<br/>Server Core]
    A --> F[ADMIN-01<br/>Desktop Experience<br/>Jump server]
    F --> B
    F --> C
    F --> D
    F --> E
```

!!! tip "Approche recommandee"

    - Deployer les serveurs d'infrastructure en **Server Core**
    - Avoir **un seul** serveur Desktop Experience comme poste d'administration
    - Installer **RSAT** sur votre poste Windows 10/11 pour la gestion quotidienne
    - Utiliser **Windows Admin Center** comme interface web centralisee

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Claire, administratrice systeme, doit deployer 5 nouveaux serveurs dans son
    infrastructure : 2 controleurs de domaine, 1 serveur de fichiers, 1 serveur Hyper-V et 1 serveur
    d'administration. Son budget est serre et elle veut minimiser la maintenance.

    **Probleme** : Elle hesite a tout deployer en Desktop Experience pour simplifier la gestion.

    **Analyse** :

    1. Comparer les ressources consommees par les deux modes :

    ```powershell
    # Check disk usage on a Core server
    Invoke-Command -ComputerName SRV-CORE-01 -ScriptBlock {
        Get-PSDrive C | Select-Object Used, Free
    }

    # Check disk usage on a GUI server
    Invoke-Command -ComputerName SRV-GUI-01 -ScriptBlock {
        Get-PSDrive C | Select-Object Used, Free
    }
    ```

    ```text
    # SRV-CORE-01 (Server Core)
         Used        Free
         ----        ----
    6442450944  26843545600

    # SRV-GUI-01 (Desktop Experience)
         Used         Free
         ----         ----
    10737418240  22548578304
    ```

    2. La difference est significative : ~6 Go vs ~10 Go pour l'OS seul.
    3. Avec 5 serveurs, cela represente 20 Go d'espace economise et moins de mises a jour.

    **Decision** :

    | Serveur | Mode | Justification |
    |---------|------|---------------|
    | DC-01, DC-02 | Core | Securite maximale pour les DCs |
    | FS-01 | Core | Service de fichiers gere par RSAT |
    | HV-01 | Core | Performance maximale pour Hyper-V |
    | ADMIN-01 | GUI | Poste d'administration avec outils graphiques |

    **Resultat** : 4 serveurs en Core, 1 en GUI. Claire installe RSAT sur ADMIN-01 et Windows Admin
    Center pour gerer tous les serveurs depuis un seul point.

## Erreurs courantes

!!! danger "Erreurs courantes"

    1. **Tenter de convertir Core en GUI apres installation** : Depuis Windows Server 2019, la conversion
       n'est plus possible. Si vous vous etes trompe, il faut reinstaller le serveur. Choisissez bien
       avant l'installation.

    2. **Deployer Desktop Experience sur tous les serveurs** : Chaque serveur GUI consomme plus de
       RAM, plus de disque et necessite plus de mises a jour. En production, cela multiplie la charge
       de maintenance inutilement.

    3. **Ne pas preparer la gestion a distance avant de deployer Core** : Avant de deployer un serveur
       Core, assurez-vous que votre poste d'administration a RSAT installe et que WinRM est configure.
       Sans cela, vous serez bloque avec seulement `sconfig` et PowerShell local.

    4. **Croire que Server Core ne supporte aucun role** : Core supporte la grande majorite des roles
       (AD DS, DNS, DHCP, Hyper-V, IIS, File Server, etc.). Seuls quelques roles marginaux comme
       Fax Server ou MultiPoint Services necessitent Desktop Experience.

## Points cles a retenir

- Server Core est le mode recommande par Microsoft pour la production
- Desktop Experience offre le confort graphique au prix de plus de ressources et de mises a jour
- La conversion entre les deux modes est impossible depuis Windows Server 2019
- La gestion a distance (RSAT, WAC, PowerShell Remoting) rend Server Core aussi simple a gerer que la GUI

## Pour aller plus loin

- [RSAT](../console/rsat.md) - outils d'administration a distance
- [Windows Admin Center](../../gestion-moderne/wac/index.md) - gestion web centralisee
- [PowerShell - Premiers pas](../powershell/premiers-pas.md) - indispensable pour Server Core
