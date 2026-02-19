---
title: "Lab 09 : Automatisation PowerShell"
description: Exercice pratique - ecrire des scripts PowerShell pour automatiser les taches d'administration courantes.
tags:
  - lab
  - powershell
  - automatisation
  - intermediaire
---

# Lab 09 : Automatisation PowerShell

!!! abstract "Objectifs du lab"

    - [ ] Ecrire un script de creation d'utilisateurs en masse depuis un CSV
    - [ ] Creer un script de rapport d'inventaire systeme
    - [ ] Mettre en place une tache planifiee pour l'automatisation
    - [ ] Creer un script de surveillance avec alertes email

## Scenario

L'equipe RH fournit regulierement des listes de nouveaux employes. Le service informatique a besoin d'automatiser la creation des comptes AD et de generer des rapports d'inventaire hebdomadaires.

## Environnement requis

| Ressource | Specification |
|-----------|---------------|
| SRV-DC01 | DC avec la structure OU (Labs precedents) |
| SRV-FILE01 | Serveur pour stocker les scripts et rapports |
| CLI-W11 | Poste pour developper et tester les scripts |

## Instructions

### Partie 1 : Creation d'utilisateurs en masse

1. Creer un fichier CSV avec les donnees des employes
2. Ecrire un script qui lit le CSV et cree les comptes AD

