---
title: Cmdlets essentielles
description: Les cmdlets PowerShell incontournables pour l'administration Windows Server.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Cmdlets essentielles

!!! info "Niveau : Debutant"

    Temps estime : 25 minutes

## Informations systeme

```powershell
# OS information
Get-ComputerInfo | Select-Object WindowsProductName, OsVersion, OsBuildNumber

# Hostname
$env:COMPUTERNAME

# System uptime
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime

# Installed hotfixes
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10

# System environment variables
Get-ChildItem Env:
```

## Gestion des services

```powershell
# List all services
Get-Service

# List running services
Get-Service | Where-Object Status -eq "Running"

# Check a specific service
Get-Service -Name "WinRM"

# Start a service
Start-Service -Name "WinRM"

# Stop a service
Stop-Service -Name "Spooler"

# Restart a service
Restart-Service -Name "WinRM"

# Set a service to start automatically
Set-Service -Name "WinRM" -StartupType Automatic

# Disable a service
Set-Service -Name "Spooler" -StartupType Disabled
```

## Gestion des processus

```powershell
# List all processes
Get-Process

# List processes sorted by CPU usage
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10

# List processes sorted by memory (Working Set)
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 Name, @{
    Name = "MemoryMB"
    Expression = { [math]::Round($_.WorkingSet64 / 1MB, 2) }
}

# Find a specific process
Get-Process -Name "explorer"

# Stop a process
Stop-Process -Name "notepad" -Force

# Stop a process by PID
Stop-Process -Id 1234 -Force
```

## Gestion du reseau

```powershell
# Display IP configuration
Get-NetIPConfiguration

# List all network adapters
Get-NetAdapter

# Display IP addresses
Get-NetIPAddress -AddressFamily IPv4

# Test connectivity (ping)
Test-Connection -ComputerName 8.8.8.8 -Count 4

# Test a specific port
Test-NetConnection -ComputerName "SRV-DC-01" -Port 3389

# DNS lookup
Resolve-DnsName -Name "google.com"

# Display the routing table
Get-NetRoute -AddressFamily IPv4

# Display active TCP connections
Get-NetTCPConnection -State Established
```

## Gestion des fichiers et dossiers

```powershell
# List files
Get-ChildItem -Path "C:\Windows\Logs" -Recurse -File

# Search for files by name
Get-ChildItem -Path "C:\" -Filter "*.log" -Recurse -ErrorAction SilentlyContinue

# Get file size
(Get-Item "C:\Windows\System32\ntoskrnl.exe").Length / 1MB

# Read file contents
Get-Content -Path "C:\Windows\System32\drivers\etc\hosts"

# Write to a file
Set-Content -Path "C:\Temp\test.txt" -Value "Hello World"

# Append to a file
Add-Content -Path "C:\Temp\test.txt" -Value "New line"

# Search text in files
Select-String -Path "C:\Temp\*.log" -Pattern "error"
```

## Gestion des roles et fonctionnalites

```powershell
# List all available roles/features
Get-WindowsFeature

# List only installed roles/features
Get-WindowsFeature | Where-Object Installed

# Search for a specific feature
Get-WindowsFeature -Name "*DNS*"

# Install a role with management tools
Install-WindowsFeature -Name "DNS" -IncludeManagementTools

# Install multiple roles
Install-WindowsFeature -Name "AD-Domain-Services", "DNS" -IncludeManagementTools

# Remove a role
Uninstall-WindowsFeature -Name "DHCP"
```

## Gestion des utilisateurs locaux

```powershell
# List local users
Get-LocalUser

# Create a local user
New-LocalUser -Name "labuser" -Password (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) -FullName "Lab User"

# Add user to a group
Add-LocalGroupMember -Group "Administrators" -Member "labuser"

# Disable a user account
Disable-LocalUser -Name "labuser"

# Remove a user
Remove-LocalUser -Name "labuser"

# List local groups
Get-LocalGroup

# List members of a group
Get-LocalGroupMember -Group "Administrators"
```

## Journaux d'evenements

```powershell
# List available event logs
Get-EventLog -List

# Display the last 20 system events
Get-EventLog -LogName System -Newest 20

# Display errors only
Get-EventLog -LogName System -EntryType Error -Newest 10

# Search in event logs (newer API)
Get-WinEvent -LogName "System" -MaxEvents 20

# Filter events by ID
Get-WinEvent -FilterHashtable @{
    LogName   = "System"
    Id        = 6005, 6006  # System startup/shutdown
    StartTime = (Get-Date).AddDays(-7)
}
```

## Gestion du pare-feu

```powershell
# Display firewall profile status
Get-NetFirewallProfile | Select-Object Name, Enabled

# List active inbound rules
Get-NetFirewallRule -Direction Inbound -Enabled True |
    Select-Object DisplayName, Action |
    Sort-Object DisplayName

# Create a new firewall rule
New-NetFirewallRule `
    -DisplayName "Allow ICMP" `
    -Direction Inbound `
    -Protocol ICMPv4 `
    -Action Allow

# Disable a firewall rule
Disable-NetFirewallRule -DisplayName "Allow ICMP"

# Remove a firewall rule
Remove-NetFirewallRule -DisplayName "Allow ICMP"
```

## Commandes utilitaires

```powershell
# Clear the screen
Clear-Host   # alias: cls

# Measure command execution time
Measure-Command { Get-Process }

# Display output in a grid (GUI)
Get-Process | Out-GridView

# Export to CSV
Get-Service | Export-Csv -Path "C:\Temp\services.csv" -NoTypeInformation

# Export to HTML
Get-Service | ConvertTo-Html | Out-File "C:\Temp\services.html"

# Restart the computer
Restart-Computer -Force

# Shutdown the computer
Stop-Computer -Force
```

## Tableau recapitulatif

| Domaine | Cmdlets cles |
|---------|-------------|
| Systeme | `Get-ComputerInfo`, `Get-HotFix`, `Restart-Computer` |
| Services | `Get-Service`, `Start-Service`, `Set-Service` |
| Processus | `Get-Process`, `Stop-Process` |
| Reseau | `Get-NetIPConfiguration`, `Test-Connection`, `Test-NetConnection` |
| Fichiers | `Get-ChildItem`, `Get-Content`, `Set-Content` |
| Roles | `Get-WindowsFeature`, `Install-WindowsFeature` |
| Utilisateurs | `Get-LocalUser`, `New-LocalUser` |
| Evenements | `Get-EventLog`, `Get-WinEvent` |
| Pare-feu | `Get-NetFirewallRule`, `New-NetFirewallRule` |

## Points cles a retenir

- Chaque domaine d'administration a ses cmdlets dediees
- Le pattern est toujours `Verbe-Nom` avec des parametres nommes
- `-ErrorAction SilentlyContinue` supprime les erreurs non critiques
- `Select-Object` permet de choisir les proprietes a afficher
- `Export-Csv` et `ConvertTo-Html` pour les rapports

## Pour aller plus loin

- [Pipeline et objets](pipeline-et-objets.md) - combiner les cmdlets
- [Aide et decouverte](aide-et-decouverte.md) - explorer les cmdlets par soi-meme
