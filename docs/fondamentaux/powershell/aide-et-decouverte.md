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
title: Aide et decouverte
description: Utiliser le systeme d'aide PowerShell pour explorer et apprendre par soi-meme.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Aide et decouverte

<span class="level-beginner">Debutant</span> Â· Temps estime : 15 minutes

## Le systeme d'aide integre

!!! example "Analogie"

    Le systeme d'aide de PowerShell fonctionne comme le mode d'emploi integre d'un appareil electromenager. Au lieu de chercher la notice sur Internet, vous appuyez sur un bouton et l'appareil vous explique lui-meme comment il fonctionne. `Get-Help` est ce bouton.

PowerShell dispose d'un systeme d'aide complet qui vous permet d'apprendre n'importe quelle cmdlet sans quitter la console.

```mermaid
flowchart LR
    Q(["Cmdlet<br/>inconnue ?"])
    CMD["<b>Get-Command</b><br/>Trouver la cmdlet"]
    HELP["<b>Get-Help</b><br/>Syntaxe et exemples"]
    MEMBER["<b>Get-Member</b><br/>Proprietes de l'objet"]
    USE["Pipeline vers<br/>Format / Export /<br/>Where / Select"]

    Q --> CMD
    CMD --> HELP
    HELP --> MEMBER
    MEMBER --> USE

    style Q fill:#ffa726,color:#fff,stroke:#fb8c00
    style CMD fill:#1565c0,color:#fff,stroke:#0d47a1
    style HELP fill:#42a5f5,color:#fff,stroke:#1e88e5
    style MEMBER fill:#7e57c2,color:#fff,stroke:#5e35b1
    style USE fill:#66bb6a,color:#fff,stroke:#43a047
```

### Mettre a jour l'aide

La premiere fois, telechargez les fichiers d'aide complets :

```powershell
# Download and install the latest help files (requires admin)
Update-Help -Force -ErrorAction SilentlyContinue
```

Resultat :

```text
PS C:\> Update-Help -Force -ErrorAction SilentlyContinue
# (Telechargement en cours... aucune sortie visible si tout se passe bien)
# Des avertissements peuvent apparaitre pour certains modules sans aide en ligne
```

### Get-Help : la commande indispensable

```powershell
# Basic help
Get-Help Get-Service

# Detailed help with examples
Get-Help Get-Service -Detailed

# Full help (all parameters, notes, examples)
Get-Help Get-Service -Full

# Show only examples
Get-Help Get-Service -Examples

# Open help in a separate window
Get-Help Get-Service -ShowWindow

# Online help (opens in the browser)
Get-Help Get-Service -Online
```

Resultat (extrait de `Get-Help Get-Service`) :

```text
NAME
    Get-Service

SYNOPSIS
    Gets the services on a local or remote computer.

SYNTAX
    Get-Service [[-Name] <String[]>] [-ComputerName <String[]>]
    [-DependentServices] [-Exclude <String[]>] [-Include <String[]>]
    [-RequiredServices] [<CommonParameters>]

DESCRIPTION
    The Get-Service cmdlet gets objects that represent the services on a
    computer, including running and stopped services.
    ...

RELATED LINKS
    Online Version: https://learn.microsoft.com/...
```

!!! tip "Raccourci"

    `help` est un alias de `Get-Help` avec pagination automatique.
    `man` fonctionne aussi (alias Unix).

    ```powershell
    help Get-Service
    man Get-Process
    ```

### Lire la syntaxe

La section **SYNTAX** utilise des conventions :

| Notation | Signification |
|----------|--------------|
| `[-Name]` | Parametre optionnel |
| `<String>` | Type attendu |
| `[-Name] <String>` | Parametre nomme, valeur obligatoire |
| `[[-Name] <String>]` | Parametre positionnel et optionnel |
| `[-Force]` | Switch (pas de valeur, juste present ou absent) |

```powershell
# Example syntax from Get-Help Get-Service:
# Get-Service [[-Name] <String[]>] [-ComputerName <String[]>]

# This means:
Get-Service                          # No parameters needed
Get-Service "WinRM"                  # -Name is positional
Get-Service -Name "WinRM"            # Explicit parameter name
Get-Service -Name "W*" -ComputerName "SRV-01"  # Multiple parameters
```

## Decouvrir les commandes

### Get-Command : trouver des cmdlets

```powershell
# Find all cmdlets containing "service"
Get-Command *service*

# Find cmdlets for a specific module
Get-Command -Module "NetAdapter"

# Find cmdlets by verb
Get-Command -Verb "Get" -Noun "Net*"

# Find all cmdlets available for DNS
Get-Command -Module "DnsServer"

# Count available cmdlets
(Get-Command -CommandType Cmdlet).Count
```