??? success "Solution"

    Fichier CSV (`C:\Scripts\new-users.csv`) :

    ```csv
    Prenom,Nom,Service,Titre
    Alice,Moreau,Comptabilite,Comptable
    Bruno,Petit,Commercial,Commercial
    Claire,Robert,Direction,Assistante
    David,Simon,Informatique,Technicien
    ```

    Script de creation :

    ```powershell
    # Import-NewUsers.ps1
    # Creates AD users from a CSV file

    param(
        [Parameter(Mandatory)]
        [string]$CsvPath,

        [string]$DefaultPassword = "Welcome2026!",
        [string]$DomainSuffix = "winopslab.local"
    )

    # Import the CSV
    $users = Import-Csv -Path $CsvPath -Delimiter ","

    # Convert password to secure string
    $securePassword = ConvertTo-SecureString $DefaultPassword -AsPlainText -Force

    foreach ($user in $users) {
        $samAccountName = "$($user.Prenom.ToLower()).$($user.Nom.ToLower())"
        $upn = "$samAccountName@$DomainSuffix"
        $displayName = "$($user.Prenom) $($user.Nom)"
        $ouPath = "OU=$($user.Service),OU=Utilisateurs,DC=winopslab,DC=local"

        # Check if user already exists
        if (Get-ADUser -Filter "SamAccountName -eq '$samAccountName'" -ErrorAction SilentlyContinue) {
            Write-Warning "User '$samAccountName' already exists. Skipping."
            continue
        }

        # Check if the OU exists
        try {
            Get-ADOrganizationalUnit -Identity $ouPath -ErrorAction Stop | Out-Null
        }
        catch {
            Write-Warning "OU '$ouPath' does not exist. Creating it."
            New-ADOrganizationalUnit -Name $user.Service `
                -Path "OU=Utilisateurs,DC=winopslab,DC=local"
        }

        # Create the user
        New-ADUser -Name $displayName `
            -GivenName $user.Prenom `
            -Surname $user.Nom `
            -SamAccountName $samAccountName `
            -UserPrincipalName $upn `
            -DisplayName $displayName `
            -Title $user.Titre `
            -Department $user.Service `
            -Path $ouPath `
            -AccountPassword $securePassword `
            -ChangePasswordAtLogon $true `
            -Enabled $true

        # Add to the department global group
        $groupName = "GG_$($user.Service)"
        if (Get-ADGroup -Filter "Name -eq '$groupName'" -ErrorAction SilentlyContinue) {
            Add-ADGroupMember -Identity $groupName -Members $samAccountName
        }

        Write-Output "Created user: $displayName ($samAccountName) in $($user.Service)"
    }

    Write-Output "`nUser creation complete."
    ```

    Execution :

    ```powershell
    .\Import-NewUsers.ps1 -CsvPath "C:\Scripts\new-users.csv"
    ```

### Partie 2 : Rapport d'inventaire systeme

??? success "Solution"

    ```powershell
    # Get-ServerInventory.ps1
    # Generates an HTML inventory report for all domain servers

    param(
        [string]$OutputPath = "C:\Reports\ServerInventory-$(Get-Date -Format 'yyyyMMdd').html"
    )

    $servers = Get-ADComputer -Filter { OperatingSystem -like "*Server*" } -Properties OperatingSystem |
        Select-Object -ExpandProperty Name

    $inventory = foreach ($server in $servers) {
        try {
            $os = Get-CimInstance Win32_OperatingSystem -ComputerName $server -ErrorAction Stop
            $cpu = Get-CimInstance Win32_Processor -ComputerName $server -ErrorAction Stop
            $disk = Get-CimInstance Win32_LogicalDisk -ComputerName $server -Filter "DriveType=3" -ErrorAction Stop

            [PSCustomObject]@{
                Server       = $server
                OS           = $os.Caption
                Uptime       = ((Get-Date) - $os.LastBootUpTime).ToString("dd\.hh\:mm")
                CPU          = $cpu.Name
                Cores        = $cpu.NumberOfCores
                TotalRAM_GB  = [math]::Round($os.TotalVisibleMemorySize / 1MB, 1)
                FreeRAM_GB   = [math]::Round($os.FreePhysicalMemory / 1MB, 1)
                DiskFree_GB  = ($disk | ForEach-Object {
                    "$($_.DeviceID) $([math]::Round($_.FreeSpace/1GB,1))GB"
                }) -join " | "
                Status       = "Online"
            }
        }
        catch {
            [PSCustomObject]@{
                Server = $server; OS = "N/A"; Uptime = "N/A"
                CPU = "N/A"; Cores = "N/A"; TotalRAM_GB = "N/A"
                FreeRAM_GB = "N/A"; DiskFree_GB = "N/A"; Status = "Offline"
            }
        }
    }

    # Generate HTML report
    $html = $inventory | ConvertTo-Html -Title "Server Inventory - $(Get-Date -Format 'yyyy-MM-dd')" `
        -PreContent "<h1>Server Inventory Report</h1><p>Generated: $(Get-Date)</p>"

    $html | Out-File -FilePath $OutputPath -Encoding UTF8
    Write-Output "Report saved to: $OutputPath"
    ```

### Partie 3 : Tache planifiee

