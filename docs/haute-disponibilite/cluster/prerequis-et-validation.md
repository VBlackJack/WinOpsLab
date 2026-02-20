---
title: "Prerequis et validation du cluster"
description: "Preparer l'infrastructure pour un cluster de basculement Windows Server 2022 : stockage partage, reseau, comptes AD et tests de validation."
tags:
  - haute-disponibilite
  - cluster
  - validation
  - windows-server
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

# Prerequis et validation du cluster

<span class="level-advanced">Avance</span> Â· Temps estime : 50 minutes

## Introduction

Avant de creer un cluster de basculement, il est indispensable de valider que l'infrastructure repond a un ensemble strict de prerequis. Windows Server integre un assistant de **validation de cluster** qui verifie automatiquement la conformite de la configuration.

!!! example "Analogie"

    Avant de construire une maison, un architecte verifie que le terrain est stable, que les materiaux sont aux normes et que les fondations sont solides. La validation du cluster est exactement la meme demarche : on inspecte methodiquement chaque composant (reseau, stockage, OS) avant de batir l'infrastructure haute disponibilite. Construire sans valider, c'est batir sur du sable.

## Prerequis materiels et logiciels

### Systeme d'exploitation

- Tous les noeuds doivent executer la **meme edition** de Windows Server 2022 (Standard ou Datacenter)
- Tous les noeuds doivent etre au **meme niveau de mise a jour** (meme CU)
- La fonctionnalite **Failover Clustering** doit etre installee sur chaque noeud

```powershell
# Install Failover Clustering feature on all future nodes
Install-WindowsFeature -Name Failover-Clustering -IncludeManagementTools
```

Resultat :

```text
Success Restart Needed Exit Code      Feature Result
------- -------------- ---------      --------------
True    No             Success        {Failover Clustering, Failover Cluster
                                       Management Tools, Failover Cluster
                                       Module for Windows PowerShell}
```

### Reseau

Un cluster de basculement necessite au minimum **deux reseaux** :

| Reseau | Role | Configuration |
|---|---|---|
| **Production** | Communication clients | IP statiques, meme sous-reseau ou routage correct |
| **Cluster (heartbeat)** | Communication intra-cluster | Reseau dedie, isole du trafic client |

!!! tip "Bonnes pratiques reseau"

    - Utiliser des adresses IP **statiques** sur tous les noeuds (pas de DHCP)
    - Configurer le NIC teaming **avant** la creation du cluster
    - Verifier que les pare-feu autorisent le trafic cluster (ports TCP 135, 3343 et plages dynamiques RPC)
    - Le DNS doit fonctionner correctement entre tous les noeuds

```powershell
# Verify network connectivity between nodes
Test-Connection -ComputerName "NODE2" -Count 4

# Verify DNS resolution for all nodes
Resolve-DnsName -Name "NODE1.yourdomain.local"
Resolve-DnsName -Name "NODE2.yourdomain.local"
```

Resultat :

```text
Source        Destination   IPV4Address   Bytes    Time(ms)
------        -----------   -----------   -----    --------
SRV-01        SRV-02        10.0.0.12     32       1
SRV-01        SRV-02        10.0.0.12     32       1
SRV-01        SRV-02        10.0.0.12     32       1
SRV-01        SRV-02        10.0.0.12     32       1

Name                          Type   TTL   Section    IPAddress
----                          ----   ---   -------    ---------
SRV-01.lab.local              A      3600  Answer     10.0.0.11
SRV-02.lab.local              A      3600  Answer     10.0.0.12
```

### Stockage partage

Le stockage partage doit etre accessible par **tous les noeuds** du cluster :

- **SAN iSCSI** : configurez l'initiateur iSCSI sur chaque noeud et connectez les LUNs
- **SAN Fibre Channel** : verifiez le zoning et les LUN masking
- **Storage Spaces Direct** : au minimum 2 noeuds avec disques locaux compatibles

```powershell
# Verify iSCSI connection on each node
Get-IscsiTarget
Get-IscsiConnection

# List available disks (should show shared disks)
Get-Disk | Where-Object { $_.BusType -eq 'iSCSI' }
```

Resultat :

```text
NodeAddress                              IsConnected
-----------                              -----------
iqn.2025-01.local.lab:storage-cluster    True

ConnectionIdentifier : ffffac10-0001-0000-0100-0100ac100014
InitiatorAddress     : 10.0.0.11
TargetAddress        : 10.0.0.50
InitiatorPortNumber  : 0

Number FriendlyName    SerialNumber   HealthStatus OperationalStatus TotalSize BusType
------ ------------    ------------   ------------ ----------------- --------- -------
2      ISCSI Disk 01   SN-00201       Healthy      Online            50 GB     iSCSI
3      ISCSI Disk 02   SN-00202       Healthy      Online            100 GB    iSCSI
```

