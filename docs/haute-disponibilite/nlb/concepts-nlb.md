---
title: "Concepts du Network Load Balancing"
description: "Comprendre le Network Load Balancing (NLB) sous Windows Server 2022 : unicast, multicast, affinite, regles de port et cas d'usage."
tags:
  - haute-disponibilite
  - nlb
  - load-balancing
  - windows-server
---

# Concepts du Network Load Balancing

<span class="level-advanced">Avance</span> · Temps estime : 35 minutes

## Introduction

Le **Network Load Balancing** (NLB) est une fonctionnalite integree a Windows Server qui distribue le trafic reseau TCP/IP entre plusieurs serveurs. Contrairement au failover clustering qui offre de la haute disponibilite pour des services avec etat (stateful), NLB est optimise pour les services **sans etat** (stateless) comme les serveurs web IIS.

!!! warning "NLB vs solutions modernes"

    NLB reste disponible dans Windows Server 2022 mais est de plus en plus remplace par des solutions plus modernes : Azure Load Balancer, HAProxy, ou le role Windows Server Software Load Balancer (SLB). NLB reste pertinent pour les environnements on-premise sans infrastructure supplementaire.

!!! example "Analogie"

    NLB fonctionne comme les caisses d'un supermarche. Quand un client (requete) arrive, il est dirige vers la caisse (serveur) la moins occupee. Si une caisse ferme (panne d'un noeud), les clients sont rediriges vers les caisses restantes. L'affinite, c'est comme un client fidele qui revient toujours a la meme caisse car la caissiere connait deja ses preferences (session).

## Architecture NLB

```mermaid
graph TD
    C[Clients] --> VIP["IP Virtuelle NLB<br/>192.168.1.200"]
    VIP --> N1["Noeud 1 - IIS<br/>192.168.1.11"]
    VIP --> N2["Noeud 2 - IIS<br/>192.168.1.12"]
    VIP --> N3["Noeud 3 - IIS<br/>192.168.1.13"]
```

### Principes de fonctionnement

- Tous les noeuds NLB recoivent **chaque paquet** du trafic entrant
- Chaque noeud applique un algorithme de filtrage pour determiner s'il doit traiter le paquet
- Un seul noeud traite chaque connexion client (pas de duplication de reponse)
- Jusqu'a **32 noeuds** par cluster NLB

!!! info "Pas de routeur ou switch special"

    NLB fonctionne au niveau de la couche 2 (MAC) ou couche 3 (IP). Il ne necessite pas de materiel specifique, contrairement aux load balancers materiels (F5, Citrix NetScaler).

## Modes de fonctionnement

### Mode Unicast

En mode **unicast**, NLB remplace l'adresse MAC physique de la carte reseau par une adresse MAC virtuelle partagee par tous les noeuds.

| Avantage | Inconvenient |
|---|---|
| Simple a configurer | Communication entre noeuds impossible via l'interface NLB |
| Compatible avec tous les switchs | Necessite une 2e carte reseau pour la gestion |
| Aucune configuration switch requise | Peut provoquer du flooding sur le switch |

!!! warning "Flooding en mode unicast"

    En mode unicast, le switch ne peut pas apprendre l'adresse MAC virtuelle car elle est partagee. Il diffuse donc le trafic sur tous les ports (flooding), ce qui peut impacter les performances du reseau.

### Mode Multicast

En mode **multicast**, NLB ajoute une adresse MAC multicast sans remplacer l'adresse MAC physique. Les noeuds conservent leur adresse MAC d'origine.

| Avantage | Inconvenient |
|---|---|
| Communication entre noeuds possible | Necessite une entree ARP statique sur le routeur |
| Pas de flooding systematique | Configuration switch/routeur necessaire |
| Meilleure performance reseau | Pas supporte par tous les equipements reseau |

### Mode IGMP Multicast

Variante du mode multicast qui utilise **IGMP** (Internet Group Management Protocol) pour limiter la diffusion aux ports des noeuds NLB uniquement.

```powershell
# Set NLB cluster mode
# Unicast = 1, Multicast = 2, IGMP Multicast = 3
Set-NlbCluster -ClusterName "YOURNLB" -OperationMode Multicast
```

Resultat :

```text
IPAddress        : 10.0.0.200
SubnetMask       : 255.255.255.0
ClusterName      : NLB-WEB.lab.local
OperationMode    : Multicast
```

### Comparaison des modes

| Critere | Unicast | Multicast | IGMP Multicast |
|---|---|---|---|
| 2e carte reseau requise | Oui | Non | Non |
| Configuration switch | Non | Oui (ARP statique) | Oui (IGMP snooping) |
| Flooding | Oui | Limite | Non |
| Communication inter-noeuds | Non (via NLB NIC) | Oui | Oui |

