---
title: Comptes ordinateurs
description: Gestion des comptes ordinateurs et jonction au domaine Active Directory.
tags:
  - active-directory
  - adds
  - intermediaire
---

# Comptes ordinateurs

!!! info "Niveau : Intermediaire"

    Temps estime : 15 minutes

## Jonction au domaine

Lorsqu'un ordinateur rejoint le domaine, un **compte ordinateur** est automatiquement cree dans Active Directory.

### Prerequis

- Le poste doit pouvoir resoudre le nom DNS du domaine
- Le DNS du poste doit pointer vers un DC
- Un compte avec les droits de jonction au domaine

### Joindre un poste au domaine

=== "PowerShell"

    ```powershell
    # Join domain and restart
    Add-Computer `
        -DomainName "lab.local" `
        -OUPath "OU=Postes,OU=Ordinateurs,DC=lab,DC=local" `
        -Credential (Get-Credential "LAB\Administrator") `
        -Restart

    # Join domain without specifying an OU (goes to default Computers container)
    Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
    ```

=== "Interface graphique"

    1. **Parametres** > **Systeme** > **A propos** > **Renommer ce PC (avance)**
    2. Cliquer sur **Modifier**
    3. Selectionner **Domaine** et saisir `lab.local`
    4. Fournir les identifiants d'un compte autorise
    5. Redemarrer

### Retirer un poste du domaine

```powershell
# Remove from domain (returns to workgroup)
Remove-Computer -UnjoinDomainCredential (Get-Credential "LAB\Administrator") -Restart
```

## Gestion des comptes ordinateurs

```powershell
# List all computer accounts
Get-ADComputer -Filter * | Select-Object Name, Enabled, DistinguishedName

# Find a specific computer
Get-ADComputer -Identity "PC-JEAN-01" -Properties *

# Find computers in a specific OU
Get-ADComputer -Filter * -SearchBase "OU=Postes,OU=Ordinateurs,DC=lab,DC=local"

# Find disabled computer accounts
Get-ADComputer -Filter { Enabled -eq $false }

# Find computers that haven't logged in for 90 days
$inactiveDate = (Get-Date).AddDays(-90)
Get-ADComputer -Filter { LastLogonDate -lt $inactiveDate } -Properties LastLogonDate |
    Select-Object Name, LastLogonDate

# Move a computer to another OU
Move-ADObject -Identity "CN=PC-JEAN-01,OU=Postes,OU=Ordinateurs,DC=lab,DC=local" `
    -TargetPath "OU=Portables,OU=Ordinateurs,DC=lab,DC=local"

# Disable a computer account
Disable-ADAccount -Identity "PC-OLD-01$"

# Delete a computer account
Remove-ADComputer -Identity "PC-OLD-01" -Confirm:$false
```

!!! note "Le `$` dans le SamAccountName"

    Le SamAccountName d'un ordinateur se termine toujours par `$`.
    Ex: `PC-JEAN-01$`. C'est ainsi qu'AD distingue les comptes ordinateurs
    des comptes utilisateurs.

## Pre-staging de comptes ordinateurs

Vous pouvez pre-creer un compte ordinateur avant la jonction :

```powershell
# Pre-create a computer account in a specific OU
New-ADComputer `
    -Name "PC-NEW-01" `
    -SamAccountName "PC-NEW-01$" `
    -Path "OU=Postes,OU=Ordinateurs,DC=lab,DC=local" `
    -Enabled $true
```

!!! tip "Avantage du pre-staging"

    En pre-creant le compte dans la bonne OU, le poste recevra immediatement
    les bonnes GPO des la jonction, sans avoir a etre deplace manuellement.

## Nettoyage des comptes inactifs

```powershell
# Find and report stale computer accounts (90+ days inactive)
$staleDate = (Get-Date).AddDays(-90)
$staleComputers = Get-ADComputer -Filter { LastLogonDate -lt $staleDate } `
    -Properties LastLogonDate, OperatingSystem |
    Select-Object Name, LastLogonDate, OperatingSystem, DistinguishedName

# Export report
$staleComputers | Export-Csv -Path "C:\Temp\stale-computers.csv" -NoTypeInformation

# Disable stale accounts (review before deleting)
$staleComputers | ForEach-Object {
    Disable-ADAccount -Identity $_.DistinguishedName
    Write-Host "Disabled: $($_.Name) - Last logon: $($_.LastLogonDate)"
}
```

## Points cles a retenir

- La jonction au domaine cree automatiquement un compte ordinateur dans AD
- Le DNS du poste doit pointer vers un DC pour la jonction
- Utilisez `redircmp` pour rediriger le conteneur par defaut vers votre OU
- Le pre-staging garantit que le poste atterrit dans la bonne OU
- Nettoyez regulierement les comptes inactifs

## Pour aller plus loin

- [Structure des OU](structure-ou.md) - ou placer les comptes ordinateurs
- [GPO - Concepts](../gpo/concepts-gpo.md) - appliquer des strategies aux ordinateurs
