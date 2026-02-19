---
title: "Adressage IPv4"
description: "Comprendre l'adressage IPv4 : classes, plages privees RFC 1918, notation CIDR, masques de sous-reseau et conversions binaire/decimal."
tags:
  - reseau
  - ipv4
  - tcp-ip
  - adressage
---

# Adressage IPv4

## Introduction

L'adressage IPv4 (Internet Protocol version 4) constitue le socle fondamental de toute communication reseau TCP/IP. Chaque equipement connecte a un reseau IP possede au moins une adresse IPv4, un identifiant numerique unique sur 32 bits qui permet l'acheminement des paquets entre source et destination.

!!! info "Contexte Windows Server 2022"

    Malgre l'adoption progressive d'IPv6, IPv4 reste le protocole dominant dans les infrastructures Windows Server. La maitrise de l'adressage IPv4 est indispensable pour configurer les interfaces reseau, le DHCP, le DNS et les regles de pare-feu.

---

## Structure d'une adresse IPv4

Une adresse IPv4 est composee de **32 bits** (4 octets), representee en notation decimale pointee :

```
192.168.1.10
```

Chaque octet peut prendre une valeur de **0 a 255** (soit 2^8 possibilites).

### Decomposition binaire

| Octet       | Decimal | Binaire    |
|-------------|---------|------------|
| 1er octet   | 192     | `11000000` |
| 2eme octet  | 168     | `10101000` |
| 3eme octet  | 1       | `00000001` |
| 4eme octet  | 10      | `00001010` |

L'adresse complete en binaire :

```
11000000.10101000.00000001.00001010
```

### Conversion decimal vers binaire

Pour convertir un octet decimal en binaire, on utilise la methode des divisions successives par 2 ou la methode des puissances de 2 :

| Position | 7   | 6  | 5  | 4  | 3 | 2 | 1 | 0 |
|----------|-----|----|----|----|---|---|---|---|
| Valeur   | 128 | 64 | 32 | 16 | 8 | 4 | 2 | 1 |

!!! tip "Methode rapide"

    Pour convertir 192 en binaire : 192 = 128 + 64 = `11000000`. On "allume" les bits correspondant aux valeurs dont la somme donne le nombre cible.

---

## Partie reseau et partie hote

Toute adresse IPv4 se decompose en deux parties :

- **Partie reseau (Network ID)** : identifie le reseau auquel appartient l'hote
- **Partie hote (Host ID)** : identifie un equipement specifique au sein de ce reseau

Le **masque de sous-reseau** determine la frontiere entre ces deux parties.

```mermaid
graph LR
    A["Adresse IP : 192.168.1.10"] --> B["Partie reseau"]
    A --> C["Partie hote"]
    B --> D["192.168.1.0<br/>definie par le masque"]
    C --> E["0.0.0.10<br/>identifiant unique"]
```

---

## Classes d'adresses (historique)

Le systeme de classes, defini a l'origine d'IPv4, repartit les adresses en cinq categories selon les premiers bits de l'adresse :

| Classe | Premiers bits | Plage du 1er octet | Masque par defaut | Nb reseaux | Nb hotes/reseau |
|--------|---------------|---------------------|-------------------|------------|-----------------|
| A      | `0`           | 1 - 126             | 255.0.0.0 (/8)    | 126        | ~16,7 millions  |
| B      | `10`          | 128 - 191           | 255.255.0.0 (/16) | 16 384     | 65 534          |
| C      | `110`         | 192 - 223           | 255.255.255.0 (/24) | ~2,1 millions | 254        |
| D      | `1110`        | 224 - 239           | Multicast          | -          | -               |
| E      | `1111`        | 240 - 255           | Experimental       | -          | -               |

!!! warning "Approche obsolete"

    Le systeme de classes est considere comme obsolete depuis l'adoption du CIDR (Classless Inter-Domain Routing) en 1993. Neanmoins, il reste frequemment mentionne dans les examens de certification et permet de comprendre l'historique de l'adressage IP.

### Adresses speciales

| Adresse              | Utilisation                                      |
|----------------------|--------------------------------------------------|
| `0.0.0.0`           | Route par defaut / adresse non definie           |
| `127.0.0.0/8`       | Loopback (boucle locale, typiquement 127.0.0.1)  |
| `169.254.0.0/16`    | APIPA (adresses automatiques sans DHCP)          |
| `255.255.255.255`   | Broadcast limite (diffusion sur le reseau local) |

---

## Plages d'adresses privees (RFC 1918)

La RFC 1918 definit trois plages d'adresses reservees aux reseaux prives, non routables sur Internet :

| Classe | Plage                         | Notation CIDR    | Nombre d'adresses |
|--------|-------------------------------|------------------|--------------------|
| A      | 10.0.0.0 - 10.255.255.255    | 10.0.0.0/8       | 16 777 216         |
| B      | 172.16.0.0 - 172.31.255.255  | 172.16.0.0/12    | 1 048 576          |
| C      | 192.168.0.0 - 192.168.255.255| 192.168.0.0/16   | 65 536             |

!!! info "Utilisation en entreprise"

    En environnement Windows Server, on utilise quasi exclusivement des adresses privees RFC 1918. Le reseau 10.0.0.0/8 est privilegie dans les grandes entreprises car il offre une grande flexibilite de decoupe en sous-reseaux. Le reseau 192.168.x.0/24 est plus courant dans les PME et les labs.

