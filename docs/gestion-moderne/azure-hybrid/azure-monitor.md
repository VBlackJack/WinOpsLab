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
title: "Azure Monitor pour serveurs on-premises"
description: Surveiller des serveurs Windows on-premises avec Azure Monitor - agents, Log Analytics, metriques, alertes et tableaux de bord.
tags:
  - gestion-moderne
  - azure
  - supervision
  - intermediaire
---

# Azure Monitor pour serveurs on-premises

<span class="level-intermediate">Intermediaire</span> Â· Temps estime : 30 minutes

## Presentation

Azure Monitor est la plateforme unifiee de surveillance de Microsoft. Elle collecte, analyse et reagit aux donnees de telemetrie provenant de serveurs (Azure, on-premises, multi-cloud). Pour les serveurs Windows on-premises, Azure Monitor utilise des agents pour envoyer les metriques et les logs vers un **workspace Log Analytics**.

```mermaid
graph TD
    A[Serveurs Windows<br/>on-premises] -->|Azure Monitor Agent| B[Log Analytics<br/>Workspace]
    B --> C[Requetes KQL]
    B --> D[Alertes]
    B --> E[Workbooks / Tableaux de bord]
    B --> F[Azure Monitor Insights]
    D --> G[Email]
    D --> H[SMS]
    D --> I[Webhook / ITSM]
```

!!! example "Analogie"

    Azure Monitor, c'est comme les tableaux de bord d'une salle de controle de centrale electrique : des capteurs installes sur chaque machine (les agents) envoient en permanence leurs mesures (temperature, pression, debit) vers la salle de controle centrale. Les operateurs voient tout, peuvent definir des seuils d'alerte et n'ont pas besoin de se deplacer dans la centrale pour surveiller chaque machine.

## Agents de collecte

### Azure Monitor Agent (AMA) - Recommande

L'Azure Monitor Agent est l'agent unifie qui remplace les anciens agents (MMA/OMS et Dependency Agent).

| Caracteristique | Azure Monitor Agent |
|-----------------|-------------------|
| Deploiement | Via Azure Arc (on-premises) ou directement |
| Configuration | Data Collection Rules (DCR) centralisees |
| Multi-homing | Support natif de plusieurs workspaces |
| Performance | Optimise, faible impact |

### Agent Log Analytics (MMA) - Heritage

| Caracteristique | Log Analytics Agent (MMA) |
|-----------------|--------------------------|
| Statut | **En cours de deprecation** |
| Installation | Manuelle ou via GPO |
| Configuration | Sur l'agent ou via le portail |

!!! warning "Deprecation"

    L'agent Log Analytics (MMA/OMS) est en cours de deprecation.
    Migrez vers l'**Azure Monitor Agent** pour les nouveaux deploiements.

## Prerequis

| Element | Requis |
|---------|--------|
| Abonnement Azure | Actif |
| Log Analytics Workspace | Cree dans une region Azure |
| Azure Arc (pour AMA) | Serveurs inscrits dans Azure Arc |
| Connectivite | HTTPS (port 443) vers Azure |

## Creer un workspace Log Analytics

1. Portail Azure > **Log Analytics workspaces** > **Creer**
2. Configurer :
    - **Nom** : `law-winopslab`
    - **Region** : `West Europe`
    - **Resource Group** : `RG-Monitoring`
3. Configurer la retention (30 jours gratuits, extensible)

## Deployer l'Azure Monitor Agent

### Via Azure Arc (recommande)

Pour les serveurs deja inscrits dans Azure Arc :

1. Portail Azure > Serveur Arc > **Extensions**
2. Ajouter l'extension **Azure Monitor Windows Agent**
3. L'agent est installe et configure automatiquement

### Via le portail Azure

1. Log Analytics workspace > **Agents** > **Windows servers**
2. Telecharger l'agent Windows
3. Installer en specifiant le Workspace ID et la cle

```powershell
# Install Azure Monitor Agent via Azure CLI (for Arc-connected servers)
az connectedmachine extension create `
    --machine-name "SRV-DC01" `
    --resource-group "RG-Arc-Servers" `
    --name "AzureMonitorWindowsAgent" `
    --publisher "Microsoft.Azure.Monitor" `
    --type "AzureMonitorWindowsAgent" `
    --location "westeurope"
```

Resultat :

```text
{
  "id": "/subscriptions/.../resourceGroups/RG-Arc-Servers/providers/Microsoft.HybridCompute/machines/SRV-DC01/extensions/AzureMonitorWindowsAgent",
  "name": "AzureMonitorWindowsAgent",
  "properties": {
    "provisioningState": "Succeeded",
    "publisher": "Microsoft.Azure.Monitor",
    "type": "AzureMonitorWindowsAgent",
    "typeHandlerVersion": "1.22.0"
  },
  "resourceGroup": "RG-Arc-Servers"
}
```

## Data Collection Rules (DCR)

