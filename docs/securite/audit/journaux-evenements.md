---
title: "Journaux d'evenements"
description: "Analyse des journaux Windows Server 2022 : Security, System, Application, Event IDs critiques et requetes PowerShell pour la detection d'incidents."
tags:
  - securite
  - audit
  - journaux
  - event-log
  - detection
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

# Journaux d'evenements

<span class="level-advanced">Avance</span> Â· Temps estime : 45 minutes

Les journaux d'evenements Windows sont la source principale d'information pour la detection d'incidents, l'investigation forensique et la conformite. Maitriser les Event IDs critiques et les requetes PowerShell associees est indispensable pour tout administrateur securite.

---

!!! example "Analogie"

    Les journaux d'evenements sont comme les boites noires d'un avion : ils enregistrent tout ce qui se passe sur le serveur. En cas d'incident, ce sont les premieres pieces a analyser pour comprendre ce qui s'est passe, quand, et par qui. Sans ces enregistrements, une investigation forensique est impossible.

## Les journaux principaux

| Journal | Chemin | Contenu |
|---------|--------|---------|
| **Security** | `Security` | Authentifications, acces aux objets, modifications de privileges |
| **System** | `System` | Evenements systeme, services, pilotes |
| **Application** | `Application` | Evenements des applications |
| **PowerShell** | `Microsoft-Windows-PowerShell/Operational` | Execution de scripts et commandes |
| **Sysmon** | `Microsoft-Windows-Sysmon/Operational` | Supervision avancee (si installe) |
| **TaskScheduler** | `Microsoft-Windows-TaskScheduler/Operational` | Taches planifiees |

```powershell
# List all available event logs
Get-WinEvent -ListLog * | Where-Object { $_.RecordCount -gt 0 } |
    Select-Object LogName, RecordCount, MaximumSizeInBytes |
    Sort-Object RecordCount -Descending |
    Format-Table -AutoSize
```

Resultat :

```text
LogName                                       RecordCount  MaximumSizeInBytes
-------                                       -----------  ------------------
Security                                        125847         4294967296
Microsoft-Windows-PowerShell/Operational         34219         1073741824
System                                           18432         1073741824
Application                                      12105         1073741824
Microsoft-Windows-TaskScheduler/Operational        4521          104857600
Microsoft-Windows-Sysmon/Operational               3892          104857600
```

---

## Event IDs critiques - Security Log

### Authentification et sessions

| Event ID | Description | Importance |
|----------|-------------|------------|
| **4624** | Ouverture de session reussie | Tracer les acces |
| **4625** | Echec d'ouverture de session | Detection de brute force |
| **4634** | Fermeture de session | Correlation avec 4624 |
| **4648** | Connexion avec identifiants explicites (RunAs) | Surveillance des elevations |
| **4672** | Privileges speciaux assignes a la session | Detection d'acces admin |
| **4776** | Validation de credentials (NTLM) | Detection Pass-the-Hash |

### Types de logon (Event 4624)

| Logon Type | Description | Contexte |
|------------|-------------|----------|
| **2** | Interactive | Connexion physique ou console |
| **3** | Network | Acces partage, impression |
| **4** | Batch | Tache planifiee |
| **5** | Service | Demarrage d'un service |
| **7** | Unlock | Deverrouillage de session |
| **8** | NetworkCleartext | Mot de passe en clair (IIS Basic Auth) |
| **10** | RemoteInteractive | Connexion RDP |
| **11** | CachedInteractive | Credentials mises en cache |

