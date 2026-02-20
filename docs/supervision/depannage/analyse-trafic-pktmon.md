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
title: Analyse de trafic avec Pktmon
description: Utiliser Pktmon, le sniffer de paquets natif de Windows Server 2022, pour diagnostiquer les problemes reseau sans Wireshark.
tags:
  - reseau
  - depannage
  - pktmon
  - wireshark
---

# Analyse de trafic avec Pktmon

<span class="level-advanced">Avance</span> · Temps estime : 30 minutes

## Pourquoi Pktmon ?

Vous avez un probleme reseau sur un serveur de production. Vous voulez voir si les paquets arrivent.
*   **Probleme :** Vous n'avez pas le droit d'installer Wireshark.
*   **Solution :** Utilisez **Pktmon** (Packet Monitor), integre nativement dans Windows Server 2019/2022 et Windows 10/11.

## Scenario : "Le Ping ne passe pas"

Vous essayez de pinguer `SRV-01` (10.0.0.20) depuis `DC-01` (10.0.0.10). Ca echoue. Est-ce le pare-feu ? Le routage ? Le paquet part-il ?

## Etape 1 : Definir un filtre

Pktmon capture TOUT par defaut. Il faut filtrer pour eviter de noyer le disque.

```powershell
# Supprimer les anciens filtres
pktmon filter remove

# Filtrer uniquement le trafic ICMP (Ping) ou le port 80
pktmon filter add -p 80
pktmon filter add -t icmp
```

## Etape 2 : Lancer la capture

On demarre l'enregistrement en temps reel.

```powershell
# Demarrer la capture sur toutes les interfaces
# --etw : Event Tracing for Windows (format natif)
# -l real-time : Voir les paquets defiler a l'ecran (Windows Server 2022 seulement)
pktmon start --etw -c
```

*Sur les versions plus anciennes de Windows, retirez l'option `-l real-time`.*

## Etape 3 : Reproduire le probleme

Lancez votre ping depuis une autre fenetre :
```powershell
ping 10.0.0.20
```

## Etape 4 : Arreter et convertir

```powershell
pktmon stop
```

Cela cree un fichier `PktMon.etl`. Ce format n'est pas lisible par les humains. Convertissons-le en texte (.txt) ou en format Wireshark (.pcapng).

```powershell
# Conversion en texte lisible
pktmon format PktMon.etl -o capture.txt

# Conversion pour Wireshark (Top !)
pktmon pcapng PktMon.etl -o capture.pcapng
```

## Analyse des resultats

Ouvrez `capture.txt`.

### Cas 1 : Paquet droppe par le Pare-feu
Vous verrez une ligne indiquant "Drop" avec une raison.
```text
Action: Drop
Reason: PolicyFrag
Component: WFP Native MAC Layer LightWeight Filter
```
*Signification : Le pare-feu Windows (WFP) a bloque le paquet.*

### Cas 2 : Paquet recu mais pas de reponse
Vous voyez le paquet "Echo Request" entrer, mais aucun "Echo Reply" sortir.
*Signification : Le serveur a recu la demande, mais l'application ou l'OS n'a pas repondu.*

## Nettoyage

N'oubliez jamais de supprimer les filtres et le fichier de capture.

```powershell
pktmon filter remove
Remove-Item PktMon.etl, capture.txt, capture.pcapng
```
