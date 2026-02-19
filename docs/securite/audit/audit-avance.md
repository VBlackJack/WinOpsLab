---
title: "Audit avance"
description: "Audit d'acces aux objets via SACL, surveillance des acces fichiers et modifications Active Directory sur Windows Server 2022."
tags:
  - securite
  - audit
  - sacl
  - active-directory
  - acces-fichiers
---

# Audit avance

!!! info "Niveau : avance | Temps estime : 40 minutes"

L'audit avance va au-dela des evenements de connexion et de gestion de comptes. Il permet de tracer les acces aux fichiers, aux objets Active Directory et aux cles de registre grace aux SACL (System Access Control Lists).

---

## Comprendre les SACL

Une **SACL** (System Access Control List) est une liste de controle d'acces definie sur un objet (fichier, dossier, objet AD, cle de registre) qui specifie quels acces doivent etre journalises.

```mermaid
flowchart TD
    A[Objet protege] --> B{DACL}
    A --> C{SACL}
    B --> D[Qui a acces ?<br/>Autorise / Refuse]
    C --> E[Quels acces journaliser ?<br/>Succes / Echec]
    E --> F[Evenement dans le journal Security]
```

### Difference entre DACL et SACL

| Liste | Role | Resultat |
|-------|------|----------|
| **DACL** | Definit les permissions (autoriser/refuser) | Acces accorde ou refuse |
| **SACL** | Definit les acces a auditer | Evenement journalise dans Security |

!!! tip "Prerequis"

    Pour que les SACL generent des evenements, l'**audit d'acces aux objets** doit etre active dans la politique d'audit :

    ```
    Computer Configuration > Policies > Windows Settings
      > Security Settings > Advanced Audit Policy Configuration
        > Object Access
          > Audit File System : Success, Failure
          > Audit Registry : Failure
    ```

---

## Audit d'acces aux fichiers

### Etape 1 : activer l'audit Object Access

```powershell
# Enable file system auditing via auditpol
auditpol /set /subcategory:"File System" /success:enable /failure:enable

# Verify the setting
auditpol /get /subcategory:"File System"
```

### Etape 2 : configurer la SACL sur le dossier cible

```powershell
# Add an audit rule on a folder (audit Everyone for delete and write operations)
$folderPath = "D:\Partage\Confidentiel"

$acl = Get-Acl -Path $folderPath -Audit

# Create audit rule: who, what, inheritance, propagation, type
$auditRule = New-Object System.Security.AccessControl.FileSystemAuditRule(
    "Everyone",                          # Principal to audit
    "Delete,Write,ChangePermissions",    # Operations to audit
    "ContainerInherit,ObjectInherit",    # Inheritance flags
    "None",                              # Propagation flags
    "Success,Failure"                    # Audit type
)

$acl.AddAuditRule($auditRule)
Set-Acl -Path $folderPath -AclObject $acl

# Verify the SACL
(Get-Acl -Path $folderPath -Audit).Audit | Format-Table -AutoSize
```

### Etape 3 : analyser les evenements generes

Les acces aux fichiers generent l'Event ID **4663** (tentative d'acces a un objet) :

```powershell
# Query file access audit events
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4663
} -MaxEvents 20 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            Account    = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
            ObjectName = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ObjectName' } | Select-Object -ExpandProperty '#text'
            AccessMask = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AccessMask' } | Select-Object -ExpandProperty '#text'
            ProcessName= $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ProcessName' } | Select-Object -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

### Event IDs lies a l'acces fichiers

| Event ID | Description |
|----------|-------------|
| **4656** | Handle demande sur un objet |
| **4658** | Handle ferme |
| **4660** | Objet supprime |
| **4663** | Tentative d'acces a un objet |
| **4670** | Permissions d'un objet modifiees |

### Masques d'acces courants

| Access Mask | Signification |
|-------------|---------------|
| `0x1` | ReadData / ListDirectory |
| `0x2` | WriteData / AddFile |
| `0x4` | AppendData / AddSubdirectory |
| `0x10000` | Delete |
| `0x40000` | WriteDACL (modification des permissions) |
| `0x80000` | WriteOwner (prise de propriete) |

---

## Audit des objets Active Directory

### Etape 1 : activer l'audit Directory Service

```powershell
# Enable DS Access auditing
auditpol /set /subcategory:"Directory Service Access" /success:enable /failure:enable
auditpol /set /subcategory:"Directory Service Changes" /success:enable
```

### Etape 2 : configurer la SACL sur les objets AD

Via ADSI Edit ou la console Active Directory Users and Computers (onglet Securite > Avance > Audit) :

```powershell
# Example: add audit entry on an OU using PowerShell
Import-Module ActiveDirectory

$ouDN = "OU=Admins,DC=lab,DC=local"
$acl = Get-Acl -Path "AD:\$ouDN" -Audit

# Audit modifications by Everyone on user objects
$identityRef = New-Object System.Security.Principal.SecurityIdentifier("S-1-1-0")  # Everyone
$adRight = [System.DirectoryServices.ActiveDirectoryRights]::WriteProperty
$auditFlags = [System.Security.AccessControl.AuditFlags]::Success
$inheritanceType = [System.DirectoryServices.ActiveDirectorySecurityInheritance]::Descendents

