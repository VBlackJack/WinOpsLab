---
title: Pipeline et objets
description: Comprendre le pipeline PowerShell et la manipulation d'objets .NET.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Pipeline et objets

<span class="level-beginner">Debutant</span> · Temps estime : 20 minutes

## PowerShell manipule des objets

!!! example "Analogie"

    Imaginez un tapis roulant dans une usine. Avec `cmd.exe`, le tapis transporte des feuilles de papier avec du texte ecrit dessus — vous devez lire et interpreter chaque feuille vous-meme. Avec PowerShell, le tapis transporte des **boites etiquetees** contenant des informations structurees (nom, taille, date...). Vous pouvez trier les boites, en ouvrir certaines, en jeter d'autres — sans jamais avoir a dechiffrer du texte brut.

La difference fondamentale entre PowerShell et les shells traditionnels (bash, cmd) :

| Shell | Type de donnees | Exemple `dir` |
|-------|----------------|---------------|
| cmd / bash | Texte brut | Lignes de texte a parser |
| PowerShell | Objets .NET | Objets `FileInfo` avec proprietes (Name, Length, CreationTime...) |

```powershell
# A cmdlet returns objects, not text
$files = Get-ChildItem C:\Windows

# Each object has properties
$files[0].Name
$files[0].Length
$files[0].CreationTime
$files[0].Extension

# And methods
$files[0].GetType()
```

Resultat :

```text
PS C:\> $files = Get-ChildItem C:\Windows
PS C:\> $files[0].Name
appcompat
PS C:\> $files[0].Length

PS C:\> $files[0].CreationTime
samedi 7 mai 2022 09:14:52
PS C:\> $files[0].Extension

PS C:\> $files[0].GetType()

IsPublic IsSerial Name                 BaseType
-------- -------- ----                 --------
True     False    DirectoryInfo        System.IO.FileSystemInfo
```

### Decouvrir les proprietes d'un objet

```powershell
# Get-Member reveals all properties and methods
Get-Service | Get-Member

# Display only properties
Get-Service | Get-Member -MemberType Property

# Quick way: pipe to Format-List *
Get-Service -Name "WinRM" | Format-List *
```

Resultat (extrait de `Get-Service -Name "WinRM" | Format-List *`) :

```text
Name                : WinRM
DisplayName         : Windows Remote Management (WS-Management)
Status              : Running
DependentServices   : {}
ServicesDependedOn  : {RPCSS, HTTP}
CanPauseAndContinue : False
CanShutdown         : True
CanStop             : True
ServiceType         : Win32ShareProcess
StartType           : Automatic
MachineName         : .
```

!!! tip "Get-Member est votre meilleur ami"

    Quand vous ne savez pas quelles proprietes un objet possede,
    utilisez `| Get-Member` (alias `| gm`).

## Le pipeline ( | )

!!! example "Analogie"

    Le pipeline fonctionne comme une chaine de montage dans une usine. La premiere station (`Get-Service`) produit les pieces. La deuxieme (`Where-Object`) effectue le controle qualite et ecarte les pieces non conformes. La troisieme (`Select-Object`) met en forme. La derniere (`Sort-Object`) range le tout dans l'ordre. Chaque station fait un travail precis et passe le resultat a la suivante.

Le **pipeline** passe les objets d'une cmdlet a la suivante. Chaque etape transforme, filtre ou formate les donnees.

```mermaid
graph LR
    A[Get-Service] -->|Objets Service| B[Where-Object]
    B -->|Objets filtres| C[Select-Object]
    C -->|Proprietes choisies| D[Sort-Object]
    D -->|Objets tries| E[Affichage]
```

### Exemple concret

```powershell
# Step by step:
# 1. Get all services
# 2. Filter: only running services
# 3. Select: only Name and DisplayName
# 4. Sort: alphabetically by DisplayName
Get-Service |
    Where-Object Status -eq "Running" |
    Select-Object Name, DisplayName |
    Sort-Object DisplayName
```