```powershell
# Query successful logons (Event 4624)
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4624
} -MaxEvents 20 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time       = $_.TimeCreated
            Account    = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
            Domain     = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetDomainName' } | Select-Object -ExpandProperty '#text'
            LogonType  = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'LogonType' } | Select-Object -ExpandProperty '#text'
            SourceIP   = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select-Object -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

Resultat :

```text
Time                  Account        Domain     LogonType  SourceIP
----                  -------        ------     ---------  --------
2025-02-20 09:32:14   T1-jdupont     LAB        10         10.0.0.50
2025-02-20 09:30:01   SRV-01$        LAB        3          10.0.0.10
2025-02-20 09:28:45   svc-backup     LAB        5          -
2025-02-20 09:15:22   T1-mlambert    LAB        10         10.0.0.51
2025-02-20 08:00:00   SYSTEM         NT AUT...  5          -
```

```powershell
# Query failed logons (Event 4625) - Brute force detection
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4625
} -MaxEvents 50 |
    ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time     = $_.TimeCreated
            Account  = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
            SourceIP = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select-Object -ExpandProperty '#text'
            Reason   = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubStatus' } | Select-Object -ExpandProperty '#text'
        }
    } | Format-Table -AutoSize
```

Resultat :

```text
Time                  Account        SourceIP       Reason
----                  -------        --------       ------
2025-02-20 09:35:12   admin          10.0.0.99      0xC000006A
2025-02-20 09:35:10   admin          10.0.0.99      0xC000006A
2025-02-20 09:35:08   administrator  10.0.0.99      0xC000006A
2025-02-20 09:35:05   Administrator  10.0.0.99      0xC000006A
2025-02-20 09:35:02   root           10.0.0.99      0xC0000064
2025-02-20 09:34:58   sa             10.0.0.99      0xC0000064
```

### Sous-statuts d'echec de connexion (4625)

| SubStatus | Signification |
|-----------|---------------|
| `0xC0000064` | Compte inexistant |
| `0xC000006A` | Mot de passe incorrect |
| `0xC0000234` | Compte verrouille |
| `0xC0000072` | Compte desactive |
| `0xC000006F` | Connexion hors des heures autorisees |
| `0xC0000070` | Restriction de station de travail |
| `0xC0000193` | Compte expire |

---

### Gestion des comptes

| Event ID | Description | Importance |
|----------|-------------|------------|
| **4720** | Compte utilisateur cree | Surveillance des creations |
| **4722** | Compte utilisateur active | Detection de comptes reactives |
| **4723** | Tentative de changement de mot de passe | Suivi des changements |
| **4724** | Reinitialisation de mot de passe par un admin | Verifier la legitimite |
| **4725** | Compte utilisateur desactive | Suivi du deprovisioning |
| **4726** | Compte utilisateur supprime | Tracer les suppressions |
| **4728** | Membre ajoute a un groupe global de securite | Surveillance des privileges |
| **4732** | Membre ajoute a un groupe local de securite | Surveillance des privileges |
| **4756** | Membre ajoute a un groupe universel de securite | Surveillance des privileges |

```powershell
# Monitor privileged group changes (Domain Admins, Enterprise Admins, etc.)
$privilegedGroupEvents = Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = @(4728, 4732, 4756)
} -MaxEvents 50 -ErrorAction SilentlyContinue

$privilegedGroupEvents | ForEach-Object {
    $xml = [xml]$_.ToXml()
    [PSCustomObject]@{
        Time      = $_.TimeCreated
        EventId   = $_.Id
        GroupName = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
        MemberAdded = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'MemberName' } | Select-Object -ExpandProperty '#text'
        ChangedBy = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
    }
} | Format-Table -AutoSize
```

Resultat :

```text
Time                  EventId  GroupName         MemberAdded                          ChangedBy
----                  -------  ---------         -----------                          ---------
2025-02-19 16:42:30   4728     Domain Admins     CN=T0-rmartin,OU=Tier0,...           T0-jdupont
2025-02-18 11:15:00   4732     Administrators    CN=svc-monitoring,OU=Services,...    T1-mlambert
```

### Autres evenements critiques

| Event ID | Journal | Description |
|----------|---------|-------------|
| **1102** | Security | Journal de securite efface (alerte critique) |
| **4688** | Security | Nouveau processus cree (avec command line logging) |
| **4697** | Security | Service installe sur le systeme |
| **4698** | Security | Tache planifiee creee |
| **4719** | Security | Politique d'audit modifiee |
| **7045** | System | Service installe |
| **7040** | System | Type de demarrage d'un service modifie |

!!! danger "Event 1102 - Journal efface"

    L'effacement du journal de securite (Event 1102) est un indicateur critique de compromission. Un attaquant efface frequemment les journaux pour couvrir ses traces. Cet evenement doit declencher une **alerte immediate**.

```powershell
# Detect Security log clearing (Event 1102)
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 1102
} -ErrorAction SilentlyContinue |
    Select-Object TimeCreated, Message