Les DCR definissent quelles donnees collecter et ou les envoyer.

### Creer une DCR

1. Portail Azure > **Monitor** > **Data Collection Rules** > **Creer**
2. Configurer :
    - **Nom** : `dcr-windows-servers`
    - **Region** : `West Europe`
    - **Platform** : Windows
3. **Ressources** : selectionner les serveurs Arc
4. **Collecter et livrer** : definir les sources de donnees

### Sources de donnees courantes

| Source | Description |
|--------|-------------|
| **Windows Event Logs** | Journaux Security, System, Application |
| **Performance Counters** | CPU, memoire, disque, reseau |
| **IIS Logs** | Journaux d'acces IIS |
| **Custom Logs** | Fichiers de log personnalises |
| **Syslog** | Messages Syslog (via agent Linux) |

### Exemple de DCR pour Windows Server

```json
{
    "dataSources": {
        "windowsEventLogs": [
            {
                "name": "SecurityEvents",
                "streams": ["Microsoft-SecurityEvent"],
                "xPathQueries": [
                    "Security!*[System[(EventID=4624 or EventID=4625 or EventID=4720)]]",
                    "System!*[System[(Level=1 or Level=2 or Level=3)]]",
                    "Application!*[System[(Level=1 or Level=2)]]"
                ]
            }
        ],
        "performanceCounters": [
            {
                "name": "PerfCounters",
                "streams": ["Microsoft-Perf"],
                "samplingFrequencyInSeconds": 60,
                "counterSpecifiers": [
                    "\\Processor(_Total)\\% Processor Time",
                    "\\Memory\\Available MBytes",
                    "\\LogicalDisk(*)\\% Free Space",
                    "\\PhysicalDisk(_Total)\\Avg. Disk Queue Length"
                ]
            }
        ]
    },
    "destinations": {
        "logAnalytics": [
            {
                "workspaceResourceId": "/subscriptions/.../law-winopslab",
                "name": "law-winopslab"
            }
        ]
    }
}
```

## Requetes KQL (Kusto Query Language)

KQL est le langage de requete utilise dans Log Analytics pour interroger les donnees collectees.

### Requetes essentielles