Resultat :

```text
Name                 DisplayName
----                 -----------
AppIDSvc             Application Identity
BFE                  Base Filtering Engine
CryptSvc             Cryptographic Services
DcomLaunch           DCOM Server Process Launcher
Dhcp                 DHCP Client
DNS                  DNS Server
EventLog             Windows Event Log
LanmanServer         Server
LanmanWorkstation    Workstation
Netlogon             Net Logon
WinRM                Windows Remote Management (WS-Management)
...
```

## Les cmdlets de pipeline essentielles

### Where-Object : filtrer

Filtre les objets selon une condition :

```powershell
# Simple syntax (PowerShell 3+)
Get-Process | Where-Object CPU -gt 100
Get-Service | Where-Object Status -eq "Stopped"

# Full syntax (for complex conditions)
Get-Process | Where-Object { $_.CPU -gt 100 -and $_.WorkingSet64 -gt 100MB }
Get-EventLog -LogName System -Newest 100 |
    Where-Object { $_.EntryType -eq "Error" -or $_.EntryType -eq "Warning" }
```

Resultat (extrait de `Get-Process | Where-Object CPU -gt 100`) :

```text
Handles  NPM(K)    PM(K)      WS(K)   CPU(s)     Id  SI ProcessName
-------  ------    -----      -----   ------     --  -- -----------
    842      45   185432     192048   312.45   1284   0 dns
    523      32    98764     104520   187.22   2048   0 lsass
    156      12    28940      31204   132.11   1876   0 WmiPrvSE
```

!!! note "`$_` represente l'objet courant"

    Dans un bloc `{ }`, `$_` (ou `$PSItem`) represente chaque objet
    traversant le pipeline.

### Select-Object : choisir les proprietes

```powershell
# Select specific properties
Get-Process | Select-Object Name, CPU, WorkingSet64

# Select first/last N items
Get-Process | Select-Object -First 5
Get-EventLog -LogName System | Select-Object -Last 10

# Create calculated properties
Get-Process | Select-Object Name, @{
    Name       = "MemoryMB"
    Expression = { [math]::Round($_.WorkingSet64 / 1MB, 2) }
}

# Select unique values
Get-EventLog -LogName System -Newest 100 |
    Select-Object -ExpandProperty Source -Unique
```

Resultat (extrait de `Select-Object` avec propriete calculee) :

```text
PS C:\> Get-Process | Select-Object Name, @{Name="MemoryMB"; Expression={[math]::Round($_.WorkingSet64/1MB,2)}} | Select-Object -First 5

Name              MemoryMB
----              --------
csrss                18.45
dns                 192.05
dwm                  42.33
lsass               104.52
services             12.87
```

### Sort-Object : trier

```powershell
# Sort ascending (default)
Get-Process | Sort-Object CPU

# Sort descending
Get-Process | Sort-Object CPU -Descending

# Sort by multiple properties
Get-Service | Sort-Object Status, DisplayName

# Sort and take top 10
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10
```

### Group-Object : regrouper

```powershell
# Group services by status
Get-Service | Group-Object Status

# Group event log entries by source
Get-EventLog -LogName System -Newest 200 | Group-Object Source | Sort-Object Count -Descending

# Group files by extension
Get-ChildItem C:\Windows -File | Group-Object Extension | Sort-Object Count -Descending
```

Resultat :

```text
PS C:\> Get-Service | Group-Object Status

Count Name                      Group
----- ----                      -----
   82 Running                   {AppIDSvc, BFE, CryptSvc, DcomLaunch...}
  143 Stopped                   {AJRouter, ALG, AppMgmt, AppReadiness...}
```

### Measure-Object : compter et calculer

