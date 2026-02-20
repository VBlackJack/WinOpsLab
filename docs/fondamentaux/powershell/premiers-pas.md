---
title: Premiers pas avec PowerShell
description: Introduction a PowerShell - lancement, navigation et concepts de base.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Premiers pas avec PowerShell

<span class="level-beginner">Debutant</span> · Temps estime : 20 minutes

## Qu'est-ce que PowerShell ?

PowerShell est le shell et langage de script de Microsoft pour l'administration systeme. Contrairement a `cmd.exe`, PowerShell manipule des **objets .NET** plutot que du texte brut, ce qui le rend bien plus puissant.

| Caracteristique | cmd.exe | PowerShell |
|-----------------|---------|------------|
| Type de donnees | Texte | Objets .NET |
| Extensibilite | Limitee | Modules illimites |
| Scripting | Batch (.bat) | Scripts (.ps1) |
| Administration a distance | Limitee | Remoting integre |
| Autocompletion | Basique | Avancee (++tab++) |

## Lancer PowerShell

### Sur Windows Server

=== "Desktop Experience"

    - Clic droit sur le menu Demarrer > **Windows PowerShell (Admin)**
    - Ou taper `powershell` dans la barre de recherche

=== "Server Core"

    PowerShell se lance automatiquement apres la connexion.

    ```powershell
    # If you're in cmd.exe, switch to PowerShell
    powershell
    ```

### PowerShell vs Windows PowerShell

| Version | Executable | Framework | Versions |
|---------|-----------|-----------|----------|
| Windows PowerShell | `powershell.exe` | .NET Framework | 5.1 (inclus dans Windows) |
| PowerShell (Core) | `pwsh.exe` | .NET (cross-platform) | 7.x (a installer separement) |

!!! tip "Quelle version utiliser ?"

    Sur Windows Server 2022, utilisez **Windows PowerShell 5.1** (inclus par defaut).
    C'est la version officielle pour l'administration Windows Server.
    PowerShell 7 est utile pour le scripting cross-platform.

```powershell
# Check the current PowerShell version
$PSVersionTable.PSVersion
```

## Concepts fondamentaux

### La console

La console PowerShell est interactive : vous tapez une commande et obtenez un resultat immediat.

```powershell
# Display the current date
Get-Date

# Display the hostname
hostname

# Display the current user
whoami

# Display the current directory
Get-Location
```

### Les commandes s'appellent des cmdlets

Une **cmdlet** (command-let) suit toujours le format **Verbe-Nom** :

```
Get-Process       # Verbe: Get, Nom: Process
Stop-Service      # Verbe: Stop, Nom: Service
New-Item          # Verbe: New, Nom: Item
Remove-NetRoute   # Verbe: Remove, Nom: NetRoute
```

Les verbes les plus courants :

| Verbe | Action | Exemple |
|-------|--------|---------|
| `Get` | Lire / Obtenir | `Get-Service` |
| `Set` | Modifier | `Set-TimeZone` |
| `New` | Creer | `New-Item` |
| `Remove` | Supprimer | `Remove-Item` |
| `Start` | Demarrer | `Start-Service` |
| `Stop` | Arreter | `Stop-Process` |
| `Restart` | Redemarrer | `Restart-Computer` |
| `Test` | Tester | `Test-Connection` |
| `Enable` | Activer | `Enable-PSRemoting` |
| `Disable` | Desactiver | `Disable-NetAdapter` |
| `Install` | Installer | `Install-WindowsFeature` |

### Les alias

PowerShell fournit des alias (raccourcis) pour les commandes courantes :

```powershell
# These are equivalent
Get-ChildItem
dir
ls

# These are equivalent
Set-Location C:\Windows
cd C:\Windows

# These are equivalent
Clear-Host
cls

# List all aliases
Get-Alias
```

!!! warning "Eviter les alias dans les scripts"

    Utilisez toujours les noms complets des cmdlets dans vos scripts
    pour la lisibilite et la maintenabilite. Les alias sont pratiques
    uniquement en mode interactif.

## Navigation dans le systeme de fichiers

```powershell
# Display current location
Get-Location   # alias: pwd

# Change directory
Set-Location C:\Windows   # alias: cd

# List files and directories
Get-ChildItem   # alias: ls, dir

# List only files
Get-ChildItem -File

# List only directories
Get-ChildItem -Directory

# List recursively
Get-ChildItem -Recurse

# Create a directory
New-Item -ItemType Directory -Name "MonDossier"

# Create a file
New-Item -ItemType File -Name "test.txt"

# Copy a file
Copy-Item -Path "test.txt" -Destination "backup.txt"

# Move a file
Move-Item -Path "backup.txt" -Destination "C:\Temp\backup.txt"

# Delete a file
Remove-Item -Path "test.txt"
```

## Execution de scripts

Par defaut, l'execution de scripts est restreinte. Verifiez et modifiez la politique :

```powershell
# Check current execution policy
Get-ExecutionPolicy

# Set execution policy to allow local scripts
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

# Run a script
.\mon-script.ps1
```

| Politique | Description |
|-----------|-------------|
| `Restricted` | Aucun script autorise (defaut) |
| `AllSigned` | Seuls les scripts signes sont autorises |
| `RemoteSigned` | Scripts locaux OK, scripts distants doivent etre signes |
| `Unrestricted` | Tous les scripts sont autorises (deconseille) |

!!! tip "Recommandation"

    Utilisez `RemoteSigned` pour un bon compromis securite/praticite en lab.
    En production, privilegiez `AllSigned`.

## Points cles a retenir

- PowerShell manipule des objets, pas du texte
- Les cmdlets suivent le format Verbe-Nom
- Utilisez ++tab++ pour l'autocompletion
- `Get-` pour lire, `Set-` pour modifier, `New-` pour creer
- Les alias sont des raccourcis (pratiques en interactif, a eviter dans les scripts)

## Pour aller plus loin

- [Cmdlets essentielles](cmdlets-essentielles.md) - les commandes a connaitre
- [Pipeline et objets](pipeline-et-objets.md) - la puissance du pipeline
- [Aide et decouverte](aide-et-decouverte.md) - apprendre a se debrouiller seul