Resultat (extrait de `Get-Command *service*`) :

```text
CommandType     Name                              Version    Source
-----------     ----                              -------    ------
Cmdlet          Get-Service                       3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          New-Service                       3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Restart-Service                   3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Resume-Service                    3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Set-Service                       3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Start-Service                     3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Stop-Service                      3.1.0.0    Microsoft.PowerShell.Management
Cmdlet          Suspend-Service                   3.1.0.0    Microsoft.PowerShell.Management

PS C:\> (Get-Command -CommandType Cmdlet).Count
1638
```

### Exploration par module

```powershell
# List all installed modules
Get-Module -ListAvailable

# List commands in a specific module
Get-Command -Module "NetTCPIP"

# Import a module (usually automatic)
Import-Module "ActiveDirectory"

# Find modules available online
Find-Module -Name "*ActiveDirectory*"
```

Resultat (extrait de `Get-Module -ListAvailable`) :

```text
    Directory: C:\Windows\system32\WindowsPowerShell\v1.0\Modules

ModuleType Version    Name                          ExportedCommands
---------- -------    ----                          ----------------
Manifest   1.0.0.0    ActiveDirectory               {Add-ADGroupMember, Get-ADUser...}
Manifest   2.0.0.0    DnsServer                     {Add-DnsServerResourceRecord...}
Manifest   2.0.0.0    NetAdapter                    {Get-NetAdapter, Set-NetAdapter...}
Manifest   1.0.0.0    NetTCPIP                      {Get-NetIPAddress, New-NetIPAddress...}
Script     1.0.0.0    ServerManager                 {Get-WindowsFeature, Install-WindowsFeature...}
...
```

### Decouvrir les proprietes d'un objet

```powershell
# What properties does a Service object have?
Get-Service | Get-Member

# What properties does a Process object have?
Get-Process | Get-Member -MemberType Property

# See all property values for a specific object
Get-Service -Name "WinRM" | Format-List *

# What type of object does a cmdlet return?
(Get-Service)[0].GetType().FullName
```

Resultat (extrait de `Get-Service | Get-Member -MemberType Property`) :

```text
   TypeName: System.ServiceProcess.ServiceController

Name                MemberType Definition
----                ---------- ----------
CanPauseAndContinue Property   bool CanPauseAndContinue {get;}
CanShutdown         Property   bool CanShutdown {get;}
CanStop             Property   bool CanStop {get;}
DependentServices   Property   ServiceController[] DependentServices {get;}
DisplayName         Property   string DisplayName {get;set;}
MachineName         Property   string MachineName {get;set;}
ServiceName         Property   string ServiceName {get;set;}
ServiceType         Property   ServiceType ServiceType {get;}
StartType           Property   ServiceStartMode StartType {get;}
Status              Property   ServiceControllerStatus Status {get;}

PS C:\> (Get-Service)[0].GetType().FullName
System.ServiceProcess.ServiceController
```

!!! example "Analogie"

    `Get-Member` est comme une radiographie : il revele la structure interne d'un objet. Tout comme un medecin utilise une radio pour voir les os sous la peau, `Get-Member` vous montre les proprietes et methodes cachees derriere chaque resultat PowerShell.

## Techniques de decouverte

### Autocompletion avec ++tab++

```powershell
# Type partially, then press Tab to complete
Get-Serv[TAB]        # -> Get-Service
Get-Service -N[TAB]  # -> Get-Service -Name
```

### Historique des commandes

```powershell
# Display command history
Get-History

# Search in history
Get-History | Where-Object CommandLine -like "*service*"

# Re-execute a command from history
Invoke-History -Id 5

# Keyboard shortcuts:
# Arrow Up/Down: navigate history
# Ctrl+R: reverse search in history
# F7: display history popup (Windows Terminal)
```

Resultat :

```text
PS C:\> Get-History

  Id CommandLine
  -- -----------
   1 Get-Service
   2 Get-Service -Name "WinRM"
   3 Get-Help Get-Service -Examples
   4 Get-Command *DNS*
   5 Get-NetIPConfiguration
```

### Wildcards

```powershell
# Use * for pattern matching
Get-Service -Name "Win*"
Get-Command -Name "*DNS*"
Get-ChildItem -Path "C:\Windows\*.log"

# Use ? for single character matching
Get-ChildItem -Path "C:\Windows\System3?"
```

Resultat :

```text
PS C:\> Get-Service -Name "Win*"

Status   Name               DisplayName
------   ----               -----------
Running  WinDefend          Microsoft Defender Antivirus Service
Running  WinHttpAutoProx... WinHTTP Web Proxy Auto-Discovery Se...
Running  Winmgmt            Windows Management Instrumentation
Running  WinRM              Windows Remote Management (WS-Manag...
Stopped  WinHttpAutoProx... WinHTTP Web Proxy Auto-Discovery Se...
```