```powershell
# Count items
Get-Service | Measure-Object

# Count only running services
Get-Service | Where-Object Status -eq "Running" | Measure-Object

# Calculate statistics on file sizes
Get-ChildItem C:\Windows -File | Measure-Object -Property Length -Sum -Average -Maximum

# Display the result
$stats = Get-ChildItem C:\Windows -File | Measure-Object -Property Length -Sum -Average
"Total: $([math]::Round($stats.Sum / 1MB, 2)) MB | Average: $([math]::Round($stats.Average / 1KB, 2)) KB"
```

Resultat :

```text
PS C:\> Get-Service | Where-Object Status -eq "Running" | Measure-Object

Count    : 82
Average  :
Sum      :
Maximum  :
Minimum  :
Property :

PS C:\> Get-ChildItem C:\Windows -File | Measure-Object -Property Length -Sum -Average -Maximum

Count    : 34
Average  : 425876.23
Sum      : 14479799
Maximum  : 3170304
Minimum  :
Property : Length

Total: 13.81 MB | Average: 415.89 KB
```

### ForEach-Object : executer une action sur chaque objet

```powershell
# Simple action on each item
Get-Service | ForEach-Object { Write-Host "$($_.Name) is $($_.Status)" }

# Restart all stopped critical services
Get-Service | Where-Object {
    $_.StartType -eq "Automatic" -and $_.Status -eq "Stopped"
} | ForEach-Object {
    Write-Host "Starting $($_.DisplayName)..."
    Start-Service -Name $_.Name -ErrorAction SilentlyContinue
}
```

Resultat :

```text
PS C:\> Get-Service | ForEach-Object { Write-Host "$($_.Name) is $($_.Status)" } | Select-Object -First 5
AppIDSvc is Running
BFE is Running
BITS is Stopped
CertPropSvc is Running
CryptSvc is Running
```

## Formatage de la sortie

Par defaut, PowerShell choisit le format d'affichage. Vous pouvez forcer un format :

```powershell
# Table format (default for most cmdlets)
Get-Service | Format-Table -AutoSize

# List format (detailed, one property per line)
Get-Service -Name "WinRM" | Format-List *

# Wide format (single property, multiple columns)
Get-Service | Format-Wide -Property DisplayName -Column 3
```

!!! danger "Format-* doit etre la derniere commande du pipeline"

    Les cmdlets `Format-*` transforment les objets en objets de formatage.
    Apres un `Format-Table`, vous ne pouvez plus filtrer ou exporter.

    ```powershell
    # WRONG: Format-Table before Export-Csv
    Get-Service | Format-Table | Export-Csv "services.csv"

    # CORRECT: Export-Csv before or instead of Format-Table
    Get-Service | Export-Csv "services.csv" -NoTypeInformation
    ```

## Export des donnees

```powershell
# Export to CSV
Get-Process | Select-Object Name, CPU | Export-Csv "processes.csv" -NoTypeInformation

# Export to JSON
Get-Service | Select-Object Name, Status | ConvertTo-Json | Out-File "services.json"

# Export to HTML
Get-Service | ConvertTo-Html -Title "Services" | Out-File "services.html"

# Send to clipboard
Get-Process | Select-Object Name | Set-Clipboard
```

## Enchainement de pipelines : exemples pratiques

```powershell
# Find the 5 largest files in C:\Windows
Get-ChildItem C:\Windows -Recurse -File -ErrorAction SilentlyContinue |
    Sort-Object Length -Descending |
    Select-Object -First 5 Name, @{N="SizeMB"; E={[math]::Round($_.Length/1MB,2)}}, DirectoryName

# List services set to Automatic but not running
Get-Service |
    Where-Object { $_.StartType -eq "Automatic" -and $_.Status -ne "Running" } |
    Select-Object Name, DisplayName, Status |
    Sort-Object DisplayName

# Count event errors per source in the last 24h
Get-WinEvent -FilterHashtable @{
    LogName   = "System"
    Level     = 2  # Error
    StartTime = (Get-Date).AddHours(-24)
} -ErrorAction SilentlyContinue |
    Group-Object ProviderName |
    Sort-Object Count -Descending |
    Select-Object Count, Name
```

