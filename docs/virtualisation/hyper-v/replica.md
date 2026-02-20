---
title: "Hyper-V Replica"
description: "Replication de VMs Hyper-V pour la reprise d'activite : configuration, types de basculement et strategie de disaster recovery sur Windows Server 2022."
tags:
  - virtualisation
  - hyper-v
  - replica
  - disaster-recovery
  - haute-disponibilite
---

# Hyper-V Replica

<span class="level-intermediate">Intermediaire</span> · Temps estime : 35 minutes

Hyper-V Replica replique une machine virtuelle d'un hote principal vers un hote secondaire de maniere asynchrone. En cas de panne du site principal, la VM repliquee peut etre activee sur le site secondaire, assurant la continuite d'activite.

---

## Principe de fonctionnement

!!! example "Analogie"

    Hyper-V Replica fonctionne comme un **systeme de copie automatique de documents entre deux bureaux**. Votre bureau principal (site primaire) travaille normalement sur ses dossiers (VMs). Toutes les 5 minutes, un coursier envoie une copie des pages modifiees au bureau de secours (site secondaire). Si le bureau principal est inonde (sinistre), le bureau de secours a une copie recente de tous les dossiers et peut reprendre le travail presque immediatement.

```mermaid
flowchart LR
    subgraph SitePrimaire["Site principal"]
        VM_Primary["VM Production<br/>SRV-APP01"]
        HV1["Hote Hyper-V primaire<br/>HV-SRV01"]
        VM_Primary --> HV1
    end

    subgraph SiteSecondaire["Site secondaire (DR)"]
        VM_Replica["VM Repliquee<br/>SRV-APP01 (Replica)"]
        HV2["Hote Hyper-V replica<br/>HV-DR01"]
        VM_Replica --> HV2
    end

    HV1 -->|"Replication asynchrone<br/>(5 min / 30 sec)"| HV2

    style VM_Replica fill:#666,color:#fff
```

### Caracteristiques

| Aspect | Detail |
|--------|--------|
| **Type de replication** | Asynchrone (pas de perte zero garantie) |
| **Frequences** | 30 secondes, 5 minutes ou 15 minutes |
| **Transport** | HTTP (port 80) ou HTTPS (port 443) |
| **Points de recuperation** | Jusqu'a 24 points de recuperation supplementaires |
| **Replication etendue** | Chaine primaire > replica > replica etendu |
| **Cout** | Aucune licence supplementaire |

---

## Prerequis

- Deux hotes Hyper-V (meme domaine ou domaines approuves, ou certificats pour les hotes standalone)
- Connectivite reseau entre les deux sites
- Espace de stockage suffisant sur le site secondaire
- Pare-feu ouvert sur les ports HTTP (80) ou HTTPS (443)

---

## Configuration

### Etape 1 : activer le serveur Replica (hote destination)

```powershell
# On the REPLICA server (destination)
# Enable Hyper-V Replica as a Replica server

# Option A: HTTP authentication (lab / same domain)
Set-VMReplicationServer -ReplicationEnabled $true `
    -AllowedAuthenticationType Kerberos `
    -ReplicationAllowedFromAnyServer $true `
    -DefaultStorageLocation "D:\Hyper-V\Replica"

# Option B: HTTPS authentication (production / cross-domain)
Set-VMReplicationServer -ReplicationEnabled $true `
    -AllowedAuthenticationType Certificate `
    -CertificateThumbprint "<certificate-thumbprint>" `
    -DefaultStorageLocation "D:\Hyper-V\Replica"

# Verify Replica server configuration
Get-VMReplicationServer
```

Resultat :

```text
RepEnabled AllowedAuthType KerberosAuthPort CertAuthPort DefaultStoreLoc
---------- --------------- ---------------- ------------ ---------------
True       Kerberos        80                            D:\Hyper-V\Replica
```

### Etape 2 : configurer le pare-feu