```
// CPU usage over the last hour
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where InstanceName == "_Total"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

```
// Failed logon attempts in the last 24 hours
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(24h)
| summarize FailedAttempts = count() by TargetAccount, Computer, IpAddress
| sort by FailedAttempts desc
```

```
// Disk space alerts (less than 15% free)
Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
| where InstanceName != "_Total"
| where TimeGenerated > ago(1h)
| summarize AvgFreeSpace = avg(CounterValue) by Computer, InstanceName
| where AvgFreeSpace < 15
| sort by AvgFreeSpace asc
```

```
// Recent critical and error events
Event
| where EventLevelName in ("Error", "Critical")
| where TimeGenerated > ago(24h)
| summarize Count = count() by Computer, Source, EventID, EventLevelName
| sort by Count desc
| take 20
```

## Alertes

### Creer une alerte

1. Portail Azure > **Monitor** > **Alertes** > **Creer une regle d'alerte**
2. **Scope** : selectionner le workspace Log Analytics
3. **Condition** : definir la requete KQL et le seuil
4. **Actions** : configurer le groupe d'actions (email, SMS, webhook)
5. **Details** : nommer la regle et definir la severite

### Exemples d'alertes recommandees

| Alerte | Requete KQL simplifiee | Seuil |
|--------|----------------------|-------|
| CPU eleve | `Perf \| where CounterName == "% Processor Time"` | > 90% pendant 15 min |
| Memoire faible | `Perf \| where CounterName == "Available MBytes"` | < 500 Mo pendant 10 min |
| Espace disque critique | `Perf \| where CounterName == "% Free Space"` | < 10% |
| Echecs de connexion | `SecurityEvent \| where EventID == 4625` | > 10 en 5 min |
| Service arrete | `Event \| where EventID == 7036` | Service critique arrete |

### Groupes d'actions

| Type d'action | Description |
|---------------|-------------|
| **Email** | Notification par email |
| **SMS** | Notification par SMS |
| **Webhook** | Appel HTTP vers un service externe |
| **ITSM** | Integration avec ServiceNow, BMC, etc. |
| **Azure Function** | Declenchement d'une fonction Azure |
| **Logic App** | Declenchement d'un workflow Logic App |

## Workbooks et tableaux de bord

### Workbooks

Les Workbooks sont des rapports interactifs qui combinent des requetes KQL, des graphiques et du texte.

1. **Monitor** > **Workbooks** > **Nouveau**
2. Ajouter des elements : requete KQL, graphique, texte, parametre
3. Les workbooks predefinies couvrent les scenarios courants :
    - **VM Insights** : performances et dependances
    - **Windows Events** : analyse des journaux
    - **Network Insights** : metriques reseau

### Tableaux de bord Azure

Epingler des requetes et graphiques sur un tableau de bord Azure partage :

1. Executer une requete dans Log Analytics
2. Cliquer sur **Epingler au tableau de bord**
3. Choisir un tableau de bord existant ou en creer un nouveau

!!! example "Scenario pratique"

    **Context :** Pierre, ingenieur systeme, doit configurer Azure Monitor sur les 4 serveurs de son infrastructure (SRV-DC01, SRV-01, SRV-WEB01, SRV-FS01) et recevoir des alertes par email si le CPU depasse 85% pendant plus de 10 minutes ou si l'espace disque passe sous 15%.

    **Etape 1 : Creer le workspace Log Analytics**

    Pierre cree `law-winopslab` dans le Resource Group `RG-Monitoring`, region West Europe, retention 30 jours.

    **Etape 2 : Deployer l'agent AMA via Azure Arc**

    Les serveurs sont deja inscrits dans Arc. Pierre deploie l'extension Azure Monitor Windows Agent depuis le portail sur les 4 machines simultanement.

    **Etape 3 : Creer une Data Collection Rule**

    Il cree la DCR `dcr-windows-servers` qui collecte :
    - Windows Event Logs : Security (4624, 4625), System (Critical, Error, Warning)
    - Performance Counters : CPU, memoire disponible, espace disque (toutes les 60 secondes)

    Il assigne la DCR aux 4 serveurs Arc.

    **Etape 4 : Creer les alertes**

    Dans Azure Monitor > Alertes, Pierre cree deux regles :

    - Alerte CPU : requete KQL sur `Perf` ou CounterName = "% Processor Time" > 85, evaluee toutes les 5 minutes sur une fenetre de 10 minutes.
    - Alerte disque : requete KQL sur `Perf` ou CounterName = "% Free Space" < 15, evaluee toutes les 15 minutes.

    Les deux alertes sont liees a un Action Group qui envoie un email a `equipe-it@lab.local`.

    **Etape 5 : Valider avec une requete KQL**

    ```
    Perf
    | where CounterName == "% Processor Time" and InstanceName == "_Total"
    | where TimeGenerated > ago(1h)
    | summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
    | sort by TimeGenerated desc
    ```

    Les donnees de SRV-DC01, SRV-01, SRV-WEB01 et SRV-FS01 apparaissent dans les resultats. La surveillance est operationnelle.

!!! danger "Erreurs courantes"

    **Utiliser l'agent MMA (Log Analytics) pour de nouveaux deploiements.** L'agent MMA est en cours de deprecation. Tout nouveau deploiement doit utiliser l'Azure Monitor Agent (AMA) avec des Data Collection Rules. Migrer les agents MMA existants vers AMA.

    **Oublier de creer une Data Collection Rule apres le deploiement de l'agent.** L'agent AMA seul ne collecte rien. Sans DCR associee, aucune donnee n'arrive dans Log Analytics. Toujours verifier qu'une DCR est bien assignee aux serveurs cibles apres le deploiement.

    **Sous-estimer les couts de retention des donnees.** Log Analytics facture le stockage au-dela de 30 jours. Sur un parc de 20 serveurs avec collecte verbose (tous les logs de securite + compteurs toutes les 30 secondes), les couts peuvent depasser les attentes. Calibrer soigneusement les DCR et les intervalles de collecte.

    **Ne pas configurer de groupe d'actions sur les alertes.** Une alerte sans Action Group n'envoie aucune notification. La regle se declenche silencieusement dans le portail sans que personne soit prevenu. Toujours associer un groupe d'actions avec au moins un canal de notification.

    **Utiliser KQL sans tester les performances sur de grandes plages temporelles.** Une requete KQL sans filtre sur `TimeGenerated` peut scanner plusieurs mois de donnees et prendre plusieurs minutes, voire echouer par timeout. Toujours inclure un filtre temporel (`ago(1h)`, `ago(24h)`) et tester sur une petite plage avant de sauvegarder une alerte.

## Points cles a retenir

- Azure Monitor centralise la surveillance des serveurs on-premises dans le cloud Azure
- L'**Azure Monitor Agent** (AMA) via Azure Arc est l'approche recommandee pour les nouveaux deploiements
- Les **Data Collection Rules** (DCR) definissent precisement quelles donnees collecter
- **KQL** est le langage de requete pour interroger les donnees dans Log Analytics
- Les **alertes** declenchent des notifications automatiques lors de depassements de seuils
- Les **Workbooks** fournissent des tableaux de bord interactifs pour la visualisation
- La retention par defaut est de 30 jours (gratuit), extensible moyennant un cout supplementaire

## Pour aller plus loin

- [Azure Arc](azure-arc.md) pour inscrire les serveurs on-premises dans Azure
- [Azure Backup](azure-backup.md) pour la sauvegarde cloud
- [Analyseur de performances](../../supervision/surveillance/performance-monitor.md) pour la surveillance locale

