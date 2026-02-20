---
title: "Modules PowerShell"
description: "Creer, importer et publier des modules PowerShell : modules de script (.psm1), manifestes (.psd1) et PowerShell Gallery."
tags:
  - automatisation
  - powershell
  - modules
  - windows-server
---

# Modules PowerShell

<span class="level-advanced">Avance</span> · Temps estime : 45 minutes

## Introduction

Un **module PowerShell** est un package reutilisable qui regroupe des fonctions, des cmdlets, des variables et des ressources. Les modules permettent d'organiser le code, de le partager et de le versionner. C'est le mecanisme standard pour distribuer des outils PowerShell.

!!! example "Analogie"

    Un module PowerShell est comme une boite a outils specialisee : un electricien a sa boite avec des pinces, des tournevis isoles et un testeur de tension (module `NetworkingDsc`), tandis qu'un plombier a la sienne avec des cles et des joints (module `StorageDsc`). Le manifeste `.psd1` est l'etiquette sur la boite qui liste le contenu et la version. La PowerShell Gallery est le magasin ou l'on achete ces boites pretes a l'emploi.

## Types de modules

| Type | Extension | Description |
|---|---|---|
| **Module de script** | `.psm1` | Fichier de script contenant des fonctions |
| **Module binaire** | `.dll` | Assembly .NET compilee |
| **Manifeste** | `.psd1` | Metadonnees du module (version, dependances, etc.) |
| **Module DSC** | `.psm1` + schema | Ressources DSC personnalisees |

## Emplacements des modules

```powershell
# List module search paths
$env:PSModulePath -split ";"
```

Resultat :

```text
C:\Users\admin\Documents\WindowsPowerShell\Modules
C:\Program Files\WindowsPowerShell\Modules
C:\Windows\System32\WindowsPowerShell\v1.0\Modules
```

| Emplacement | Portee | Chemin typique |
|---|---|---|
| Utilisateur courant | Utilisateur | `$HOME\Documents\PowerShell\Modules` |
| Tous les utilisateurs | Machine | `$env:ProgramFiles\PowerShell\Modules` |
| Systeme | Built-in | `$PSHOME\Modules` |

## Architecture d'un module

```mermaid
graph TD
    MANIFEST["Manifeste (.psd1)\nVersion, auteur,\ndependances"] --> ROOT_MOD["Module racine (.psm1)\nPoint d'entree"]
    ROOT_MOD --> PUB["Public/\nFonctions exportees"]
    ROOT_MOD --> PRIV["Private/\nFonctions internes"]
    PUB --> F1["Get-Something.ps1"]
    PUB --> F2["Set-Something.ps1"]
    PRIV --> F3["Invoke-Helper.ps1"]
    MANIFEST --> TESTS["Tests/\nTests Pester"]
    MANIFEST --> HELP["en-US/\nAide localisee"]

    style MANIFEST fill:#1565c0,color:#fff
    style ROOT_MOD fill:#2e7d32,color:#fff
    style PUB fill:#00838f,color:#fff
    style PRIV fill:#795548,color:#fff
    style TESTS fill:#e65100,color:#fff
    style HELP fill:#6a1b9a,color:#fff
```

## Creer un module de script

### Structure minimale

```
MonModule/
    MonModule.psd1    # Manifeste (metadonnees)
    MonModule.psm1    # Code du module (fonctions)
```

### Structure recommandee

```
MonModule/
    MonModule.psd1         # Manifeste
    MonModule.psm1         # Point d'entree (dot-source les fichiers)
    Public/                # Fonctions exportees
        Get-Something.ps1
        Set-Something.ps1
    Private/               # Fonctions internes (non exportees)
        Invoke-Helper.ps1
    en-US/                 # Aide localisee
        about_MonModule.help.txt
    Tests/                 # Tests Pester
        MonModule.Tests.ps1
```

### Fichier .psm1 (module de script)

