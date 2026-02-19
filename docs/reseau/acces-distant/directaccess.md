---
title: "DirectAccess"
description: "Comprendre DirectAccess : concepts, technologies de transition IPv6 (6to4, Teredo, ISATAP), avantages, inconvenients et statut de depreciation."
tags:
  - reseau
  - directaccess
  - ipv6
  - acces-distant
  - windows-server
---

# DirectAccess

## Introduction

**DirectAccess** est une technologie d'acces distant introduite avec Windows Server 2008 R2 qui permet aux ordinateurs joints au domaine de se connecter automatiquement et de maniere transparente au reseau d'entreprise, sans intervention de l'utilisateur. Contrairement au VPN traditionnel, DirectAccess ne necessite pas de connexion manuelle : des que l'ordinateur client detecte qu'il est sur un reseau externe, le tunnel est etabli automatiquement.

!!! warning "Technologie en fin de vie"

    Microsoft considere DirectAccess comme une technologie **en fin de vie** et recommande **Always On VPN** comme successeur. DirectAccess reste disponible dans Windows Server 2022 mais ne recoit plus d'evolutions fonctionnelles.

---

## Principes de fonctionnement

### Connexion transparente

DirectAccess etablit des tunnels **IPsec** bidirectionnels entre le client et le serveur DirectAccess. Ces tunnels sont crees automatiquement au demarrage de l'ordinateur, avant meme l'ouverture de session de l'utilisateur.

```mermaid
sequenceDiagram
    participant C as Client DirectAccess
    participant NLS as Network Location Server
    participant DA as Serveur DirectAccess
    participant RES as Ressources internes
    C->>NLS: Test de localisation (HTTPS)
    NLS-->>C: Echec (client externe)
    C->>DA: Tunnel IPsec infrastructure
    DA->>C: Tunnel etabli
    C->>C: Ouverture de session utilisateur
    C->>DA: Tunnel IPsec utilisateur
    C->>RES: Acces aux ressources internes (via DA)
```

### Deux tunnels

| Tunnel              | Moment de creation           | Utilisation                               |
|---------------------|------------------------------|-------------------------------------------|
| Tunnel infrastructure | Au demarrage de l'ordinateur | GPO, scripts de demarrage, gestion machine |
| Tunnel utilisateur    | Apres ouverture de session   | Acces aux ressources par l'utilisateur     |

### Network Location Server (NLS)

Le **NLS** est un serveur web interne (HTTPS) que le client DirectAccess contacte pour determiner s'il se trouve sur le reseau interne ou externe :

- **NLS accessible** : le client est sur le reseau interne, DirectAccess est desactive
- **NLS inaccessible** : le client est sur un reseau externe, DirectAccess s'active

!!! danger "Disponibilite du NLS"

    Le NLS est un composant critique. S'il est inaccessible depuis le reseau interne (panne, maintenance), les clients internes activeront DirectAccess a tort, causant des problemes de connectivite. Il est imperatif d'assurer la haute disponibilite du NLS.

---

## Technologies de transition IPv6

DirectAccess repose fondamentalement sur **IPv6**. Puisque la plupart des reseaux utilisent encore IPv4, DirectAccess utilise des technologies de transition pour encapsuler le trafic IPv6 dans des paquets IPv4.

### 6to4

| Propriete      | Detail                                                |
|----------------|-------------------------------------------------------|
| Prefixe        | `2002::/16`                                           |
| Encapsulation  | IPv6 dans IPv4 (protocole 41)                         |
| Prerequis      | Adresse IPv4 publique sur le client                   |
| Inconvenient   | Ne fonctionne pas derriere un NAT                     |
| Statut         | Obsolete (RFC 7526)                                   |

### Teredo

| Propriete      | Detail                                                |
|----------------|-------------------------------------------------------|
| Prefixe        | `2001:0000::/32`                                      |
| Encapsulation  | IPv6 dans UDP/IPv4                                    |
| Prerequis      | Serveur Teredo accessible                             |
| Avantage       | Fonctionne derriere un NAT                            |
| Inconvenient   | Performance reduite, necessite un serveur relais       |
| Statut         | En fin de vie                                         |

### ISATAP (Intra-Site Automatic Tunnel Addressing Protocol)

| Propriete      | Detail                                                |
|----------------|-------------------------------------------------------|
| Utilisation    | Communication IPv6 au sein d'un reseau IPv4 interne   |
| Encapsulation  | IPv6 dans IPv4 (protocole 41)                         |
| Portee         | Intra-site uniquement                                 |
| Statut         | Obsolete                                              |

### IP-HTTPS (protocole prefere)

| Propriete      | Detail                                                |
|----------------|-------------------------------------------------------|
| Encapsulation  | IPv6 dans HTTPS (TCP 443)                             |
| Avantage       | Traverse tous les pare-feux et proxys                 |
| Prerequis      | Certificat SSL valide sur le serveur DirectAccess     |
| Utilisation    | Methode de repli par defaut quand 6to4/Teredo echouent|

```mermaid
graph TD
    A["Client DirectAccess"] --> B{"IPv6 natif<br/>disponible ?"}
    B -->|Oui| C["Connexion IPv6 native"]
    B -->|Non| D{"Adresse IPv4<br/>publique ?"}
    D -->|Oui| E["6to4"]
    D -->|Non| F{"UDP sortant<br/>autorise ?"}
    F -->|Oui| G["Teredo"]
    F -->|Non| H["IP-HTTPS<br/>(TCP 443)"]
```