!!! warning "Disques partages"

    Les disques partages doivent etre **initialises** mais **non formates** avant la creation du cluster. Le cluster gerera le formatage et l'attribution.

### Active Directory

Le cluster necessite des comptes et permissions specifiques dans Active Directory :

1. **Compte de creation** : l'administrateur qui cree le cluster doit avoir les droits de creer des objets ordinateur dans l'OU cible
2. **CNO (Cluster Name Object)** : un compte ordinateur sera cree automatiquement pour le nom du cluster
3. **VCO (Virtual Computer Object)** : chaque role clustered creera un objet ordinateur supplementaire

```powershell
# Verify that the cluster creation account has appropriate AD permissions
# Pre-stage the CNO (optional but recommended)
New-ADComputer -Name "YOURCLUSTER" -Path "OU=Clusters,DC=yourdomain,DC=local" -Enabled $false

# Grant the CNO full control (needed for VCO creation)
# This is typically done via ADUC GUI > Security tab
```

!!! tip "Pre-staging du CNO"

    Le pre-staging (creation prealable) du compte ordinateur du cluster dans AD est recommande en environnement securise. Cela permet de controler l'OU de destination et les permissions sans donner de droits eleves a l'administrateur de cluster.

## Checklist avant validation

Avant de lancer l'assistant de validation, verifiez :

- [ ] Meme edition et meme CU de Windows Server sur tous les noeuds
- [ ] Fonctionnalite Failover Clustering installee sur tous les noeuds
- [ ] Adresses IP statiques sur tous les noeuds
- [ ] Connectivite reseau entre tous les noeuds (ping, DNS)
- [ ] Stockage partage accessible depuis tous les noeuds
- [ ] Pare-feu configure pour autoriser le trafic cluster
- [ ] Noeuds membres du meme domaine Active Directory
- [ ] Droits AD suffisants pour la creation du cluster

## Tests de validation du cluster

### Lancer la validation

L'assistant de validation execute une batterie de tests pour verifier la compatibilite de la configuration.

```powershell
# Run all validation tests (recommended before cluster creation)
Test-Cluster -Node "NODE1", "NODE2" -Verbose

# Run specific test categories only
Test-Cluster -Node "NODE1", "NODE2" -Include "Storage", "Network"
```

Resultat :

```text
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        01/15/2025   14:32         185420 Validation Report 2025.01.15 At 14.32.12.htm

WARNING: Testing Storage
Test                          Result
----                          ------
List Potential Cluster Disks  Success
Validate Disk Access          Success
Validate Disk Arbitration     Success
Validate Disk Failover        Success

WARNING: Testing Network
Test                          Result
----                          ------
List Network Binding Order    Success
Validate Network Communication Success
Validate IP Configuration     Success

Test-Cluster completed successfully.
```

Via l'interface graphique :

1. Ouvrir **Failover Cluster Manager**
2. Cliquer sur **Validate Configuration** dans le panneau Actions
3. Ajouter les noeuds a tester
4. Choisir entre executer tous les tests ou selectionner des categories

### Categories de tests

| Categorie | Verification effectuee |
|---|---|
| **Inventory** | Configuration materielle, pilotes, mises a jour |
| **Network** | Connectivite, latence, configuration IP, DNS |
| **Storage** | Acces aux disques partages, performance I/O |
| **System Configuration** | Coherence OS, services, fonctionnalites installees |
| **Hyper-V** | Configuration specifique si cluster Hyper-V |

### Interpreter les resultats

Les resultats sont classes en trois niveaux :

```mermaid
graph TD
    R[Resultats de validation] --> S["Succes<br/>(coche verte)"]
    R --> W["Avertissement<br/>(triangle jaune)"]
    R --> E["Echec<br/>(croix rouge)"]
    S --> OK[Cluster supportable]
    W --> WD["Cluster supportable<br/>mais correctifs recommandes"]
    E --> NOK["Cluster NON supportable<br/>corriger avant creation"]
```