```

Resultat :

```text
TimeCreated            Message
-----------            -------
2025-02-15 03:22:47    The audit log was cleared.
                        Subject:
                          Security ID: LAB\unknown-user
                          Account Name: unknown-user
                          Logon ID: 0x3E7
```

---

## Event IDs cles - System Log

| Event ID | Description |
|----------|-------------|
| **6005** | Demarrage du service Event Log (boot) |
| **6006** | Arret du service Event Log (shutdown) |
| **6008** | Arret inattendu (crash/blue screen) |
| **7045** | Nouveau service installe |
| **1074** | Arret/redemarrage initie par un processus |

```powershell
# Find unexpected shutdowns
Get-WinEvent -FilterHashtable @{
    LogName = 'System'
    Id = @(6008, 1074)
} -MaxEvents 10 |
    Select-Object TimeCreated, Id, Message |
    Format-Table -Wrap
```

Resultat :

```text
TimeCreated            Id    Message
-----------            --    -------
2025-02-18 02:15:33    6008  The previous system shutdown at 02:14:12 on
                              18/02/2025 was unexpected.
2025-02-10 22:00:05    1074  The process C:\Windows\system32\shutdown.exe (SRV-01)
                              has initiated the restart of computer SRV-01 on behalf
                              of user LAB\T1-jdupont for the following reason:
                              Operating System: Service pack (Planned)
```

---

## Requetes avancees avec XPath

Pour des recherches precises, utilisez des filtres XPath :

```powershell
# Find RDP logons (LogonType 10) from a specific IP range
$xpathQuery = @"
*[System[EventID=4624]]
and
*[EventData[Data[@Name='LogonType']='10']]
and
*[EventData[Data[@Name='IpAddress']='192.168.1.']]
"@

# Using FilterXPath (simplified)
Get-WinEvent -LogName Security -FilterXPath `
    "*[System[EventID=4624] and EventData[Data[@Name='LogonType']='10']]" `
    -MaxEvents 10

# Find failed logons for a specific account
Get-WinEvent -LogName Security -FilterXPath `
    "*[System[EventID=4625] and EventData[Data[@Name='TargetUserName']='Administrator']]" `
    -MaxEvents 20
```

---

## Centralisation avec Windows Event Forwarding

Pour les environnements multi-serveurs, configurez le **Windows Event Forwarding (WEF)** pour centraliser les journaux :

```mermaid
flowchart LR
    SRV1[Serveur 1] -->|WinRM| COLLECTOR[Serveur collecteur WEF]
    SRV2[Serveur 2] -->|WinRM| COLLECTOR
    DC01[DC01] -->|WinRM| COLLECTOR
    COLLECTOR --> SIEM[SIEM / Analyseur]
```

```powershell
# On the collector server: configure the Windows Event Collector service
wecutil qc /q

# Create a subscription for critical security events
wecutil cs "C:\WEF\security-subscription.xml"

# On source servers: configure WinRM
winrm quickconfig -q