??? success "Solution"

    ```powershell
    # Create a scheduled task to run the inventory report weekly
    $action = New-ScheduledTaskAction `
        -Execute "PowerShell.exe" `
        -Argument '-NoProfile -ExecutionPolicy Bypass -File "C:\Scripts\Get-ServerInventory.ps1"'

    $trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Monday -At "07:00"

    $principal = New-ScheduledTaskPrincipal -UserId "WINOPSLAB\admin.lab" `
        -LogonType Password -RunLevel Highest

    Register-ScheduledTask -TaskName "Weekly-ServerInventory" `
        -Action $action -Trigger $trigger -Principal $principal `
        -Description "Generate weekly server inventory report"

    # Verify the task
    Get-ScheduledTask -TaskName "Weekly-ServerInventory"

    # Run the task immediately to test
    Start-ScheduledTask -TaskName "Weekly-ServerInventory"
    ```

### Partie 4 : Script de surveillance avec alertes

??? success "Solution"

    ```powershell
    # Watch-ServerHealth.ps1
    # Monitors server health and logs warnings

    param(
        [int]$CpuThreshold = 90,
        [int]$DiskThresholdPercent = 15,
        [int]$MemoryThresholdMB = 500
    )

    $servers = @("SRV-DC01", "SRV-FILE01", "SRV-WEB01")
    $alerts = @()

    foreach ($server in $servers) {
        try {
            # Check CPU
            $cpu = (Get-Counter "\\$server\Processor(_Total)\% Processor Time" `
                -ErrorAction Stop).CounterSamples[0].CookedValue
            if ($cpu -gt $CpuThreshold) {
                $alerts += "$server - CPU at $([math]::Round($cpu,1))% (threshold: $CpuThreshold%)"
            }

            # Check memory
            $mem = (Get-Counter "\\$server\Memory\Available MBytes" `
                -ErrorAction Stop).CounterSamples[0].CookedValue
            if ($mem -lt $MemoryThresholdMB) {
                $alerts += "$server - Available RAM: ${mem}MB (threshold: ${MemoryThresholdMB}MB)"
            }

            # Check disk space
            $disks = Get-CimInstance Win32_LogicalDisk -ComputerName $server `
                -Filter "DriveType=3" -ErrorAction Stop
            foreach ($disk in $disks) {
                $freePercent = [math]::Round(($disk.FreeSpace / $disk.Size) * 100, 1)
                if ($freePercent -lt $DiskThresholdPercent) {
                    $alerts += "$server - Disk $($disk.DeviceID) at ${freePercent}% free (threshold: $DiskThresholdPercent%)"
                }
            }
        }
        catch {
            $alerts += "$server - UNREACHABLE: $($_.Exception.Message)"
        }
    }

    # Output alerts
    if ($alerts.Count -gt 0) {
        Write-Warning "=== HEALTH ALERTS ==="
        $alerts | ForEach-Object { Write-Warning $_ }

        # Log to Event Log
        foreach ($alert in $alerts) {
            Write-EventLog -LogName Application -Source "ServerHealth" `
                -EventId 9001 -EntryType Warning -Message $alert
        }
    }
    else {
        Write-Output "All servers healthy."
    }
    ```

## Verification

!!! question "Questions de validation"

    1. Pourquoi utiliser `-ChangePasswordAtLogon $true` lors de la creation d'utilisateurs ?
    2. Quelle est la difference entre une tache planifiee executee en tant que SYSTEM et en tant qu'utilisateur ?
    3. Comment gerer les erreurs dans un script qui interroge des serveurs distants ?
    4. Pourquoi est-il important de valider les donnees d'entree (CSV) avant de creer des comptes ?

??? success "Reponses"

    1. Cela oblige l'utilisateur a changer son mot de passe initial lors de la premiere connexion,
       garantissant que seul l'utilisateur connait son mot de passe final.
    2. **SYSTEM** : s'execute meme si personne n'est connecte, avec les droits de la machine.
       **Utilisateur** : s'execute avec les droits du compte specifie, peut necesiter que l'utilisateur
       soit connecte (sauf si "Run whether user is logged on or not" est active).
    3. Utiliser `try/catch` avec `-ErrorAction Stop` pour capturer les erreurs et continuer
       le traitement des autres serveurs. Ne jamais laisser une erreur bloquer tout le script.
    4. Un CSV malformee (caracteres speciaux, champs manquants, doublons) peut creer des
       comptes invalides ou provoquer des erreurs. La validation en amont evite ces problemes.

## Nettoyage

```powershell
# Remove the scheduled task
Unregister-ScheduledTask -TaskName "Weekly-ServerInventory" -Confirm:$false

# Remove test users created from CSV
Import-Csv "C:\Scripts\new-users.csv" | ForEach-Object {
    $sam = "$($_.Prenom.ToLower()).$($_.Nom.ToLower())"
    Remove-ADUser -Identity $sam -Confirm:$false -ErrorAction SilentlyContinue
}
```

## Prochaine etape

:material-arrow-right: [Lab 10 : Supervision](lab-10-supervision.md)