## Affinite (Affinity)

L'**affinite** determine comment les connexions clientes sont distribuees entre les noeuds.

### Types d'affinite

| Mode | Description | Cas d'usage |
|---|---|---|
| **None** | Chaque connexion est repartie sans consideration de source | Services 100% stateless |
| **Single** | Toutes les connexions d'une meme IP client vont au meme noeud | Sessions applicatives (ASP.NET) |
| **Network (Class C)** | Toutes les connexions d'un meme sous-reseau /24 vont au meme noeud | Clients derriere un proxy |

```powershell
# Set affinity mode on a port rule
Set-NlbClusterPortRule -ClusterName "YOURNLB" `
    -Port 80 `
    -Affinity Single
```

Resultat :

```text
IPAddress  StartPort EndPort Protocol Mode     Affinity Timeout
---------  --------- ------- -------- ----     -------- -------
10.0.0.200 80        80      TCP      Multiple Single   0
```

!!! tip "Sessions et affinite"

    Si votre application web utilise des sessions en memoire (non distribuees), l'affinite **Single** est indispensable. Sinon, un utilisateur pourrait perdre sa session a chaque requete. La meilleure approche reste de stocker les sessions en base de donnees ou dans un cache distribue.

## Regles de port

Les **regles de port** (port rules) definissent comment le trafic est reparti pour chaque plage de ports.

### Types de regles

| Type | Description |
|---|---|
| **Multiple Hosts** | Trafic reparti entre tous les noeuds (mode standard) |
| **Single Host** | Tout le trafic va a un seul noeud (failover actif-passif) |
| **Disabled** | Le trafic sur ces ports est bloque |

```powershell
# Add a port rule for HTTP (80) and HTTPS (443) with load distribution
Add-NlbClusterPortRule -ClusterName "YOURNLB" `
    -StartPort 80 `
    -EndPort 80 `
    -Protocol TCP `
    -Mode Multiple `
    -Affinity Single

Add-NlbClusterPortRule -ClusterName "YOURNLB" `
    -StartPort 443 `
    -EndPort 443 `
    -Protocol TCP `
    -Mode Multiple `
    -Affinity Single
```

### Ponderation (Load Weight)

Chaque noeud peut avoir un poids de charge different pour repartir le trafic de maniere inegale.

```powershell
# Set load weight for a specific node (0-100)
# Higher weight = more traffic
Set-NlbClusterPortRule -ClusterName "YOURNLB" `
    -Port 80 `
    -HostID 1 `
    -LoadWeight 80

Set-NlbClusterPortRule -ClusterName "YOURNLB" `
    -Port 80 `
    -HostID 2 `
    -LoadWeight 20
```

## Drain Stop

Le **drain stop** est une fonctionnalite qui permet de retirer gracieusement un noeud du cluster NLB. Le noeud continue de traiter les connexions **existantes** mais n'accepte plus de **nouvelles** connexions.

```powershell
# Drain stop a node (finish existing connections, reject new ones)
Stop-NlbClusterNode -HostName "NODE1" -Drain

# Check drain status
Get-NlbClusterNode -HostName "NODE1" | Format-List Name, HostState

# Force stop (immediately drops all connections)
Stop-NlbClusterNode -HostName "NODE1"

# Resume a node
Start-NlbClusterNode -HostName "NODE1"
```

Resultat :

```text
Name      : SRV-01
HostState : Suspended

Name      : SRV-01
HostState : Stopped

Name      : SRV-01
HostState : Started
```

```mermaid
sequenceDiagram
    participant Admin
    participant N1 as Noeud 1
    participant N2 as Noeud 2
    participant Client1 as Client existant
    participant Client2 as Nouveau client

    Admin->>N1: Drain Stop
    Note over N1: Etat : Draining
    Client1->>N1: Connexion existante (acceptee)
    Client2->>N2: Nouvelle connexion (redirigee)
    Client1->>N1: Fin de la session
    Note over N1: Etat : Stopped
    Admin->>N1: Maintenance possible
```

!!! tip "Maintenance sans interruption"

    Utilisez toujours le drain stop avant une maintenance planifiee. Cela evite de couper brutalement les sessions utilisateur en cours.

## Cas d'usage typiques

