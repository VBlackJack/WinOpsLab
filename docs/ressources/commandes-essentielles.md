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
title: "Commandes essentielles"
description: Aide-memoire des commandes PowerShell et CMD indispensables pour l'administration Windows Server.
tags:
  - ressources
  - reference
---

# Commandes essentielles

<span class="level-beginner">Reference</span> · Temps estime : 10 minutes

---

Aide-memoire organise par categorie des commandes PowerShell et CMD les plus utilisees en administration Windows Server.

!!! tip "Convention"

    Les commandes sont presentees en **PowerShell** sauf mention contraire.
    Les equivalents CMD sont indiques quand ils existent et restent couramment utilises.

## Systeme et informations

### Informations generales

```powershell
# OS version and system information
Get-ComputerInfo | Select-Object OsName, OsVersion, OsBuildNumber, CsName
systeminfo                        # CMD - Full system information

# Uptime
(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime

# Installed hotfixes
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10

# Windows features and roles
Get-WindowsFeature | Where-Object Installed -eq $true

# Environment variables
Get-ChildItem Env:
[System.Environment]::GetEnvironmentVariable("PATH", "Machine")
```

### Services

```powershell
# List all services
Get-Service | Sort-Object Status, Name

# Filter by status
Get-Service | Where-Object Status -eq "Running"
Get-Service | Where-Object Status -eq "Stopped"

# Manage a service
Start-Service -Name "wuauserv"
Stop-Service -Name "wuauserv" -Force
Restart-Service -Name "wuauserv"
Set-Service -Name "wuauserv" -StartupType Automatic

# Service dependencies
Get-Service -Name "DNS" -DependentServices
Get-Service -Name "DNS" -RequiredServices
```

### Processus

```powershell
# List processes sorted by CPU
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, Id, CPU, WorkingSet64

# Find a process by name
Get-Process -Name "dns*"

# Stop a process
Stop-Process -Name "notepad" -Force
Stop-Process -Id 1234 -Force

# CMD equivalents
tasklist /svc                     # List processes with services
taskkill /PID 1234 /F             # Kill by PID
```

## Reseau

### Configuration IP

```powershell
# Show IP configuration
Get-NetIPConfiguration
Get-NetIPAddress | Where-Object AddressFamily -eq "IPv4"
ipconfig /all                     # CMD

# Set static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "10.10.10.10" `
    -PrefixLength 24 -DefaultGateway "10.10.10.1"

# Set DNS servers
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses "10.10.10.10", "10.10.10.11"

# Release / renew DHCP
ipconfig /release                 # CMD
ipconfig /renew                   # CMD

# Flush DNS cache
Clear-DnsClientCache
ipconfig /flushdns                # CMD
```

### Diagnostic reseau

```powershell
# Connectivity test (ping equivalent with port test)
Test-NetConnection -ComputerName "dc01" -Port 389
Test-NetConnection -ComputerName "8.8.8.8" -TraceRoute

# Classic ping
Test-Connection -ComputerName "dc01" -Count 4
ping dc01                         # CMD

# DNS resolution
Resolve-DnsName "dc01.technova.local"
Resolve-DnsName "_ldap._tcp.dc._msdcs.technova.local" -Type SRV
nslookup dc01.technova.local      # CMD

# Route and ARP
Get-NetRoute -AddressFamily IPv4
Get-NetNeighbor                   # ARP table
route print                       # CMD
arp -a                            # CMD

# Active connections
Get-NetTCPConnection -State Established | Sort-Object RemotePort
netstat -ano                      # CMD
```

### Pare-feu

```powershell
# List firewall rules
Get-NetFirewallRule | Where-Object Enabled -eq "True" |
    Select-Object DisplayName, Direction, Action, Profile

# Create inbound rule
New-NetFirewallRule -DisplayName "Allow HTTPS" `
    -Direction Inbound -Protocol TCP -LocalPort 443 -Action Allow

# Enable/disable a rule
Enable-NetFirewallRule -DisplayName "Allow HTTPS"
Disable-NetFirewallRule -DisplayName "Allow HTTPS"

# Remove a rule
Remove-NetFirewallRule -DisplayName "Allow HTTPS"

# Check specific port
Get-NetFirewallPortFilter | Where-Object LocalPort -eq 3389
```

## Active Directory

### Gestion du domaine