```powershell
# On the REPLICA server: allow incoming replication
# For HTTP (Kerberos authentication)
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTP Listener (TCP-In)"

# For HTTPS (Certificate authentication)
Enable-NetFirewallRule -DisplayName "Hyper-V Replica HTTPS Listener (TCP-In)"

# Verify firewall rules
Get-NetFirewallRule -DisplayName "Hyper-V Replica*" |
    Select-Object DisplayName, Enabled, Direction
```

Resultat :

```text
DisplayName                                  Enabled Direction
-----------                                  ------- ---------
Hyper-V Replica HTTP Listener (TCP-In)       True    Inbound
Hyper-V Replica HTTPS Listener (TCP-In)      False   Inbound
```

### Etape 3 : activer la replication sur la VM (hote source)

```powershell
# On the PRIMARY server: enable replication for a VM
Enable-VMReplication -VMName "SRV-APP01" `
    -ReplicaServerName "HV-DR01.lab.local" `
    -ReplicaServerPort 80 `
    -AuthenticationType Kerberos `
    -ReplicationFrequencySec 300 `
    -RecoveryHistory 4 `
    -CompressionEnabled $true

# Start the initial replication
Start-VMInitialReplication -VMName "SRV-APP01"

# Options for initial replication:
# - Over network (default)
# - Export to external media
# - Use existing copy on the replica server
```

### Replication initiale via media externe

Pour les VMs volumineuses ou les liaisons lentes :

```powershell
# Export initial replication to external media
Start-VMInitialReplication -VMName "SRV-APP01" `
    -DestinationPath "E:\InitialReplica" `
    -InitialReplicationStartTime "2025-01-15 22:00:00"

# On the Replica server: import the initial replication
# Copy the export to the replica server, then:
Import-VMInitialReplication -VMName "SRV-APP01" `
    -Path "E:\InitialReplica\SRV-APP01"
```

---

## Supervision de la replication

```powershell
# Check replication health for a VM
Get-VMReplication -VMName "SRV-APP01" |
    Select-Object VMName, State, Health, Mode,
        PrimaryServerName, ReplicaServerName,
        ReplicationFrequencySec,
        LastReplicationTime

# Detailed replication statistics
Measure-VMReplication -VMName "SRV-APP01" |
    Select-Object VMName,
        AverageReplicationSize,
        AverageReplicationLatency,
        ReplicationHealth,
        LastReplicationTime,
        @{N='MissedCount';E={$_.MissedReplicationCount}}

# Check replication health for ALL VMs
Get-VMReplication | Select-Object VMName, State, Health, LastReplicationTime |
    Format-Table -AutoSize
```

Resultat :

```text
VMName    State      Health   Mode    PrimaryServerName   ReplicaServerName    ReplicationFrequencySec LastReplicationTime
------    -----      ------   ----    -----------------   -----------------    ----------------------- -------------------
SRV-APP01 Replicating Normal  Primary HV-SRV01.lab.local  HV-DR01.lab.local    300                     2026-02-20 14:25:00

VMName                  AverageReplicationSize AverageReplicationLatency ReplicationHealth LastReplicationTime  MissedCount
------                  ---------------------- ------------------------- ----------------- -------------------  -----------
SRV-APP01               45 MB                  00:00:12                  Normal            2026-02-20 14:25:00  0

VMName    State       Health   LastReplicationTime
------    -----       ------   -------------------
SRV-APP01 Replicating Normal   2026-02-20 14:25:00
SRV-WEB01 Replicating Normal   2026-02-20 14:24:30
DC-01     Replicating Warning  2026-02-20 14:15:00
```

### Etats de sante

| Etat | Signification | Action |
|------|---------------|--------|
| **Normal** | Replication fonctionne correctement | Aucune |
| **Warning** | Retards de replication | Verifier la bande passante |
| **Critical** | Replication echouee | Verifier la connectivite et les journaux |

---

## Types de basculement (Failover)

!!! example "Analogie"

    Imaginez trois scenarios de demenagement. Le **Test Failover**, c'est visiter le nouvel appartement avec un plan pour verifier que vos meubles rentrent, sans rien deplacer. Le **Planned Failover**, c'est un demenagement organise : vous emballez tout proprement, verifiez que rien ne manque, puis vous installez dans le nouvel appartement. Le **Unplanned Failover**, c'est un incendie : vous courez dans le nouvel appartement avec ce que le coursier avait deja copie, en acceptant que les dernieres modifications soient peut-etre perdues.

### Test Failover (sans impact)

Le test failover cree une copie temporaire de la VM repliquee pour valider la replication. La VM de production continue de fonctionner normalement.

```powershell
# Start a test failover (creates a temporary test VM)
Start-VMFailover -VMName "SRV-APP01" -AsTest

