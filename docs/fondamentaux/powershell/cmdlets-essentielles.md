---
title: Cmdlets essentielles
description: Les cmdlets PowerShell incontournables pour l'administration Windows Server.
tags:
  - fondamentaux
  - powershell
  - debutant
---
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

# Cmdlets essentielles

<span class="level-beginner">Debutant</span> Â· Temps estime : 25 minutes

## Informations systeme

!!! example "Analogie"

    Les cmdlets PowerShell sont comme la boite a outils d'un technicien de maintenance. Chaque outil a un usage precis : le multimetre pour mesurer (`Get-`), le tournevis pour ajuster (`Set-`), la cle a molette pour assembler (`New-`). Un bon technicien connait ses outils de base par coeur â€” c'est exactement ce que cette page vous apprend.

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

Resultat :

```text
PS C:\> Get-ComputerInfo | Select-Object WindowsProductName, OsVersion, OsBuildNumber

WindowsProductName              OsVersion  OsBuildNumber
------------------              ---------  -------------
Windows Server 2022 Standard    10.0.20348 20348

PS C:\> $env:COMPUTERNAME
SRV-DC01

PS C:\> (Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime

Days              : 3
Hours             : 7
Minutes           : 42
Seconds           : 15
TotalHours        : 79.70
TotalMinutes      : 4782.25
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

Resultat (extrait) :

```text
PS C:\> Get-Service -Name "WinRM"