```powershell
# Domain information
Get-ADDomain | Select-Object Name, DomainMode, PDCEmulator
Get-ADForest | Select-Object Name, ForestMode, RootDomain

# Domain controllers
Get-ADDomainController -Filter * |
    Select-Object Name, IPv4Address, Site, IsGlobalCatalog, OperationMasterRoles

# FSMO roles
netdom query fsmo                 # CMD

# Replication status
repadmin /replsummary
repadmin /showrepl
Get-ADReplicationPartnerMetadata -Target "dc01" -Scope Server

# Trust relationships
Get-ADTrust -Filter *
netdom trust technova.local /verify /domain:partner.local  # CMD
```

### Utilisateurs

```powershell
# Search users
Get-ADUser -Filter * -Properties Department, Title |
    Select-Object SamAccountName, Name, Department, Enabled

# Find specific user
Get-ADUser -Identity "jdupont" -Properties *
Get-ADUser -Filter 'Name -like "Dupont*"'

# Create user
New-ADUser -Name "Jean Dupont" `
    -SamAccountName "jdupont" `
    -UserPrincipalName "jdupont@technova.local" `
    -GivenName "Jean" -Surname "Dupont" `
    -Path "OU=Utilisateurs,DC=technova,DC=local" `
    -AccountPassword (ConvertTo-SecureString "P@ssw0rd!" -AsPlainText -Force) `
    -ChangePasswordAtLogon $true `
    -Enabled $true

# Modify user
Set-ADUser -Identity "jdupont" -Department "IT" -Title "Administrateur"

# Disable / enable
Disable-ADAccount -Identity "jdupont"
Enable-ADAccount -Identity "jdupont"

# Unlock account
Unlock-ADAccount -Identity "jdupont"

# Reset password
Set-ADAccountPassword -Identity "jdupont" `
    -NewPassword (ConvertTo-SecureString "NewP@ss1!" -AsPlainText -Force) `
    -Reset

