---
title: "Syslog sur Windows"
description: Integrer Windows Server 2022 avec des serveurs Syslog - transfert d'evenements, outils tiers et integration avec les SIEM.
tags:
  - supervision
  - logs
  - intermediaire
---

# Syslog sur Windows

<span class="level-intermediate">Intermediaire</span> · Temps estime : 20 minutes

## Presentation

Le protocole **Syslog** (RFC 5424) est le standard de journalisation dans les environnements Linux et les equipements reseau. Windows Server n'integre pas nativement de client Syslog, mais plusieurs approches permettent de rediriger les evenements Windows vers un serveur Syslog ou un SIEM.

```mermaid
graph LR
    A[Windows Server<br/>Event Logs] --> B{Methode de transfert}
    B --> C[Agent tiers<br/>NXLog, Winlogbeat]
    B --> D[WEF + convertisseur]
    B --> E[Script PowerShell]
    C --> F[Serveur Syslog / SIEM]
    D --> F
    E --> F
```

## Pourquoi integrer Syslog ?

| Besoin | Explication |
|--------|-------------|
| **Centralisation heterogene** | Agreger les logs Windows, Linux, reseau dans un meme outil |
| **SIEM** | Alimenter un SIEM (Splunk, Elastic SIEM, Graylog, Wazuh) |
| **Conformite** | Repondre aux exigences de conservation et d'audit (PCI-DSS, ISO 27001) |
| **Correlation** | Correler les evenements Windows avec d'autres sources |
| **Retention longue** | Stocker les logs au-dela de la capacite native des Event Logs |

## Comparaison des approches

| Approche | Complexite | Performance | Fiabilite | Cout |
|----------|-----------|-------------|-----------|------|
| **NXLog Community** | Moyenne | Elevee | Elevee | Gratuit |
| **Winlogbeat (Elastic)** | Moyenne | Elevee | Elevee | Gratuit |
| **Syslog via script PowerShell** | Faible | Faible | Moyenne | Gratuit |
| **WEF + convertisseur Syslog** | Elevee | Elevee | Elevee | Variable |
| **Agent SIEM proprietaire** | Faible | Elevee | Elevee | Licence |

## NXLog Community Edition

NXLog est un agent de collecte de logs multiplateforme, tres utilise pour transferer les Event Logs Windows vers des serveurs Syslog.

### Installation

1. Telecharger NXLog Community Edition depuis le site officiel
2. Installer avec les options par defaut
3. Le service `nxlog` est cree automatiquement

### Configuration de base

Le fichier de configuration se trouve dans `C:\Program Files\nxlog\conf\nxlog.conf` :

```xml
## NXLog configuration - Forward Windows events to Syslog

define ROOT C:\Program Files\nxlog
define CERTDIR %ROOT%\cert
define CONFDIR %ROOT%\conf
define LOGDIR %ROOT%\data
define LOGFILE %LOGDIR%\nxlog.log

Moduledir %ROOT%\modules
CacheDir  %ROOT%\data
Pidfile   %ROOT%\data\nxlog.pid
SpoolDir  %ROOT%\data

<Extension _syslog>
    Module  xm_syslog
</Extension>

# Input: Windows Event Log
<Input eventlog>
    Module  im_msvistalog
    Query   <QueryList>\
                <Query Id="0">\
                    <Select Path="Security">*</Select>\
                    <Select Path="System">*[System[(Level=1 or Level=2 or Level=3)]]</Select>\
                    <Select Path="Application">*[System[(Level=1 or Level=2 or Level=3)]]</Select>\
                </Query>\
            </QueryList>
</Input>

# Output: Syslog server via UDP
<Output syslog_udp>
    Module  om_udp
    Host    192.168.10.50
    Port    514
    Exec    to_syslog_bsd();
</Output>

# Output: Syslog server via TCP (more reliable)
<Output syslog_tcp>
    Module  om_tcp
    Host    192.168.10.50
    Port    1514
    Exec    to_syslog_bsd();
</Output>

# Route: connect input to output
<Route eventlog_to_syslog>
    Path    eventlog => syslog_tcp
</Route>
```

### Gestion du service

```powershell
# Start NXLog service
Start-Service nxlog

# Check service status
Get-Service nxlog

# Restart after configuration change
Restart-Service nxlog

# Check NXLog logs for errors
Get-Content "C:\Program Files\nxlog\data\nxlog.log" -Tail 20
```

## Winlogbeat (Elastic Stack)

Winlogbeat est l'agent officiel d'Elastic pour collecter les Event Logs Windows. Il envoie directement vers Elasticsearch, Logstash ou un serveur Syslog.