| Service | Configuration NLB recommandee |
|---|---|
| **Serveurs web IIS** | Multicast, affinite Single, ports 80/443 |
| **Remote Desktop Gateway** | Unicast, affinite Single, port 443 |
| **ADFS (federation)** | Multicast, affinite Single, port 443 |
| **Exchange CAS** | Deconseille (utiliser le load balancing L4/L7) |
| **VPN (SSTP)** | Unicast, affinite Single, port 443 |

## Limitations de NLB

| Limitation | Impact |
|---|---|
| Pas de health check applicatif | NLB ne detecte pas si le service (IIS) a plante, seulement si le noeud est injoignable |
| Pas de load balancing L7 | Pas d'inspection du contenu HTTP (URL, cookies) |
| Flooding en unicast | Impact reseau potentiel |
| 32 noeuds maximum | Suffisant pour la plupart des cas |
| Pas de support UDP fiable | Certaines applications UDP posent probleme |

!!! example "Scenario pratique"

    **Contexte :** Thomas, administrateur web dans un cabinet comptable, gere deux serveurs IIS (SRV-01 et SRV-02) qui hebergent l'application web interne de comptabilite. Les utilisateurs se plaignent que l'application est lente aux heures de pointe et qu'elle est inaccessible lors des maintenances.

    **Probleme :** Tout le trafic passe par un seul serveur (SRV-01). Quand il est en maintenance, l'application est indisponible.

    **Diagnostic et solution :**

    1. Thomas decide de deployer NLB pour repartir la charge et assurer la disponibilite :

        ```powershell
        # Install NLB on both servers
        Install-WindowsFeature -Name NLB -IncludeManagementTools -ComputerName SRV-01
        Install-WindowsFeature -Name NLB -IncludeManagementTools -ComputerName SRV-02
        ```

    2. Il choisit le mode Multicast avec affinite Single (sessions en memoire) :

        ```powershell
        New-NlbCluster -HostName "SRV-01" -ClusterName "NLB-WEB" `
            -InterfaceName "Ethernet" -ClusterPrimaryIP "10.0.0.200" `
            -SubnetMask "255.255.255.0" -OperationMode Multicast
        ```

    3. Il verifie le fonctionnement en envoyant des requetes :

        ```powershell
        1..10 | ForEach-Object {
            (Invoke-WebRequest -Uri "http://10.0.0.200" -UseBasicParsing).StatusCode
        }
        ```

        Resultat :

        ```text
        200
        200
        200
        200
        200
        200
        200
        200
        200
        200
        ```

    4. Pour la maintenance de SRV-01, il utilise le drain stop :

        ```powershell
        Stop-NlbClusterNode -HostName "SRV-01" -Drain
        ```

    **Resultat :** L'application est deux fois plus reactive aux heures de pointe et les maintenances se font sans interruption grace au drain stop.

!!! danger "Erreurs courantes"

    - **Utiliser NLB pour des services avec etat (stateful) sans affinite** : sans affinite Single, un utilisateur peut etre dirige vers un serveur different a chaque requete et perdre sa session. Activez l'affinite ou stockez les sessions dans un cache distribue.
    - **Choisir le mode Unicast sans deuxieme carte reseau** : en mode Unicast, l'adresse MAC physique est remplacee. Sans deuxieme NIC, les noeuds ne peuvent plus communiquer entre eux pour la gestion.
    - **Ne pas supprimer la regle de port par defaut** : la regle par defaut couvre tous les ports (0-65535), ce qui repartit aussi le trafic RDP, ICMP et autre. Supprimez-la et creez des regles specifiques pour les ports applicatifs.
    - **Confondre NLB et failover clustering** : NLB repartit la charge entre des serveurs identiques pour les services stateless. Le failover clustering offre la haute disponibilite pour les services avec etat (SQL, fichiers). Ce ne sont pas interchangeables.
    - **Ignorer l'absence de health check applicatif** : NLB ne detecte pas si IIS a plante, seulement si le noeud est injoignable. Combinez NLB avec une surveillance externe (monitoring) pour detecter les pannes applicatives.

## Points cles a retenir

- NLB distribue le trafic reseau entre plusieurs serveurs identiques pour les **services stateless**
- Le mode **unicast** est le plus simple mais provoque du flooding ; le **multicast** est preferable
- L'**affinite** garantit qu'un client est toujours dirige vers le meme noeud (essentiel pour les sessions)
- Le **drain stop** permet une maintenance sans interruption des sessions existantes
- NLB n'effectue pas de health check applicatif : combinez-le avec une surveillance externe
- Pour les charges modernes, envisagez Azure Load Balancer ou des solutions L7 (reverse proxy)

## Pour aller plus loin

- Configuration NLB : [Configuration](configuration.md)
- Documentation Microsoft : Network Load Balancing Overview