# Bulk import from CSV
Import-Csv "C:\Users.csv" | ForEach-Object {
    New-ADUser -Name "$($_.GivenName) $($_.Surname)" `
        -SamAccountName $_.SamAccountName `
        -UserPrincipalName "$($_.SamAccountName)@technova.local" `
        -GivenName $_.GivenName -Surname $_.Surname `
        -Department $_.Department `
        -Path $_.OU `
        -AccountPassword (ConvertTo-SecureString $_.Password -AsPlainText -Force) `
        -Enabled $true
}
```

### Groupes

```powershell
# List groups
Get-ADGroup -Filter * | Select-Object Name, GroupScope, GroupCategory

# Create group
New-ADGroup -Name "GG_IT" -GroupScope Global -GroupCategory Security `
    -Path "OU=Groupes,DC=technova,DC=local"

# Add / remove members
Add-ADGroupMember -Identity "GG_IT" -Members "jdupont", "mmartin"
Remove-ADGroupMember -Identity "GG_IT" -Members "jdupont" -Confirm:$false

# List group members
Get-ADGroupMember -Identity "GG_IT" | Select-Object Name, ObjectClass

# List groups of a user
Get-ADPrincipalGroupMembership -Identity "jdupont" | Select-Object Name
```

### Unites d'organisation

```powershell
# List OUs
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName

# Create OU
New-ADOrganizationalUnit -Name "Serveurs" `
    -Path "DC=technova,DC=local" `
    -ProtectedFromAccidentalDeletion $true

# Move an object to another OU
Move-ADObject -Identity "CN=Jean Dupont,OU=Temp,DC=technova,DC=local" `
    -TargetPath "OU=IT,DC=technova,DC=local"

# Search in a specific OU
Get-ADUser -Filter * -SearchBase "OU=IT,DC=technova,DC=local"
```

### Ordinateurs

```powershell
# List computers
Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonTimestamp |
    Select-Object Name, OperatingSystem, Enabled,
    @{N='LastLogon'; E={[DateTime]::FromFileTime($_.LastLogonTimestamp)}}

# Find inactive computers (90 days)
$threshold = (Get-Date).AddDays(-90)
Get-ADComputer -Filter 'LastLogonTimestamp -lt $threshold' -Properties LastLogonTimestamp

# Test secure channel
Test-ComputerSecureChannel -Verbose

# Reset computer account
Reset-ComputerMachinePassword -Server "dc01.technova.local"
```

## DNS

```powershell
# List DNS zones
Get-DnsServerZone

# List records in a zone
Get-DnsServerResourceRecord -ZoneName "technova.local"

# Add records
Add-DnsServerResourceRecordA -Name "srv-app01" -ZoneName "technova.local" `
    -IPv4Address "10.10.10.40"
Add-DnsServerResourceRecordCName -Name "intranet" -ZoneName "technova.local" `
    -HostNameAlias "srv-web01.technova.local"
Add-DnsServerResourceRecord -ZoneName "technova.local" -MX `
    -Name "." -MailExchange "mail.technova.local" -Preference 10

# Remove a record
Remove-DnsServerResourceRecord -ZoneName "technova.local" `
    -RRType A -Name "srv-old01" -Force

# Create reverse lookup zone
Add-DnsServerPrimaryZone -NetworkID "10.10.10.0/24" -ReplicationScope Domain

# Add PTR record
Add-DnsServerResourceRecordPtr -Name "40" -ZoneName "10.10.10.in-addr.arpa" `
    -PtrDomainName "srv-app01.technova.local"

# Conditional forwarder
Add-DnsServerConditionalForwarderZone -Name "partner.local" `
    -MasterServers "10.20.20.10"

# Clear DNS server cache
Clear-DnsServerCache -Force
```

## DHCP

```powershell
# List scopes
Get-DhcpServerv4Scope

# Create a scope
Add-DhcpServerv4Scope -Name "LAN Principal" `
    -StartRange 10.10.10.100 -EndRange 10.10.10.250 `
    -SubnetMask 255.255.255.0 -State Active

# Set scope options
Set-DhcpServerv4OptionValue -ScopeId "10.10.10.0" `
    -Router "10.10.10.1" `
    -DnsServer "10.10.10.10", "10.10.10.11" `
    -DnsDomain "technova.local"

# Add exclusion range
Add-DhcpServerv4ExclusionRange -ScopeId "10.10.10.0" `
    -StartRange 10.10.10.10 -EndRange 10.10.10.50

# Add reservation
Add-DhcpServerv4Reservation -ScopeId "10.10.10.0" `
    -IPAddress "10.10.10.60" -ClientId "00-15-5D-01-02-03" `
    -Name "Printer-RDC" -Description "Imprimante RDC"

# List active leases
Get-DhcpServerv4Lease -ScopeId "10.10.10.0" | Where-Object AddressState -eq "Active"

# Authorize DHCP in AD
Add-DhcpServerInDC -DnsName "dc01.technova.local" -IPAddress "10.10.10.10"
```

## GPO (Strategies de groupe)

```powershell
# List all GPOs
Get-GPO -All | Select-Object DisplayName, GpoStatus, CreationTime

# Create a GPO
New-GPO -Name "SEC-PasswordPolicy" -Comment "Password complexity requirements"

# Link GPO to OU
New-GPLink -Name "SEC-PasswordPolicy" `
    -Target "OU=Utilisateurs,DC=technova,DC=local"

# GPO report (HTML)
Get-GPOReport -Name "SEC-PasswordPolicy" -ReportType Html `
    -Path "C:\Reports\GPO-PasswordPolicy.html"

# All GPOs report
Get-GPO -All | ForEach-Object {
    Get-GPOReport -Guid $_.Id -ReportType Html `
        -Path "C:\Reports\GPO-$($_.DisplayName).html"
}

# Force GPO update on a remote computer
Invoke-GPUpdate -Computer "CLI-W11" -Force

# GPO result for a user/computer
gpresult /r                       # CMD - Summary
gpresult /h "C:\gpresult.html"    # CMD - HTML report
Get-GPResultantSetOfPolicy -ReportType Html -Path "C:\rsop.html"

# Backup all GPOs
Backup-GPO -All -Path "C:\GPOBackup"

# Restore a GPO
Restore-GPO -Name "SEC-PasswordPolicy" -Path "C:\GPOBackup"
```

## Partage de fichiers et permissions

### Partages SMB

```powershell
# List shares
Get-SmbShare | Where-Object Special -eq $false

# Create a share
New-SmbShare -Name "Partage_IT" -Path "D:\Partages\IT" `
    -FullAccess "TECHNOVA\DL_IT_FullControl" `
    -ReadAccess "TECHNOVA\DL_IT_Read"

# Modify share permissions
Grant-SmbShareAccess -Name "Partage_IT" `
    -AccountName "TECHNOVA\DL_IT_Modify" -AccessRight Change -Force

# Remove a share
Remove-SmbShare -Name "Partage_IT" -Force

# Connected sessions
Get-SmbSession | Select-Object ClientComputerName, ClientUserName, NumOpens
Get-SmbOpenFile | Select-Object ClientComputerName, Path
```