Status   Name               DisplayName
------   ----               -----------
Running  WinRM              Windows Remote Management (WS-Manag...

PS C:\> Get-Service | Where-Object Status -eq "Running"

Status   Name               DisplayName
------   ----               -----------
Running  CryptSvc           Cryptographic Services
Running  DcomLaunch         DCOM Server Process Launcher
Running  Dhcp               DHCP Client
Running  DNS                DNS Server
Running  EventLog           Windows Event Log
Running  LanmanServer       Server
Running  WinRM              Windows Remote Management (WS-Manag...
...
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

Resultat (extrait) :

```text
PS C:\> Get-Process | Sort-Object CPU -Descending | Select-Object -First 10

Handles  NPM(K)    PM(K)      WS(K)   CPU(s)     Id  SI ProcessName
-------  ------    -----      -----   ------     --  -- -----------
    842      45   185432     192048   312.45   1284   0 dns
    523      32    98764     104520   187.22   2048   0 lsass
    312      28    67432      72180    95.67    892   0 svchost
    198      15    42156      45680    54.33   3456   1 ServerManager
    156      12    28940      31204    32.11   1876   0 WmiPrvSE
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

Resultat (extrait) :

```text
PS C:\> Get-NetIPConfiguration

InterfaceAlias       : Ethernet0
InterfaceIndex       : 4
InterfaceDescription : Intel(R) 82574L Gigabit Network Connection
NetProfile.Name      : lab.local
IPv4Address          : 10.0.0.10
IPv4DefaultGateway   : 10.0.0.1
DNSServer            : 10.0.0.10
                       10.0.0.11

PS C:\> Test-Connection -ComputerName 10.0.0.1 -Count 4

   Destination: 10.0.0.1

Ping Source           Address          Latency Status
                                        (ms)
---- ------           -------          ------- ------
   1 SRV-DC01         10.0.0.1               1 Success
   2 SRV-DC01         10.0.0.1               1 Success
   3 SRV-DC01         10.0.0.1               1 Success
   4 SRV-DC01         10.0.0.1               1 Success

PS C:\> Test-NetConnection -ComputerName "SRV-DC01" -Port 3389

ComputerName     : SRV-DC01
RemoteAddress    : 10.0.0.10
RemotePort       : 3389
InterfaceAlias   : Ethernet0
SourceAddress    : 10.0.0.10
TcpTestSucceeded : True
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

Resultat (extrait) :

```text
PS C:\> Get-Content -Path "C:\Windows\System32\drivers\etc\hosts"
# Copyright (c) 1993-2009 Microsoft Corp.
#
# This is a sample HOSTS file used by Microsoft TCP/IP for Windows.
#
# localhost name resolution is handled within DNS itself.
#	127.0.0.1       localhost
#	::1             localhost
10.0.0.10    SRV-DC01.lab.local    SRV-DC01
10.0.0.11    SRV-01.lab.local      SRV-01

PS C:\> Select-String -Path "C:\Temp\*.log" -Pattern "error"
C:\Temp\install.log:42:  [ERROR] Failed to connect to 10.0.0.10:445
C:\Temp\install.log:87:  [ERROR] Timeout waiting for DNS response
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

Resultat (extrait) :

```text
PS C:\> Get-WindowsFeature -Name "*DNS*"

Display Name                                            Name            Install State
------------                                            ----            -------------
    [X] DNS Server                                      DNS             Installed
        [X] DNS Server Tools                            RSAT-DNS-Server Installed

PS C:\> Install-WindowsFeature -Name "DNS" -IncludeManagementTools

Success Restart Needed Exit Code      Feature Result
------- -------------- ---------      --------------
True    No             NoChangeNeeded {}
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

Resultat (extrait) :

```text
PS C:\> Get-LocalUser

Name               Enabled Description
----               ------- -----------
Administrator      True    Built-in account for administering the computer/domain
DefaultAccount     False   A user account managed by the system.
Guest              False   Built-in account for guest access to the computer/domain
labuser            True    Lab User

PS C:\> Get-LocalGroupMember -Group "Administrators"

ObjectClass Name                      PrincipalSource
----------- ----                      ---------------
User        SRV-DC01\Administrator    Local
User        SRV-DC01\labuser          Local
Group       LAB\Domain Admins         ActiveDirectory
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

Resultat (extrait) :

```text
PS C:\> Get-EventLog -LogName System -EntryType Error -Newest 10

   Index Time          EntryType   Source                 InstanceID Message
   ----- ----          ---------   ------                 ---------- -------
   14523 Feb 19 14:12  Error       DCOM                        10016 The application-specific permission settings...
   14501 Feb 19 08:45  Error       Service Control M...   3221232472 The DNS Client service terminated unexpect...
   14487 Feb 18 22:30  Error       Disk                          153 The IO operation at logical block address...
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

Resultat (extrait) :

```text
PS C:\> Get-NetFirewallProfile | Select-Object Name, Enabled

Name    Enabled
----    -------
Domain     True
Private    True
Public     True
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

Resultat (extrait de `Measure-Command`) :

```text
PS C:\> Measure-Command { Get-Process }

Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 47
Ticks             : 478923
TotalMilliseconds : 47.8923
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

!!! example "Scenario pratique"

    **Contexte** : Marc, administrateur systeme, recoit un appel : le serveur `SRV-01` (10.0.0.20) semble lent. Il doit diagnostiquer rapidement le probleme depuis `SRV-DC01`.

    **Etape 1** : Verifier la connectivite reseau vers `SRV-01`

    ```powershell
    Test-Connection -ComputerName 10.0.0.20 -Count 2
    ```

    ```text
    Ping Source           Address          Latency Status
    ---- ------           -------          ------- ------
       1 SRV-DC01         10.0.0.20              1 Success
       2 SRV-DC01         10.0.0.20              1 Success
    ```

    **Etape 2** : Verifier le temps de fonctionnement (uptime) du serveur

    ```powershell
    (Get-Date) - (Get-CimInstance Win32_OperatingSystem -ComputerName 10.0.0.20).LastBootUpTime
    ```

    ```text
    Days              : 47
    Hours             : 12
    Minutes           : 33
    ```

    Le serveur n'a pas redemarre depuis 47 jours.

    **Etape 3** : Identifier les processus consommant le plus de memoire

    ```powershell
    Get-Process -ComputerName 10.0.0.20 | Sort-Object WorkingSet64 -Descending |
        Select-Object -First 5 Name, @{N="MemoryMB"; E={[math]::Round($_.WorkingSet64/1MB,2)}}
    ```

    ```text
    Name             MemoryMB
    ----             --------
    sqlservr          2048.45
    w3wp               512.33
    svchost            256.12
    dns                128.67
    lsass               95.44
    ```

    **Etape 4** : Verifier les erreurs recentes dans les journaux

    ```powershell
    Get-EventLog -LogName Application -ComputerName 10.0.0.20 -EntryType Error -Newest 5
    ```

    Marc identifie que le processus `sqlservr` consomme plus de 2 Go de memoire. Il recommande un redemarrage planifie du service SQL et une augmentation de la RAM du serveur.

!!! danger "Erreurs courantes"

    1. **Oublier `-ErrorAction SilentlyContinue`** : lors d'un parcours recursif avec `Get-ChildItem -Recurse`, les dossiers sans permission generent des erreurs rouges en cascade. Ajoutez `-ErrorAction SilentlyContinue` pour les ignorer proprement.

    2. **Confondre `Get-EventLog` et `Get-WinEvent`** : `Get-EventLog` ne fonctionne qu'avec les journaux classiques (System, Application, Security). Pour les journaux ETW modernes, utilisez `Get-WinEvent`.

    3. **Arreter un service sans verifier ses dependances** : `Stop-Service -Name "LanmanServer"` arrete le service Serveur, ce qui coupe tous les partages de fichiers. Verifiez les dependances avec `Get-Service -Name "LanmanServer" -DependentServices`.

    4. **Utiliser `Remove-Item` sans `-Confirm` sur des dossiers importants** : `Remove-Item -Recurse` supprime tout sans confirmation. Ajoutez `-WhatIf` pour tester ou `-Confirm` pour valider chaque suppression.

    5. **Ne pas specifier `-NoTypeInformation` avec `Export-Csv`** : sans ce parametre, la premiere ligne du CSV contient `#TYPE System.ServiceProcess.ServiceController`, ce qui pollue le fichier.

## Points cles a retenir

- Chaque domaine d'administration a ses cmdlets dediees
- Le pattern est toujours `Verbe-Nom` avec des parametres nommes
- `-ErrorAction SilentlyContinue` supprime les erreurs non critiques
- `Select-Object` permet de choisir les proprietes a afficher
- `Export-Csv` et `ConvertTo-Html` pour les rapports

## Pour aller plus loin

- [Pipeline et objets](pipeline-et-objets.md) - combiner les cmdlets
- [Aide et decouverte](aide-et-decouverte.md) - explorer les cmdlets par soi-meme