# The test VM is named "SRV-APP01 - Test"
# Verify it works, then clean up:
Stop-VMFailover -VMName "SRV-APP01"
```

### Planned Failover (maintenance planifiee)

Le basculement planifie assure zero perte de donnees en synchronisant les dernieres modifications avant le basculement.

```powershell
# 1. On the PRIMARY: stop the VM and send final replication
Stop-VM -Name "SRV-APP01"
Start-VMFailover -VMName "SRV-APP01" -Prepare

# 2. On the REPLICA: complete the planned failover
Start-VMFailover -VMName "SRV-APP01"

# 3. Start the VM on the replica site
Start-VM -Name "SRV-APP01"

# 4. Reverse the replication direction
Set-VMReplication -VMName "SRV-APP01" -Reverse
```

### Unplanned Failover (sinistre)

Le basculement non planifie est utilise en cas de panne du site principal. Il peut entrainer une perte de donnees correspondant au dernier intervalle de replication.

```powershell
# On the REPLICA server (primary is unavailable)
Start-VMFailover -VMName "SRV-APP01"

# Choose a recovery point if multiple are available
$recoveryPoints = Get-VMSnapshot -VMName "SRV-APP01" -SnapshotType Replica
$recoveryPoints | Select-Object Name, CreationTime

# Use a specific recovery point
Start-VMFailover -VMName "SRV-APP01" -VMRecoverySnapshot $recoveryPoints[0]

# Complete and start the VM
Complete-VMFailover -VMName "SRV-APP01"
Start-VM -Name "SRV-APP01"
```

```mermaid
flowchart TD
    A{Type de basculement ?}
    A -->|Test| B[Test Failover]
    A -->|Planifie| C[Planned Failover]
    A -->|Sinistre| D[Unplanned Failover]

    B --> B1["VM de test creee<br/>Aucun impact production"]
    B1 --> B2[Validation puis nettoyage]

    C --> C1["Arret VM primaire<br/>Synchro finale"]
    C1 --> C2["Demarrage sur replica<br/>Zero perte de donnees"]

    D --> D1["Demarrage sur replica<br/>Perte potentielle selon RPO"]
    D1 --> D2["Choix du point de recuperation"]
```

---

## Replication etendue

La replication etendue permet de chainer un troisieme site :

```
Site A (Primaire) --> Site B (Replica) --> Site C (Extended Replica)
```

```powershell
# On Site B: enable extended replication to Site C
Enable-VMReplication -VMName "SRV-APP01" `
    -ReplicaServerName "HV-DR02.lab.local" `
    -ReplicaServerPort 80 `
    -AuthenticationType Kerberos `
    -ReplicationFrequencySec 900