### Permissions NTFS

```powershell
# View permissions
Get-Acl "D:\Partages\IT" | Format-List
(Get-Acl "D:\Partages\IT").Access | Format-Table IdentityReference, FileSystemRights, AccessControlType

# icacls (CMD - often more practical)
icacls "D:\Partages\IT"                                  # View
icacls "D:\Partages\IT" /grant "TECHNOVA\GG_IT:(OI)(CI)M"  # Modify
icacls "D:\Partages\IT" /remove "TECHNOVA\GG_Temp"         # Remove
icacls "D:\Partages\IT" /reset /T                           # Reset inheritance

# Disable inheritance
$acl = Get-Acl "D:\Partages\Direction"
$acl.SetAccessRuleProtection($true, $true)  # $true = copy inherited
Set-Acl "D:\Partages\Direction" $acl

# Take ownership
takeown /f "D:\Partages\Lost" /r /d Y      # CMD
```

## Stockage

```powershell
# Disk information
Get-Disk | Select-Object Number, FriendlyName, Size, PartitionStyle
Get-Partition | Select-Object DiskNumber, PartitionNumber, DriveLetter, Size
Get-Volume | Select-Object DriveLetter, FileSystemLabel, Size, SizeRemaining

# Initialize a new disk
Initialize-Disk -Number 1 -PartitionStyle GPT

# Create partition and format
New-Partition -DiskNumber 1 -UseMaximumSize -DriveLetter D
Format-Volume -DriveLetter D -FileSystem NTFS -NewFileSystemLabel "Data"

# Extend a volume
Resize-Partition -DriveLetter D -Size (Get-PartitionSupportedSize -DriveLetter D).SizeMax

# VHDX management (Hyper-V)
New-VHD -Path "C:\VMs\Data.vhdx" -SizeBytes 100GB -Dynamic
Mount-VHD -Path "C:\VMs\Data.vhdx"
Dismount-VHD -Path "C:\VMs\Data.vhdx"
Optimize-VHD -Path "C:\VMs\Data.vhdx" -Mode Full
```

## Hyper-V

```powershell
# List VMs
Get-VM | Select-Object Name, State, CPUUsage, MemoryAssigned, Uptime

# Create a VM
New-VM -Name "SRV-TEST" -MemoryStartupBytes 2GB `
    -NewVHDPath "C:\VMs\SRV-TEST.vhdx" -NewVHDSizeBytes 60GB `
    -SwitchName "vSwitch-LAN" -Generation 2

# Start / stop
Start-VM -Name "SRV-TEST"
Stop-VM -Name "SRV-TEST"
Stop-VM -Name "SRV-TEST" -Force     # Force power off

# Checkpoints
Checkpoint-VM -Name "SRV-TEST" -SnapshotName "Before-Update"
Restore-VMCheckpoint -Name "Before-Update" -VMName "SRV-TEST" -Confirm:$false
Remove-VMCheckpoint -VMName "SRV-TEST" -Name "Before-Update"

# Modify resources
Set-VM -Name "SRV-TEST" -ProcessorCount 4 -MemoryStartupBytes 4GB
Add-VMNetworkAdapter -VMName "SRV-TEST" -SwitchName "vSwitch-DMZ"

# Virtual switch
Get-VMSwitch
New-VMSwitch -Name "vSwitch-LAN" -SwitchType Internal
```

## Sauvegarde et restauration

```powershell
# Install Windows Server Backup
Install-WindowsFeature -Name Windows-Server-Backup

# Configure and start system state backup
$policy = New-WBPolicy
Add-WBSystemState -Policy $policy
$target = New-WBBackupTarget -VolumePath "E:"
Add-WBBackupTarget -Policy $policy -Target $target
Set-WBSchedule -Policy $policy -Schedule 03:00
Set-WBPolicy -Policy $policy -Force

# Manual backup
Start-WBBackup -Policy (Get-WBPolicy)

# Check backup status
Get-WBSummary
Get-WBJob -Previous 5

# List backup versions
Get-WBBackupSet

# Restore a file
Start-WBFileRecovery -BackupSet (Get-WBBackupSet | Select-Object -Last 1) `
    -FileToRecover "D:\Partages\IT\rapport.docx" `
    -TargetPath "C:\Restore"
```