## About_* : documentation conceptuelle

Au-dela des cmdlets, PowerShell inclut des articles conceptuels :

```powershell
# List all conceptual help topics
Get-Help about_*

# Key topics for beginners
Get-Help about_Comparison_Operators  # -eq, -ne, -gt, -lt, -like, -match
Get-Help about_Operators             # Arithmetic, assignment, logical
Get-Help about_Pipelines             # How the pipeline works
Get-Help about_Wildcards             # *, ?, [a-z]
Get-Help about_Variables             # $var, $_, $env:
Get-Help about_If                    # if/elseif/else
Get-Help about_ForEach               # ForEach-Object and foreach
Get-Help about_Execution_Policies    # Script execution policies
```

## Aide-memoire rapide

| Je veux... | Commande |
|------------|----------|
| Aide sur une cmdlet | `Get-Help <cmdlet>` |
| Exemples d'utilisation | `Get-Help <cmdlet> -Examples` |
| Trouver une cmdlet | `Get-Command *motcle*` |
| Proprietes d'un objet | `<cmdlet> \| Get-Member` |
| Toutes les valeurs d'un objet | `<cmdlet> \| Format-List *` |
| Commandes d'un module | `Get-Command -Module <module>` |
| Modules disponibles | `Get-Module -ListAvailable` |
| Documentation conceptuelle | `Get-Help about_<sujet>` |

!!! example "Scenario pratique"

    **Contexte** : Thomas, technicien reseau, doit configurer les regles de pare-feu sur `SRV-01` mais ne connait pas les cmdlets PowerShell pour le firewall.

    **Etape 1** : Chercher les cmdlets liees au pare-feu

    ```powershell
    Get-Command *firewall*
    ```

    ```text
    CommandType     Name                                    Version    Source
    -----------     ----                                    -------    ------
    Function        Copy-NetFirewallRule                    2.0.0.0    NetSecurity
    Function        Get-NetFirewallProfile                  2.0.0.0    NetSecurity
    Function        Get-NetFirewallRule                     2.0.0.0    NetSecurity
    Function        New-NetFirewallRule                     2.0.0.0    NetSecurity
    Function        Remove-NetFirewallRule                  2.0.0.0    NetSecurity
    Function        Set-NetFirewallProfile                  2.0.0.0    NetSecurity
    ...
    ```

    **Etape 2** : Consulter l'aide avec des exemples concrets

    ```powershell
    Get-Help New-NetFirewallRule -Examples
    ```

    ```text
    EXAMPLE 1
        New-NetFirewallRule -DisplayName "Allow Inbound Telnet" -Direction Inbound
        -Program %SystemRoot%\System32\tlntsvr.exe -RemoteAddress LocalSubnet
        -Action Allow
    ...
    ```

    **Etape 3** : Decouvrir les proprietes d'une regle existante

    ```powershell
    Get-NetFirewallRule -DisplayName "Windows Remote Management*" | Format-List *
    ```

    Thomas a trouve toutes les informations necessaires sans quitter la console ni ouvrir un navigateur.

!!! danger "Erreurs courantes"

    1. **Oublier de lancer `Update-Help`** : sans cette commande, `Get-Help` affiche une aide minimale generee automatiquement. Lancez `Update-Help -Force` une premiere fois pour obtenir la documentation complete.

    2. **Confondre `Get-Help` et `Get-Command`** : `Get-Command` sert a *trouver* une cmdlet par son nom. `Get-Help` sert a *comprendre* comment l'utiliser. Les deux sont complementaires.

    3. **Ne pas utiliser `-Examples`** : la section d'exemples est souvent la plus utile pour comprendre rapidement une cmdlet. `Get-Help <cmdlet> -Examples` devrait etre votre premier reflexe.

    4. **Ignorer `Get-Member`** : beaucoup de debutants ne savent pas quelles proprietes un objet possede. Prenez l'habitude de taper `| Get-Member` apres n'importe quelle cmdlet pour decouvrir ses proprietes.

## Points cles a retenir

- `Get-Help` est votre documentation en ligne de commande
- `Get-Command` vous aide a trouver les bonnes cmdlets
- `Get-Member` revele la structure des objets
- ++tab++ pour l'autocompletion, ++up++ et ++down++ pour l'historique
- Executez `Update-Help` une premiere fois pour avoir l'aide complete

## Pour aller plus loin

- [Pipeline et objets](pipeline-et-objets.md) - appliquer ces connaissances
- [PowerShell avance](../../automatisation/powershell-avance/index.md) - scripting et automatisation