Resultat (extrait du premier exemple : 5 plus gros fichiers) :

```text
Name                SizeMB  DirectoryName
----                ------  -------------
MRT.exe             135.42  C:\Windows\System32
ntoskrnl.exe         11.24  C:\Windows\System32
explorer.exe          5.12  C:\Windows
shell32.dll           4.98  C:\Windows\System32
setupapi.dev.log      3.87  C:\Windows\inf
```

!!! example "Scenario pratique"

    **Contexte** : Claire, administratrice systeme, doit preparer un rapport hebdomadaire sur l'etat des services de `SRV-DC01` pour son responsable. Elle veut un fichier CSV listant les services automatiques qui ne tournent pas.

    **Etape 1** : Construire le pipeline etape par etape

    ```powershell
    # D'abord, recuperer tous les services
    Get-Service | Select-Object -First 3
    ```

    ```text
    Status   Name               DisplayName
    ------   ----               -----------
    Stopped  AJRouter           AllJoyn Router Service
    Stopped  ALG                Application Layer Gateway Service
    Running  AppIDSvc           Application Identity
    ```

    **Etape 2** : Filtrer les services automatiques arretes

    ```powershell
    Get-Service |
        Where-Object { $_.StartType -eq "Automatic" -and $_.Status -ne "Running" } |
        Select-Object Name, DisplayName, Status, StartType
    ```

    ```text
    Name              DisplayName                           Status  StartType
    ----              -----------                           ------  ---------
    BITS              Background Intelligent Transfer Serv. Stopped Automatic
    TrustedInstaller  Windows Modules Installer             Stopped Automatic
    wuauserv          Windows Update                        Stopped Automatic
    ```

    **Etape 3** : Exporter en CSV pour le rapport

    ```powershell
    Get-Service |
        Where-Object { $_.StartType -eq "Automatic" -and $_.Status -ne "Running" } |
        Select-Object Name, DisplayName, Status, StartType |
        Export-Csv -Path "C:\Temp\services-arretes.csv" -NoTypeInformation -Encoding UTF8
    ```

    Claire planifie ce script en tache planifiee chaque lundi a 8h. Son responsable recoit ainsi un rapport automatique sans aucune intervention manuelle.

!!! danger "Erreurs courantes"

    1. **Placer `Format-Table` avant `Export-Csv`** : les cmdlets `Format-*` transforment les objets en objets de formatage. Apres un `Format-Table`, les donnees ne sont plus exploitables. Placez toujours `Format-*` en derniere position ou utilisez `Select-Object` avant l'export.

    2. **Oublier `$_` dans les blocs `Where-Object` complexes** : ecrire `Where-Object { Status -eq "Running" }` au lieu de `Where-Object { $_.Status -eq "Running" }` provoque une erreur silencieuse. La syntaxe simplifiee (`Where-Object Status -eq "Running"`) ne fonctionne que pour les conditions simples.

    3. **Enchainer trop d'etapes sans tester** : construisez votre pipeline progressivement. Testez chaque etape avant d'en ajouter une nouvelle. Un pipeline de 6 etapes qui ne renvoie rien est difficile a debugger.

    4. **Confondre `Select-Object` et `Where-Object`** : `Where-Object` filtre les *lignes* (quels objets garder), `Select-Object` filtre les *colonnes* (quelles proprietes afficher). Ce sont deux operations complementaires mais distinctes.

## Points cles a retenir

- PowerShell passe des **objets** dans le pipeline, pas du texte
- `Get-Member` revele les proprietes et methodes d'un objet
- `Where-Object` filtre, `Select-Object` choisit, `Sort-Object` trie
- `$_` represente l'objet courant dans un bloc de script
- Les cmdlets `Format-*` sont toujours en derniere position

## Pour aller plus loin

- [Aide et decouverte](aide-et-decouverte.md) - explorer les cmdlets
- [PowerShell avance](../../automatisation/powershell-avance/index.md) - scripting et fonctions