!!! info "En pratique"

    La grande majorite des clients DirectAccess utilisent **IP-HTTPS** car la plupart se trouvent derriere un NAT sans acces UDP direct. IP-HTTPS est le seul protocole qui fonctionne de maniere fiable dans tous les environnements.

---

## Prerequis de deploiement

### Cote serveur

| Prerequis                      | Detail                                          |
|--------------------------------|-------------------------------------------------|
| Windows Server 2012+           | Role Acces distant avec DirectAccess             |
| Active Directory               | Domaine AD DS obligatoire                        |
| Certificats                    | PKI interne pour les certificats machine         |
| DNS                            | Enregistrements DNS internes et externes         |
| NLS                            | Serveur web HTTPS interne (haute disponibilite)  |
| Interfaces reseau              | 2 interfaces recommandees (interne + externe)    |

### Cote client

| Prerequis                      | Detail                                          |
|--------------------------------|-------------------------------------------------|
| Windows Enterprise/Education   | Les editions Pro ne sont **pas** supportees      |
| Joint au domaine               | L'ordinateur doit etre membre du domaine AD      |
| Certificat machine             | Delivre par la PKI interne                       |
| IPv6                           | Active sur l'interface (par defaut)              |

!!! warning "Limitation majeure"

    DirectAccess ne fonctionne qu'avec les editions **Enterprise** et **Education** de Windows. Cette restriction exclut de nombreuses organisations utilisant Windows Pro, ce qui a contribue a son abandon au profit d'Always On VPN.

---

## Avantages et inconvenients

### Avantages

| Avantage                        | Description                                    |
|---------------------------------|------------------------------------------------|
| Connexion transparente          | Pas d'action requise de l'utilisateur           |
| Tunnel machine                  | Gestion a distance meme sans ouverture de session|
| Gestion par GPO                 | Configuration centralisee via Active Directory  |
| Securite IPsec                  | Chiffrement et authentification robustes        |
| Toujours connecte              | Application automatique des mises a jour et GPO |

### Inconvenients

| Inconvenient                    | Description                                    |
|---------------------------------|------------------------------------------------|
| Complexite de deploiement       | PKI, IPv6, NLS, certificats : mise en place lourde |
| Dependance IPv6                 | Necessite des technologies de transition        |
| Windows Enterprise uniquement   | Exclut les editions Pro                         |
| Performance IP-HTTPS            | Encapsulation dans HTTPS = surcharge            |
| Depreciation                    | Plus d'evolution, successeur : Always On VPN    |
| NLS critique                    | Point de defaillance unique si non redonde      |
| Diagnostic complexe             | Problemes difficiles a debugger (IPv6, IPsec)   |

---

## Statut de depreciation

Microsoft a annonce que DirectAccess est en **mode maintenance** :

- **Pas de nouvelles fonctionnalites** dans les versions futures de Windows Server
- Le role reste disponible dans Windows Server 2022 pour la compatibilite
- **Always On VPN** est le remplacement officiel
- Microsoft recommande la migration vers Always On VPN pour tous les nouveaux deployements

### Comparaison DirectAccess vs Always On VPN

| Critere                | DirectAccess            | Always On VPN             |
|------------------------|-------------------------|---------------------------|
| Editions Windows       | Enterprise uniquement   | Toutes les editions       |
| Dependance IPv6        | Obligatoire             | Non requise               |
| Infrastructure         | PKI + NLS + DA Server   | RRAS + NPS (plus simple)  |
| Protocoles VPN         | IPsec uniquement        | IKEv2, SSTP, etc.         |
| Gestion                | GPO uniquement          | GPO, MDM, SCCM            |
| Azure AD               | Non supporte            | Supporte                  |
| Statut                 | Maintenance             | Actif, en evolution       |

---

## Commandes de diagnostic

```powershell
# Check DirectAccess client status (on the client machine)
Get-DAConnectionStatus

# Display DirectAccess client configuration
Get-DAClientExperienceConfiguration

# Display DirectAccess server configuration
Get-RemoteAccess

# Test IP-HTTPS connectivity
Get-NetIPHttpsState

# Check DirectAccess infrastructure tunnel
netsh interface httpstunnel show interfaces

# Display transition technology status
netsh interface 6to4 show state
netsh interface teredo show state
netsh interface isatap show state
```

---

## Points cles a retenir

| Concept                  | Detail                                                      |
|--------------------------|-------------------------------------------------------------|
| Connexion transparente   | Tunnel automatique sans action de l'utilisateur             |
| IPv6 obligatoire         | Technologies de transition : 6to4, Teredo, IP-HTTPS         |
| IP-HTTPS                 | Methode de repli la plus fiable (TCP 443)                   |
| Windows Enterprise       | Les editions Pro ne sont pas supportees                     |
| NLS                      | Serveur critique pour la detection de localisation           |
| Deprecie                 | Remplace par Always On VPN pour les nouveaux deployements    |

---

## Pour aller plus loin

- Decouvrir le remplacement de DirectAccess : voir la page [Always On VPN](always-on-vpn.md)
- Configurer un serveur VPN classique : voir la page [Serveur VPN (RRAS)](vpn-server.md)
- Centraliser l'authentification : voir la page [NPS / RADIUS](nps-radius.md)