```powershell
# MonModule.psm1
# Dot-source all public and private function files

$publicFunctions = @(Get-ChildItem -Path "$PSScriptRoot\Public\*.ps1" -ErrorAction SilentlyContinue)
$privateFunctions = @(Get-ChildItem -Path "$PSScriptRoot\Private\*.ps1" -ErrorAction SilentlyContinue)

foreach ($function in @($publicFunctions + $privateFunctions)) {
    try {
        . $function.FullName
    }
    catch {
        Write-Error "Failed to import function $($function.FullName): $_"
    }
}

# Export only public functions
Export-ModuleMember -Function $publicFunctions.BaseName
```

### Exemple de fonction publique

```powershell
# Public/Get-ServerDiskSpace.ps1
function Get-ServerDiskSpace {
    [CmdletBinding()]
    [OutputType([PSCustomObject])]
    param(
        [Parameter(Mandatory, ValueFromPipeline)]
        [string[]]$ComputerName
    )

    process {
        foreach ($computer in $ComputerName) {
            Write-Verbose "Querying disk space on $computer"
            $disks = Get-CimInstance -ClassName Win32_LogicalDisk `
                -ComputerName $computer -Filter "DriveType = 3"

            foreach ($disk in $disks) {
                [PSCustomObject]@{
                    ComputerName = $computer
                    Drive        = $disk.DeviceID
                    SizeGB       = [math]::Round($disk.Size / 1GB, 2)
                    FreeGB       = [math]::Round($disk.FreeSpace / 1GB, 2)
                    PercentFree  = [math]::Round(($disk.FreeSpace / $disk.Size) * 100, 1)
                }
            }
        }
    }
}
```

## Manifeste de module (.psd1)

Le manifeste contient les metadonnees du module : version, auteur, dependances, fonctions exportees, etc.

### Generer un manifeste

```powershell
# Generate a new module manifest
New-ModuleManifest -Path ".\MonModule\MonModule.psd1" `
    -RootModule "MonModule.psm1" `
    -ModuleVersion "1.0.0" `
    -Author "Julien Bombled" `
    -Description "Server administration toolkit" `
    -PowerShellVersion "5.1" `
    -FunctionsToExport @("Get-ServerDiskSpace", "Get-ServerHealth") `
    -CmdletsToExport @() `
    -VariablesToExport @() `
    -AliasesToExport @() `
    -Tags @("server", "admin", "monitoring")
```

### Contenu du manifeste

```powershell
# MonModule.psd1
@{
    # Script module associated with this manifest
    RootModule        = 'MonModule.psm1'

    # Version number
    ModuleVersion     = '1.0.0'

    # Unique ID
    GUID              = 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx'

    # Author
    Author            = 'Julien Bombled'

    # Description
    Description       = 'Server administration toolkit'

    # Minimum PowerShell version
    PowerShellVersion = '5.1'

    # Required modules (dependencies)
    RequiredModules   = @()

    # Functions to export (use explicit list for performance)
    FunctionsToExport = @(
        'Get-ServerDiskSpace'
        'Get-ServerHealth'
    )

    # Do not export cmdlets, variables, or aliases
    CmdletsToExport   = @()
    VariablesToExport  = @()
    AliasesToExport    = @()

    # Private data for PSGallery
    PrivateData       = @{
        PSData = @{
            Tags       = @('server', 'admin', 'monitoring')
            ProjectUri = 'https://github.com/yourname/MonModule'
            LicenseUri = 'https://github.com/yourname/MonModule/blob/main/LICENSE'
        }
    }
}
```

!!! warning "FunctionsToExport"

    Utilisez **toujours** une liste explicite dans `FunctionsToExport`. L'utilisation du wildcard `'*'` ralentit l'import du module car PowerShell doit analyser tout le code pour decouvrir les fonctions.

## Importer et utiliser un module

```powershell
# Import a module (auto-discovery from PSModulePath)
Import-Module MonModule

# Import from a specific path
Import-Module "C:\Modules\MonModule\MonModule.psd1"

# Import with verbose output (see what gets loaded)
Import-Module MonModule -Verbose

# List functions exported by a module
Get-Command -Module MonModule

# Get module info
Get-Module MonModule | Format-List *

# Remove a module from the session
Remove-Module MonModule

