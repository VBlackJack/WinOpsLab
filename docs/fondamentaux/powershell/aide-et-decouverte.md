---
title: Aide et decouverte
description: Utiliser le systeme d'aide PowerShell pour explorer et apprendre par soi-meme.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Aide et decouverte

!!! info "Niveau : Debutant"

    Temps estime : 15 minutes

## Le systeme d'aide integre

PowerShell dispose d'un systeme d'aide complet qui vous permet d'apprendre n'importe quelle cmdlet sans quitter la console.

### Mettre a jour l'aide

La premiere fois, telechargez les fichiers d'aide complets :

```powershell
# Download and install the latest help files (requires admin)
Update-Help -Force -ErrorAction SilentlyContinue
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

### Wildcards

```powershell
# Use * for pattern matching
Get-Service -Name "Win*"
Get-Command -Name "*DNS*"
Get-ChildItem -Path "C:\Windows\*.log"

# Use ? for single character matching
Get-ChildItem -Path "C:\Windows\System3?"
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

## Points cles a retenir

- `Get-Help` est votre documentation en ligne de commande
- `Get-Command` vous aide a trouver les bonnes cmdlets
- `Get-Member` revele la structure des objets
- ++tab++ pour l'autocompletion, ++up++ et ++down++ pour l'historique
- Executez `Update-Help` une premiere fois pour avoir l'aide complete

## Pour aller plus loin

- [Pipeline et objets](pipeline-et-objets.md) - appliquer ces connaissances
- [PowerShell avance](../../automatisation/powershell-avance/index.md) - scripting et automatisation