!!! danger "Tests obligatoires"

    Microsoft exige que **tous les tests de validation reussissent** (pas d'echec) pour que le cluster soit dans une configuration supportee. Les avertissements sont acceptables mais doivent etre analyses.

```powershell
# View the validation report (HTML file)
# The report path is displayed at the end of Test-Cluster
# Default location:
$reportPath = "$env:SystemRoot\Cluster\Reports"
Get-ChildItem -Path $reportPath -Filter "*.htm" | Sort-Object LastWriteTime -Descending | Select-Object -First 1
```

Resultat :

```text
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----        01/15/2025   14:32         185420 Validation Report 2025.01.15 At 14.32.12.htm
```

### Problemes courants et solutions

| Probleme | Cause probable | Solution |
|---|---|---|
| Echec test reseau | Pare-feu bloquant | Ouvrir les ports 135, 3343 et RPC dynamique |
| Echec test stockage | LUN non visible | Verifier iSCSI/FC, rescanner les disques |
| Avertissement pilotes | Versions differentes | Mettre a jour les pilotes sur tous les noeuds |
| Echec configuration | CU differentes | Appliquer les memes mises a jour partout |
| DNS non resolu | Enregistrement manquant | Verifier les zones DNS, forcer `ipconfig /registerdns` |

```powershell
# Force DNS registration on all nodes
Invoke-Command -ComputerName "NODE1", "NODE2" -ScriptBlock {
    Register-DnsClient
}

# Rescan disks on all nodes
Invoke-Command -ComputerName "NODE1", "NODE2" -ScriptBlock {
    Update-HostStorageCache
    Get-Disk | Where-Object { $_.OperationalStatus -eq 'Offline' } | Set-Disk -IsOffline $false
}
```

!!! example "Scenario pratique"

    **Contexte :** Marc, technicien infrastructure, doit preparer deux serveurs pour un cluster de basculement. Le projet demarre lundi et il doit valider les prerequis avant la fin de la semaine.

    **Probleme :** La validation echoue avec des erreurs sur les tests reseau et stockage.

    **Diagnostic :**

    1. Marc lance la validation complete :

        ```powershell
        Test-Cluster -Node "SRV-01", "SRV-02" -Verbose
        ```

        Resultat :

        ```text
        WARNING: Testing Network
        Validate Network Communication    FAILED
        WARNING: Testing Storage
        Validate Disk Access              FAILED
        ```

    2. Il verifie la connectivite reseau :

        ```powershell
        Test-Connection -ComputerName "SRV-02" -Count 4
        ```

        Resultat :

        ```text
        Test-Connection : Testing connection to computer 'SRV-02' failed:
        No resources are available to complete this operation.
        ```

    3. Il identifie que le pare-feu bloque le trafic cluster :

        ```powershell
        Get-NetFirewallRule -DisplayGroup "Failover Cluster*" | Select-Object DisplayName, Enabled
        ```

        Resultat :

        ```text
        DisplayName                              Enabled
        -----------                              -------
        Failover Clusters (UDP-In)               False
        Failover Clusters (TCP-In)               False
        ```

    4. Il active les regles de pare-feu et relance la validation :

        ```powershell
        Get-NetFirewallRule -DisplayGroup "Failover Cluster*" | Enable-NetFirewallRule
        Test-Cluster -Node "SRV-01", "SRV-02" -Include "Network"
        ```

        Resultat :

        ```text
        Validate Network Communication    Success
        Validate IP Configuration         Success
        ```

    5. Pour le stockage, il decouvre que les LUNs iSCSI ne sont pas connectes sur SRV-02 :

        ```powershell
        Invoke-Command -ComputerName "SRV-02" -ScriptBlock { Get-IscsiConnection }
        ```

        Resultat :

        ```text
        (aucun resultat - pas de connexion iSCSI)
        ```

    **Solution :** Marc configure l'initiateur iSCSI sur SRV-02, connecte les LUNs, et la validation passe integralement.

!!! danger "Erreurs courantes"

    - **Lancer la creation du cluster sans validation** : Microsoft ne supportera pas un cluster cree sans validation reussie. Toujours executer `Test-Cluster` avant `New-Cluster`.
    - **Oublier d'installer la fonctionnalite Failover Clustering sur tous les noeuds** : la validation echouera si un seul noeud ne dispose pas de la fonctionnalite. Verifiez avec `Get-WindowsFeature -Name Failover-Clustering` sur chaque noeud.
    - **Utiliser des adresses IP dynamiques (DHCP)** : les noeuds du cluster doivent avoir des IP statiques. Un changement d'adresse IP en cours de fonctionnement cassera le cluster.
    - **Ne pas verifier la coherence des mises a jour** : deux noeuds avec des niveaux de CU differents peuvent provoquer des comportements imprevisibles. Utilisez `Get-HotFix` pour comparer.
    - **Ignorer les avertissements de validation** : meme si le cluster peut etre cree avec des warnings, ils indiquent souvent des problemes qui se manifesteront en production.

## Points cles a retenir

- La validation est une **etape obligatoire** avant la creation d'un cluster supporte par Microsoft
- Tous les noeuds doivent etre **identiques** en termes d'OS, de mises a jour et de fonctionnalites
- Deux reseaux minimum sont necessaires : production et heartbeat
- Le pre-staging du CNO dans Active Directory simplifie la gestion des permissions
- Les rapports de validation sont au format HTML et stockes dans `%SystemRoot%\Cluster\Reports`
- Un echec dans les tests de validation doit etre corrige **avant** de creer le cluster

## Pour aller plus loin

- Creation du cluster : [Creation du cluster](creation-cluster.md)
- Concepts du clustering : [Concepts Cluster](concepts-cluster.md)
- Documentation Microsoft : Validate Hardware for a Failover Cluster