## Supervision et performance

```powershell
# Real-time counters
Get-Counter "\Processor(_Total)\% Processor Time" -Continuous
Get-Counter "\Memory\Available MBytes"
Get-Counter "\LogicalDisk(*)\% Free Space"

# Multiple counters (single sample)
Get-Counter -Counter @(
    "\Processor(_Total)\% Processor Time"
    "\Memory\Available MBytes"
    "\PhysicalDisk(_Total)\Avg. Disk Queue Length"
) -SampleInterval 5 -MaxSamples 12

# Event logs
Get-WinEvent -LogName System -MaxEvents 20
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4625
    StartTime = (Get-Date).AddHours(-24)
}

# Data collector sets
logman create counter "Baseline" -c "\Processor(_Total)\% Processor Time" `
    "\Memory\Available MBytes" -si 15 -o "C:\PerfLogs\Baseline" -f bin
logman start "Baseline"
logman stop "Baseline"
logman delete "Baseline"
```

## Administration a distance

```powershell
# WinRM configuration
Enable-PSRemoting -Force
winrm quickconfig                 # CMD

# Test WinRM connectivity
Test-WSMan -ComputerName "srv-file01"

# Remote session
Enter-PSSession -ComputerName "srv-file01"
Exit-PSSession

# Execute command on remote server
Invoke-Command -ComputerName "srv-file01" -ScriptBlock {
    Get-Service | Where-Object Status -eq "Running"
}

# Execute on multiple servers
Invoke-Command -ComputerName "srv-file01", "srv-web01" -ScriptBlock {
    Get-ComputerInfo | Select-Object CsName, OsName, OsVersion
}

# Copy files to/from remote server
Copy-Item -Path "C:\Scripts\deploy.ps1" `
    -Destination "C:\Scripts\" `
    -ToSession (New-PSSession -ComputerName "srv-file01")
```

## Divers utilitaires

### Taches planifiees

```powershell
# List scheduled tasks
Get-ScheduledTask | Where-Object State -eq "Ready"

# Create a scheduled task
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" `
    -Argument "-File C:\Scripts\Backup-Check.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "07:00"
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest
Register-ScheduledTask -TaskName "Daily-BackupCheck" `
    -Action $action -Trigger $trigger -Principal $principal

# Run a task immediately
Start-ScheduledTask -TaskName "Daily-BackupCheck"

# Remove a task
Unregister-ScheduledTask -TaskName "Daily-BackupCheck" -Confirm:$false
```

### Export et rapports

```powershell
# Export to CSV
Get-ADUser -Filter * | Export-Csv "C:\Reports\Users.csv" -NoTypeInformation -Encoding UTF8

# Export to HTML
Get-Service | ConvertTo-Html -Title "Services" | Out-File "C:\Reports\Services.html"

# Export to JSON
Get-Process | Select-Object Name, Id, CPU | ConvertTo-Json | Out-File "C:\Reports\Processes.json"

# Transcript (log all console output)
Start-Transcript -Path "C:\Logs\session-$(Get-Date -Format 'yyyyMMdd-HHmmss').log"
# ... commands ...
Stop-Transcript
```

### Raccourcis clavier utiles

| Raccourci | Action |
|-----------|--------|
| ++tab++ | Auto-completion PowerShell |
| ++ctrl+r++ | Recherche dans l'historique |
| `Get-Command *dns*` | Trouver des commandes par mot-cle |
| `Get-Help Get-ADUser -Full` | Aide complete d'une commande |
| `Get-ADUser -Filter * \| Get-Member` | Decouvrir les proprietes disponibles |

## Points cles a retenir

- **PowerShell** est l'outil principal d'administration Windows Server moderne
- Les cmdlets suivent la convention **Verbe-Nom** (`Get-`, `Set-`, `New-`, `Remove-`)
- Utiliser `Get-Help <commande> -Examples` pour voir les exemples d'utilisation
- Utiliser `Get-Command *motcle*` pour decouvrir les commandes disponibles
- Le **pipe** (`|`) permet de chainer les commandes pour filtrer et formater les resultats
- Les commandes CMD restent utiles pour certains diagnostics rapides (`ipconfig`, `netstat`, `gpresult`)

## Pour aller plus loin

- [Glossaire des termes](glossaire.md)
- [Liens utiles](liens-utiles.md)
- [Lab 09 : Automatisation PowerShell](../labs/exercices/lab-09-automatisation.md)