# Add the collector to the Event Log Readers local group on source servers
Add-LocalGroupMember -Group "Event Log Readers" -Member "LAB\WEF-Collector$"
```

Resultat :

```text
Windows Event Collector service is already running.
WinRM service is already running on this machine.
WinRM is already set up for remote management on this computer.
```

---

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Karim, analyste securite, recoit une alerte a 3h du matin : le compte `svc-backup` est utilise pour des connexions RDP inhabituelles sur le serveur `SRV-01` (10.0.0.11) depuis une adresse IP inconnue. Il doit investiguer rapidement.

    **Investigation** :

    ```powershell
    # Step 1: Find all RDP logons (LogonType 10) for svc-backup
    Get-WinEvent -ComputerName "SRV-01" -FilterXPath `
        "*[System[EventID=4624] and EventData[Data[@Name='TargetUserName']='svc-backup'] and EventData[Data[@Name='LogonType']='10']]" `
        -MaxEvents 20 |
        ForEach-Object {
            $xml = [xml]$_.ToXml()
            [PSCustomObject]@{
                Time     = $_.TimeCreated
                SourceIP = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select-Object -ExpandProperty '#text'
            }
        }
    ```

    Resultat :

    ```text
    Time                   SourceIP
    ----                   --------
    2025-02-20 03:14:22    10.0.0.99
    2025-02-20 03:02:15    10.0.0.99
    2025-02-20 02:55:08    10.0.0.99
    ```

    **Analyse** : l'adresse `10.0.0.99` n'appartient pas au reseau d'administration. Le compte de service `svc-backup` ne devrait jamais se connecter en RDP.

    ```powershell
    # Step 2: Check if any privileged group changes occurred
    Get-WinEvent -ComputerName "DC-01" -FilterHashtable @{
        LogName = 'Security'; Id = @(4728, 4732, 4756)
        StartTime = (Get-Date).AddHours(-6)
    } | ForEach-Object {
        $xml = [xml]$_.ToXml()
        [PSCustomObject]@{
            Time      = $_.TimeCreated
            Group     = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
            ChangedBy = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'SubjectUserName' } | Select-Object -ExpandProperty '#text'
        }
    }
    ```

    Resultat :

    ```text
    Time                   Group              ChangedBy
    ----                   -----              ---------
    2025-02-20 03:16:45    Administrators     svc-backup
    ```

    **Conclusion** : le compte `svc-backup` a ete compromis et utilise pour s'ajouter au groupe Administrators local. Karim desactive immediatement le compte, isole le serveur et lance une investigation forensique complete.

---

!!! danger "Erreurs courantes"

    1. **Ignorer les echecs de connexion repetitifs (Event 4625)** : des centaines de 4625 en quelques minutes depuis la meme IP signalent une attaque par brute force. Configurez des alertes automatiques sur ce pattern.

    2. **Ne pas surveiller l'Event 1102 (journal efface)** : un journal de securite efface est un indicateur fort de compromission. Centralisez les journaux via WEF ou SIEM pour en conserver une copie meme si l'attaquant efface le journal local.

    3. **Confondre les Logon Types** : un LogonType 10 (RDP) sur un compte de service est anormal. Un LogonType 3 (reseau) depuis une IP externe est suspect. Apprenez a interpreter les types de connexion en contexte.

    4. **Analyser uniquement le journal Security** : les journaux System (Event 7045 - nouveau service), PowerShell (ScriptBlock Logging) et Sysmon fournissent des informations complementaires essentielles pour reconstituer une attaque.

---

## Points cles a retenir

- Les **Event IDs 4624/4625** (connexions) sont la base de toute surveillance de securite
- Les **Event IDs 4720-4726** (gestion des comptes) permettent de tracer le cycle de vie des comptes
- L'**Event 1102** (journal efface) est un indicateur critique de compromission
- Les **Logon Types** donnent le contexte essentiel : RDP (10), reseau (3), interactif (2)
- Les requetes **PowerShell avec XPath** permettent des recherches precises et performantes
- La **centralisation** via WEF ou SIEM est indispensable en environnement multi-serveurs

---

## Pour aller plus loin

- Audit avance des objets AD et fichiers (voir la page [Audit avance](audit-avance.md))
- Politique d'audit (voir la page [Politique d'audit](politique-audit.md))
- Microsoft : Windows Security Event Log reference
- MITRE ATT&CK : correlations Event ID / techniques d'attaque