### Configuration de base

Fichier `winlogbeat.yml` :

```yaml
# Winlogbeat configuration
winlogbeat.event_logs:
  - name: Security
    event_id: 4624, 4625, 4720, 4726, 4732, 4672
  - name: System
    level: critical, error, warning
  - name: Application
    level: critical, error

# Output to Logstash
output.logstash:
  hosts: ["192.168.10.50:5044"]

# Alternative: output to Elasticsearch directly
# output.elasticsearch:
#   hosts: ["192.168.10.50:9200"]
```

### Gestion du service

```powershell
# Install Winlogbeat as a service
.\winlogbeat.exe install

# Start the service
Start-Service winlogbeat

# Test the configuration
.\winlogbeat.exe test config -e
```

## Envoi Syslog via PowerShell

Pour les environnements simples ou les tests, un script PowerShell peut envoyer des evenements au format Syslog.

```powershell
# Simple Syslog sender function (UDP)
function Send-SyslogMessage {
    param(
        [string]$Server,
        [int]$Port = 514,
        [int]$Facility = 1,
        [int]$Severity = 6,
        [string]$Message
    )

    $priority = ($Facility * 8) + $Severity
    $timestamp = Get-Date -Format "MMM dd HH:mm:ss"
    $hostname = $env:COMPUTERNAME
    $syslogMessage = "<$priority>$timestamp $hostname $Message"

    $udpClient = New-Object System.Net.Sockets.UdpClient
    $bytes = [System.Text.Encoding]::ASCII.GetBytes($syslogMessage)
    $udpClient.Send($bytes, $bytes.Length, $Server, $Port) | Out-Null
    $udpClient.Close()
}

# Forward recent security events to Syslog
$events = Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = 4625
    StartTime = (Get-Date).AddMinutes(-5)
} -ErrorAction SilentlyContinue

foreach ($event in $events) {
    $message = "EventID=$($event.Id) Source=$($event.ProviderName) $($event.Message -replace '\r?\n',' ')"
    Send-SyslogMessage -Server "192.168.10.50" -Port 514 -Severity 4 -Message $message
}
```

!!! warning "Limitations du script"

    Cette approche par script est adaptee aux tests ou au transfert d'evenements specifiques.
    Pour la production, privilegiez un agent comme NXLog ou Winlogbeat qui gerent la mise en
    tampon, les reconnexions et la garantie de livraison.

## Architecture avec collecteur intermediaire

Pour les grands environnements, combiner WEF/WEC avec un convertisseur Syslog :

```mermaid
graph LR
    A[Serveurs Windows] -->|WEF| B[Collecteur WEC]
    B --> C[NXLog sur WEC]
    C -->|Syslog TCP| D[SIEM / Serveur Syslog]
    E[Equipements reseau] -->|Syslog| D
    F[Serveurs Linux] -->|Syslog| D
```

Cette architecture offre :

- Centralisation Windows native (WEF) avant la conversion
- Un seul point de conversion Syslog (le collecteur WEC)
- Reduction de la charge reseau (filtrage en amont par WEF)

## Protocoles de transport

| Protocole | Port | Fiabilite | Usage |
|-----------|------|-----------|-------|
| **UDP** | 514 | Faible (pas d'acquittement) | Tests, reseaux locaux fiables |
| **TCP** | 1514 / 6514 | Moyenne (acquittement TCP) | Production |
| **TLS/TCP** | 6514 | Elevee (chiffre + acquitte) | Production securisee |

!!! tip "Recommandation"

    Utilisez **TCP** ou **TLS/TCP** en production pour garantir la livraison des evenements.
    Le UDP peut perdre des messages en cas de congestion reseau.

## Points cles a retenir

- Windows Server n'integre pas de client Syslog natif ; un agent tiers est necessaire
- **NXLog** et **Winlogbeat** sont les agents les plus utilises, gratuits et fiables
- Pour les grands environnements, combiner WEF (centralisation Windows) + agent Syslog sur le collecteur
- Utiliser TCP ou TLS pour le transport Syslog en production (eviter UDP)
- L'integration Syslog est essentielle pour les SIEM et la correlation multi-sources
- Un script PowerShell suffit pour les tests mais ne convient pas a la production

## Pour aller plus loin

- [Windows Event Forwarding (WEF/WEC)](wef-wec.md) pour la centralisation native Windows
- [Observateur d'evenements](../surveillance/event-viewer.md) pour configurer les filtres d'evenements
- [Politique d'audit](../../securite/audit/politique-audit.md) pour generer les evenements de securite