Start-VMInitialReplication -VMName "SRV-APP01"
```

!!! warning "Limites de la replication etendue"

    - Frequence minimale : 5 minutes (pas de 30 secondes)
    - Pas de points de recuperation supplementaires
    - Replication etendue uniquement depuis le replica, pas depuis le primaire

---

## Bonnes pratiques

- **Tester regulierement** le failover (Test Failover mensuel)
- Utiliser **HTTPS** pour la replication entre sites distants
- Activer la **compression** pour reduire la bande passante
- Configurer des **alertes** sur l'etat de sante de la replication
- Documenter la **procedure de failover** et la tester
- Prevoir le **dimensionnement reseau** : bande passante suffisante pour le volume de donnees modifiees

---

## Points cles a retenir

- Hyper-V Replica fournit une **replication asynchrone** de VMs pour la reprise d'activite
- Trois frequences de replication : **30 secondes**, **5 minutes** ou **15 minutes**
- Le **Test Failover** valide la replication sans impacter la production
- Le **Planned Failover** garantit zero perte de donnees (maintenance planifiee)
- Le **Unplanned Failover** est utilise en cas de sinistre (perte possible selon le RPO)
- La replication etendue permet de proteger sur un **troisieme site**
- Hyper-V Replica est **gratuit** (inclus dans la licence Windows Server)

---

!!! example "Scenario pratique"

    **Contexte :** Laurent, responsable infrastructure dans une entreprise de e-commerce, a configure Hyper-V Replica entre le site principal (Paris) et le site de secours (Lyon) pour les VMs critiques. La replication est configuree toutes les 5 minutes via HTTP.

    **Probleme :** Le tableau de bord montre un etat "Warning" sur la replication de `SRV-APP01` depuis 2 jours. La replication accumule du retard.

    **Diagnostic :**

    ```powershell
    # Check replication statistics
    Measure-VMReplication -VMName "SRV-APP01" |
        Select-Object VMName, ReplicationHealth, AverageReplicationSize,
            AverageReplicationLatency, MissedReplicationCount
    ```

    ```text
    VMName    ReplicationHealth AverageReplicationSize AverageReplicationLatency MissedReplicationCount
    ------    ----------------- ---------------------- ------------------------- ----------------------
    SRV-APP01 Warning           250 MB                 00:04:55                  12
    ```

    La taille moyenne de replication etait de 250 Mo par cycle, avec une latence proche des 5 minutes. La liaison WAN entre Paris et Lyon ne suffisait plus.

    **Solution :**

    ```powershell
    # Enable compression to reduce bandwidth usage
    Set-VMReplication -VMName "SRV-APP01" -CompressionEnabled $true

    # Increase replication frequency to reduce per-cycle size
    Set-VMReplication -VMName "SRV-APP01" -ReplicationFrequencySec 300

    # Resynchronize to resolve the warning state
    Resume-VMReplication -VMName "SRV-APP01" -Resynchronize
    ```

    Apres activation de la compression, la taille moyenne de replication est passee de 250 Mo a 80 Mo par cycle. Laurent a egalement demande une augmentation de la bande passante WAN et planifie un Test Failover mensuel pour valider la replication.

!!! danger "Erreurs courantes"

    1. **Ne jamais tester le failover** : Configurer la replication sans jamais faire de Test Failover, c'est comme avoir une assurance sans verifier les clauses. Planifiez un test mensuel pour valider que la VM repliquee demarre correctement.

    2. **Oublier de configurer le pare-feu** : La replication echoue silencieusement si les regles de pare-feu pour les ports HTTP (80) ou HTTPS (443) ne sont pas activees sur le serveur replica.

    3. **Utiliser HTTP entre sites distants** : HTTP transmet les donnees en clair. Pour la replication entre sites distants (surtout via Internet), utilisez toujours HTTPS avec des certificats.

    4. **Ignorer les alertes Warning** : Un etat Warning signifie que la replication accumule du retard. Si le site principal tombe, la perte de donnees sera superieure au RPO prevu. Reagissez immediatement.

    5. **Oublier de configurer le DNS apres un failover** : Apres un basculement, la VM repliquee demarre avec la meme adresse IP. Si les clients resolvent le nom via DNS, il faut mettre a jour l'enregistrement DNS pour pointer vers le site de secours.

---

## Pour aller plus loin

- Live Migration pour la mobilite sans interruption (voir la page [Live Migration](live-migration.md))
- Microsoft : Hyper-V Replica deployment guide
- Microsoft : Set up disaster recovery with Hyper-V Replica