# Force reimport (useful during development)
Import-Module MonModule -Force
```

Resultat (Import-Module -Verbose) :

```text
VERBOSE: Loading module from path 'C:\Program Files\WindowsPowerShell\Modules\MonModule\MonModule.psd1'.
VERBOSE: Loading module from path 'C:\Program Files\WindowsPowerShell\Modules\MonModule\MonModule.psm1'.
VERBOSE: Importing function 'Get-ServerDiskSpace'.
VERBOSE: Importing function 'Get-ServerHealth'.
```

Resultat (Get-Command -Module MonModule) :

```text
CommandType     Name                    Version    Source
-----------     ----                    -------    ------
Function        Get-ServerDiskSpace     1.0.0      MonModule
Function        Get-ServerHealth        1.0.0      MonModule
```

## Installer un module depuis la PowerShell Gallery

La **PowerShell Gallery** (PSGallery) est le depot public de modules PowerShell.

```powershell
# Search for modules
Find-Module -Name "*ActiveDirectory*" | Select-Object Name, Version, Author

# Install a module for the current user
Install-Module -Name PSWindowsUpdate -Scope CurrentUser

# Install a module for all users (requires admin)
Install-Module -Name PSWindowsUpdate -Scope AllUsers

# Update an installed module
Update-Module -Name PSWindowsUpdate

# List installed modules
Get-InstalledModule
```

Resultat (Find-Module) :

```text
Name                           Version Author
----                           ------- ------
ActiveDirectoryDsc             6.4.0   DSC Community
PSActiveDirectoryCmdlets       1.0.2   Joe Smith
ActiveDirectoryManagementTools 1.3.0   Julien Bombled
```

Resultat (Get-InstalledModule) :

```text
Version Name               Repository Description
------- ----               ---------- -----------
2.4.0   PSWindowsUpdate    PSGallery  This module contain cmdlets to manage Windows Update Client.
6.4.0   ActiveDirectoryDsc PSGallery  DSC resources for configuring and managing Active Directory.
```

### Configurer un depot prive

```powershell
# Register a private NuGet repository
Register-PSRepository -Name "InternalRepo" `
    -SourceLocation "https://nuget.yourcompany.local/v2" `
    -PublishLocation "https://nuget.yourcompany.local/v2/package" `
    -InstallationPolicy Trusted

# Install from private repo
Install-Module -Name MonModule -Repository "InternalRepo"
```

## Publier un module

### Vers la PowerShell Gallery

```powershell
# Test the module manifest
Test-ModuleManifest -Path ".\MonModule\MonModule.psd1"

# Publish to PSGallery (requires API key)
Publish-Module -Path ".\MonModule" -NuGetApiKey "YOUR-API-KEY" -Repository PSGallery
```

Resultat (Test-ModuleManifest) :

```text
ModuleType Version    Name          ExportedCommands
---------- -------    ----          ----------------
Script     1.0.0      MonModule     {Get-ServerDiskSpace, Get-ServerHealth}
```

### Vers un depot prive

```powershell
# Publish to internal repository
Publish-Module -Path ".\MonModule" -Repository "InternalRepo" -NuGetApiKey "YOUR-KEY"
```

## Versionnement

Suivez le **Semantic Versioning** (SemVer) :

| Version | Signification | Exemple |
|---|---|---|
| `MAJOR.0.0` | Changements incompatibles | Suppression d'une fonction |
| `1.MINOR.0` | Nouvelles fonctionnalites compatibles | Ajout d'un parametre |
| `1.0.PATCH` | Corrections de bugs | Fix d'un bug |

```powershell
# Update module version in the manifest
Update-ModuleManifest -Path ".\MonModule\MonModule.psd1" -ModuleVersion "1.1.0"
```

Resultat :

```text
PS C:\Modules> Test-ModuleManifest -Path ".\MonModule\MonModule.psd1"

ModuleType Version    Name          ExportedCommands
---------- -------    ----          ----------------
Script     1.1.0      MonModule     {Get-ServerDiskSpace, Get-ServerHealth}
```