# GUID for User object class
$userObjectGuid = [Guid]"bf967aba-0de6-11d0-a285-00aa003049e2"

$auditRule = New-Object System.DirectoryServices.ActiveDirectoryAuditRule(
    $identityRef,
    $adRight,
    $auditFlags,
    $inheritanceType,
    $userObjectGuid
)

$acl.AddAuditRule($auditRule)
Set-Acl -Path "AD:\$ouDN" -AclObject $acl
```

### Event IDs Active Directory

| Event ID | Description |
|----------|-------------|
| **4662** | Operation effectuee sur un objet AD |
| **5136** | Attribut d'objet AD modifie |
| **5137** | Objet AD cree |
| **5138** | Objet AD restaure (undelete) |
| **5139** | Objet AD deplace |
| **5141** | Objet AD supprime |

### Analyser les modifications AD

```powershell
# Query AD object modifications (Event 5136)
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 5136
} -MaxEvents 20 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time          = $_.TimeCreated
            ChangedBy     = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
            ObjectDN      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ObjectDN' } | Select-Object -ExpandProperty '#text'
            AttributeName = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AttributeLDAPDisplayName' } | Select-Object -ExpandProperty '#text'
            OldValue      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AttributeValue' -and $_.InnerXml -match 'Old' } | Select-Object -ExpandProperty '#text'
            NewValue      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'AttributeValue' } | Select-Object -First 1 -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

---

## Scenarios de detection

### Detection de suppression de fichiers

```powershell
# Detect file deletions in a monitored folder
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4660
} -MaxEvents 50 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time    = $_.TimeCreated
            Account = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
            Object  = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ObjectName' } | Select-Object -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

### Detection de modification de permissions

```powershell
# Detect permission changes on objects (Event 4670)
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4670
} -MaxEvents 20 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            Account    = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
            ObjectName = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'ObjectName' } | Select-Object -ExpandProperty '#text'
            OldSD      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'OldSd' } | Select-Object -ExpandProperty '#text'
            NewSD      = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'NewSd' } | Select-Object -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

### Detection d'ajout a Domain Admins

```powershell
# Detect additions to the Domain Admins group specifically
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = @(4728, 4756)
} -MaxEvents 50 -ErrorAction SilentlyContinue |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        $groupName = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
        if ($groupName -match "Domain Admins|Enterprise Admins|Schema Admins") {
            [PSCustomObject]@{
                Time        = $_.TimeCreated
                GroupName   = $groupName
                MemberAdded = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'MemberName' } | Select-Object -ExpandProperty '#text'
                ChangedBy   = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
            }
        }
    } | Format-Table -AutoSize
```

---

## Audit du registre

### Configurer une SACL sur une cle de registre

```powershell
# Enable registry auditing
auditpol /set /subcategory:"Registry" /success:enable /failure:enable

# Add audit rule on a sensitive registry key
$regPath = "HKLM:\SYSTEM\CurrentControlSet\Services"
$acl = Get-Acl -Path $regPath -Audit

$auditRule = New-Object System.Security.AccessControl.RegistryAuditRule(
    "Everyone",
    "SetValue,Delete",
    "ContainerInherit",
    "None",
    "Success,Failure"
)

$acl.AddAuditRule($auditRule)
Set-Acl -Path $regPath -AclObject $acl
```

---

## Volume des evenements et optimisation

!!! warning "Attention au bruit"

    L'audit d'acces aux objets peut generer un **volume enorme** d'evenements. Recommandations :

    - N'auditez que les dossiers/objets **sensibles** (pas la racine C:\)
    - Ciblez les operations **critiques** (Delete, Write, ChangePermissions), pas Read
    - Utilisez des **groupes specifiques** plutot que Everyone quand possible
    - Dimensionnez le journal Security en consequence (4 Go minimum)
    - Centralisez les evenements dans un **SIEM** pour l'analyse

---

## Points cles a retenir

- Les **SACL** definissent quels acces sont journalises sur un objet specifique
- L'audit d'**acces aux fichiers** (4663) necessite l'activation de "File System" dans l'audit avance ET une SACL sur le dossier cible
- L'audit **Active Directory** (5136, 5137, 5141) trace les modifications d'attributs, creations et suppressions d'objets
- Le **volume d'evenements** doit etre maitrise en ciblant les objets et operations sensibles uniquement
- Les requetes PowerShell avec `Get-WinEvent` et parsing XML sont les outils essentiels d'analyse
- La centralisation via **SIEM** est incontournable pour l'exploitation operationnelle

---

## Pour aller plus loin

- Politique d'audit (voir la page [Politique d'audit](politique-audit.md))
- Journaux d'evenements et Event IDs cles (voir la page [Journaux d'evenements](journaux-evenements.md))
- Microsoft : Auditing file access
- Microsoft : Monitoring Active Directory for signs of compromise
- MITRE ATT&CK : Data Source - Windows Event Logs
