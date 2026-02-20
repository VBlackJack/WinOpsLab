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

### Decouvrir les proprietes d'un objet

```powershell
# Get-Member reveals all properties and methods
Get-Service | Get-Member

# Display only properties
Get-Service | Get-Member -MemberType Property

# Quick way: pipe to Format-List *
Get-Service -Name "WinRM" | Format-List *
```

!!! tip "Get-Member est votre meilleur ami"

    Quand vous ne savez pas quelles proprietes un objet possede,
    utilisez `| Get-Member` (alias `| gm`).

## Le pipeline ( | )

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

## Points cles a retenir

- PowerShell passe des **objets** dans le pipeline, pas du texte
- `Get-Member` revele les proprietes et methodes d'un objet
- `Where-Object` filtre, `Select-Object` choisit, `Sort-Object` trie
- `$_` represente l'objet courant dans un bloc de script
- Les cmdlets `Format-*` sont toujours en derniere position

## Pour aller plus loin

- [Aide et decouverte](aide-et-decouverte.md) - explorer les cmdlets
- [PowerShell avance](../../automatisation/powershell-avance/index.md) - scripting et fonctions