## Points cles a retenir

- Un module regroupe des fonctions reutilisables dans un package versionne
- La structure recommandee separe les fonctions **Public** (exportees) et **Private** (internes)
- Le manifeste (`.psd1`) est obligatoire pour la publication et ameliore les performances d'import
- Listez explicitement les fonctions dans `FunctionsToExport` (jamais `'*'`)
- La PowerShell Gallery est le depot public ; configurez un depot NuGet prive pour l'entreprise
- Suivez le Semantic Versioning pour le numerotage des versions

!!! example "Scenario pratique"

    **Claire**, administratrice systeme, a cree plusieurs fonctions utiles dans un fichier `utils.ps1` qu'elle copie manuellement sur chaque serveur. Un collegue modifie sa copie locale, ce qui provoque des differences de comportement entre les serveurs.

    **Diagnostic :**

    1. Verifier les versions du fichier sur les serveurs :
    ```powershell
    $servers = @("SRV-01", "SRV-02", "DC-01")
    Invoke-Command -ComputerName $servers -ScriptBlock {
        (Get-Item "C:\Scripts\utils.ps1").LastWriteTime
    } | Format-Table PSComputerName, LastWriteTime
    ```
    Resultat :
    ```text
    PSComputerName LastWriteTime
    -------------- -------------
    SRV-01         12/15/2025 2:30:00 PM
    SRV-02         1/8/2026 9:15:00 AM
    DC-01          11/22/2025 4:45:00 PM
    ```
    Les dates sont toutes differentes -- les fichiers ont diverge.

    2. Convertir le fichier en module avec un manifeste versionne :
    ```powershell
    New-ModuleManifest -Path "C:\Modules\LabTools\LabTools.psd1" `
        -RootModule "LabTools.psm1" `
        -ModuleVersion "1.0.0" `
        -Author "Claire" `
        -FunctionsToExport @("Get-ServerDiskSpace", "Test-ServerConnection")
    ```
    Resultat :
    ```text
    PS C:\Modules> Test-ModuleManifest .\LabTools\LabTools.psd1
    ModuleType Version Name      ExportedCommands
    ---------- ------- ----      ----------------
    Script     1.0.0   LabTools  {Get-ServerDiskSpace, Test-ServerConnection}
    ```

    3. Publier sur le depot NuGet interne pour une distribution centralisee :
    ```powershell
    Publish-Module -Path "C:\Modules\LabTools" -Repository "InternalRepo"
    ```

    **Resolution :** En convertissant le fichier en module versionne et en le publiant sur un depot interne, Claire garantit que tous les serveurs utilisent la meme version. Les mises a jour se font avec `Update-Module`.

!!! danger "Erreurs courantes"

    - **Utiliser `'*'` dans `FunctionsToExport`** : le wildcard force PowerShell a charger et analyser tout le module pour decouvrir les fonctions, ce qui ralentit significativement l'import. Listez toujours les fonctions explicitement.
    - **Oublier le manifeste `.psd1`** : un module sans manifeste fonctionne mais ne peut pas etre publie, versionne ni avoir de dependances. Creez toujours un manifeste avec `New-ModuleManifest`.
    - **Placer le module hors de `$env:PSModulePath`** : PowerShell ne decouvre automatiquement que les modules dans les chemins de `$env:PSModulePath`. Un module place ailleurs necessite un chemin complet pour l'import.
    - **Ne pas exporter les fonctions dans le `.psm1`** : sans `Export-ModuleMember`, toutes les fonctions sont exportees par defaut. Cela expose les fonctions privees qui ne sont pas destinees a etre utilisees directement.
    - **Confondre `Remove-Module` et desinstallation** : `Remove-Module` retire le module de la session active mais ne le desinstalle pas. Utilisez `Uninstall-Module` pour supprimer physiquement un module installe via `Install-Module`.

## Pour aller plus loin

- Fonctions avancees : [Fonctions avancees](fonctions-avancees.md)
- Gestion des erreurs : [Gestion des erreurs](gestion-erreurs.md)
- Documentation Microsoft : About Modules