### Verifier l'adresse IP sous Windows Server

```powershell
# Display all IP addresses configured on the server
Get-NetIPAddress -AddressFamily IPv4

# Display configuration details of a specific interface
Get-NetIPConfiguration -InterfaceAlias "Ethernet0"
```

---

## Notation CIDR

La notation **CIDR** (Classless Inter-Domain Routing), definie dans la RFC 4632, remplace le systeme de classes et permet un decoupage plus flexible de l'espace d'adressage.

La notation CIDR ajoute un suffixe `/n` a l'adresse, ou `n` represente le nombre de bits a 1 dans le masque de sous-reseau.

### Correspondance masque / CIDR

| Notation CIDR | Masque              | Bits reseau | Bits hote | Nb hotes utilisables |
|---------------|---------------------|-------------|-----------|----------------------|
| /8            | 255.0.0.0           | 8           | 24        | 16 777 214           |
| /16           | 255.255.0.0         | 16          | 16        | 65 534               |
| /24           | 255.255.255.0       | 24          | 8         | 254                  |
| /25           | 255.255.255.128     | 25          | 7         | 126                  |
| /26           | 255.255.255.192     | 26          | 6         | 62                   |
| /27           | 255.255.255.224     | 27          | 5         | 30                   |
| /28           | 255.255.255.240     | 28          | 4         | 14                   |
| /29           | 255.255.255.248     | 29          | 3         | 6                    |
| /30           | 255.255.255.252     | 30          | 2         | 2                    |
| /31           | 255.255.255.254     | 31          | 1         | 2 (point a point)    |
| /32           | 255.255.255.255     | 32          | 0         | 1 (hote unique)      |

!!! tip "Formule rapide"

    Nombre d'hotes utilisables = 2^(32 - n) - 2, ou `n` est le prefixe CIDR. On soustrait 2 pour l'adresse reseau et l'adresse de broadcast.

---

## Le masque de sous-reseau en detail

Le masque de sous-reseau est une suite de 32 bits composee d'une serie ininterrompue de **1** (partie reseau) suivie d'une serie ininterrompue de **0** (partie hote).

### Operation AND logique

Pour determiner l'adresse reseau a partir d'une adresse IP et de son masque, on applique un **AND logique** bit a bit :

```
Adresse IP :   192.168.1.10    = 11000000.10101000.00000001.00001010
Masque :       255.255.255.0   = 11111111.11111111.11111111.00000000
               ─────────────────────────────────────────────────────
Adresse reseau: 192.168.1.0    = 11000000.10101000.00000001.00000000
```

### Adresse de broadcast

L'adresse de broadcast est obtenue en mettant tous les bits de la partie hote a **1** :

```
Adresse reseau: 192.168.1.0    = 11000000.10101000.00000001.00000000
Broadcast :     192.168.1.255  = 11000000.10101000.00000001.11111111
```

### Verifier le masque avec PowerShell

```powershell
# Display prefix length (CIDR notation) for all IPv4 addresses
Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength

# Calculate network address using .NET
$ip = [System.Net.IPAddress]::Parse("192.168.1.10")
$mask = [System.Net.IPAddress]::Parse("255.255.255.0")
$networkBytes = @()
for ($i = 0; $i -lt 4; $i++) {
    $networkBytes += $ip.GetAddressBytes()[$i] -band $mask.GetAddressBytes()[$i]
}
$networkAddress = [System.Net.IPAddress]::new($networkBytes)
Write-Output "Network address: $networkAddress"
```

---

## APIPA (Automatic Private IP Addressing)

Lorsqu'un client Windows ne parvient pas a contacter un serveur DHCP, il s'attribue automatiquement une adresse dans la plage **169.254.0.0/16** (APIPA). Ce mecanisme permet une connectivite locale minimale mais ne fournit ni passerelle, ni DNS.

```powershell
# Check if an interface is using an APIPA address
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {
    $_.IPAddress -like "169.254.*"
} | Select-Object InterfaceAlias, IPAddress
```

!!! danger "Diagnostic"

    Une adresse APIPA sur un serveur indique un probleme de communication DHCP. Verifiez la connectivite physique, le VLAN et la disponibilite du serveur DHCP.

---

## Points cles a retenir

| Concept                 | Detail                                                        |
|-------------------------|---------------------------------------------------------------|
| Taille adresse IPv4     | 32 bits (4 octets), notation decimale pointee                 |
| Masque de sous-reseau   | Separe la partie reseau de la partie hote                     |
| RFC 1918                | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16                   |
| CIDR                    | Notation /n remplacant le systeme de classes                  |
| APIPA                   | 169.254.0.0/16 en cas d'echec DHCP                           |
| Nombre d'hotes          | 2^(32-n) - 2 pour un prefixe /n                              |

---

## Pour aller plus loin

- Pratiquer le calcul de sous-reseaux avec des exercices concrets : voir la page [Sous-reseaux](sous-reseaux.md)
- Comprendre les fondamentaux d'IPv6 : voir la page [IPv6 Fondamentaux](ipv6-fondamentaux.md)
- Configurer les interfaces reseau sur Windows Server : voir la page [Configuration des interfaces](configuration-interfaces.md)
