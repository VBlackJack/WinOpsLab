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
title: MMC et snap-ins
description: Microsoft Management Console et les snap-ins d'administration Windows Server.
tags:
  - fondamentaux
  - console
  - debutant
---

# MMC et snap-ins

<span class="level-beginner">Debutant</span> Â· Temps estime : 10 minutes

## Qu'est-ce que la MMC ?

!!! example "Analogie"

    Pensez a la MMC comme un classeur a anneaux vide. Chaque **snap-in** est un intercalaire
    specialise : un pour la gestion des employes (Active Directory), un pour l'annuaire telephonique
    (DNS), un pour la comptabilite (certificats), etc. Vous composez votre propre classeur selon
    vos besoins, et vous pouvez meme ajouter des intercalaires qui consultent les dossiers
    d'un autre bureau (serveur distant).

La **Microsoft Management Console** (MMC) est un framework qui heberge des outils d'administration appeles **snap-ins**. Chaque snap-in fournit une interface graphique pour gerer un aspect specifique du serveur.

## Lancer la MMC

```powershell
# Open an empty MMC console
mmc

# Open a specific pre-configured console
dnsmgmt.msc       # DNS Manager
dhcpmgmt.msc      # DHCP Manager
dsa.msc           # Active Directory Users and Computers
dssite.msc        # Active Directory Sites and Services
gpmc.msc          # Group Policy Management Console
compmgmt.msc      # Computer Management
diskmgmt.msc      # Disk Management
devmgmt.msc       # Device Manager
services.msc      # Services
eventvwr.msc      # Event Viewer
certlm.msc        # Certificates (Local Machine)
wf.msc            # Windows Firewall with Advanced Security
fsmgmt.msc        # Shared Folders
lusrmgr.msc       # Local Users and Groups
```

Vous pouvez egalement lister les fichiers `.msc` disponibles sur le systeme :

```powershell
# List all available .msc files
Get-ChildItem -Path "$env:SystemRoot\System32" -Filter "*.msc" | Select-Object Name, Length
```

Resultat (extrait) :

```text
Name              Length
----              ------
certlm.msc         2254
compmgmt.msc        2330
devmgmt.msc         1460
dhcpmgmt.msc        1672
diskmgmt.msc        1854
dnsmgmt.msc         1502
dsa.msc             1438
dssite.msc          1464
eventvwr.msc        1580
fsmgmt.msc          1512
gpmc.msc            1398
lusrmgr.msc         1468
perfmon.msc         1790
services.msc        1584
wf.msc              2104
```

## Snap-ins courants par role

| Snap-in (.msc) | Description | Role requis |
|----------------|-------------|-------------|
| `dsa.msc` | Utilisateurs et ordinateurs AD | AD DS |
| `dssite.msc` | Sites et services AD | AD DS |
| `dnsmgmt.msc` | Gestionnaire DNS | DNS |
| `dhcpmgmt.msc` | Gestionnaire DHCP | DHCP |
| `gpmc.msc` | Gestion des GPO | AD DS (GPMC) |
| `certsrv.msc` | Autorite de certification | AD CS |
| `wf.msc` | Pare-feu avance | Aucun |
| `diskmgmt.msc` | Gestion des disques | Aucun |
| `eventvwr.msc` | Observateur d'evenements | Aucun |
| `services.msc` | Services | Aucun |
| `compmgmt.msc` | Gestion de l'ordinateur | Aucun |
| `perfmon.msc` | Moniteur de performances | Aucun |

## Creer une console MMC personnalisee

Vous pouvez creer votre propre console regroupant plusieurs snap-ins :

1. Lancer `mmc` (console vide)
2. **File** > **Add/Remove Snap-in** (++ctrl+m++)
3. Selectionner les snap-ins souhaites dans la liste
4. Pour les snap-ins distants, choisir **Another computer** et saisir le nom du serveur
5. Valider avec **OK**
6. **File** > **Save As** pour sauvegarder la console (`.msc`)

!!! tip "Console d'administration centralisee"

    Creez une console MMC personnalisee sur votre poste d'administration avec les snap-ins
    connectes a vos serveurs distants. Vous pourrez gerer plusieurs serveurs depuis une seule fenetre.

## Connexion a un serveur distant

La plupart des snap-ins permettent de se connecter a un serveur distant :

1. Clic droit sur la racine du snap-in
2. **Connect to another computer**
3. Saisir le nom ou l'IP du serveur distant

!!! warning "Prerequis reseau"

    La gestion a distance necessite :

    - WinRM active sur le serveur cible
    - Pare-feu ouvert pour la gestion a distance
    - Droits d'administration sur le serveur cible

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Nathalie, technicienne de support, administre 4 serveurs depuis son poste Windows 11 :
    DC-01 (Active Directory), DC-02 (DNS secondaire), FS-01 (fichiers) et SRV-01 (IIS). Elle perd du
    temps a ouvrir des sessions RDP sur chaque serveur pour gerer les services.

    **Probleme** : La gestion multi-serveurs par RDP est lente et desorganisee. Elle a parfois 4 fenetres
    RDP ouvertes simultanement.

    **Solution** : Creer une console MMC personnalisee centralisee.

    1. Ouvrir une console vide : `mmc`
    2. **File** > **Add/Remove Snap-in** (++ctrl+m++)
    3. Ajouter les snap-ins suivants en ciblant les serveurs distants :

    | Snap-in | Serveur cible |
    |---------|---------------|
    | Active Directory Users and Computers | DC-01 |
    | DNS | DC-01 |
    | DNS | DC-02 |
    | Shared Folders | FS-01 |
    | Services | SRV-01 |
    | Event Viewer | DC-01 |

    4. **File** > **Save As** > `C:\Admin\MaConsole.msc`

    **Verification** :

    ```powershell
    # Verify connectivity to all servers before opening the console
    "DC-01", "DC-02", "FS-01", "SRV-01" | ForEach-Object {
        Test-WSMan -ComputerName $_ | Select-Object @{N='Server';E={$_.'wsmid'}}, ProductVersion
    }
    ```

    ```text
    Server                                                                ProductVersion
    ------                                                                --------------
    http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd      OS: 10.0.20348 SP: 0.0 Stack: 3.0
    http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd      OS: 10.0.20348 SP: 0.0 Stack: 3.0
    http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd      OS: 10.0.20348 SP: 0.0 Stack: 3.0
    http://schemas.dmtf.org/wbem/wsman/identity/1/wsmanidentity.xsd      OS: 10.0.20348 SP: 0.0 Stack: 3.0
    ```

    **Resultat** : Nathalie gere ses 4 serveurs depuis une seule fenetre. Plus besoin de RDP pour
    les taches courantes d'administration.

## Erreurs courantes

!!! danger "Erreurs courantes"

    1. **Lancer un snap-in sans le role correspondant installe** : Par exemple, ouvrir `dnsmgmt.msc`
       sur un serveur qui n'a pas le role DNS installe genere une erreur. Assurez-vous que le role
       ou les outils RSAT correspondants sont presents.

    2. **Oublier de sauvegarder la console personnalisee** : Apres avoir ajoute plusieurs snap-ins
       connectes a des serveurs distants, sauvegardez la console (`.msc`). Sinon, vous devrez
       tout reconfigurer a chaque ouverture.

    3. **Ne pas lancer la MMC en tant qu'administrateur** : Certains snap-ins necessitent des
       privileges eleves. Lancez toujours `mmc` avec **Executer en tant qu'administrateur** pour
       eviter les erreurs d'acces.

    4. **Confondre les snap-ins locaux et distants** : Lors de l'ajout d'un snap-in, verifiez bien
       si vous l'ajoutez pour la machine locale ou un serveur distant. Choisir "Local computer" par
       defaut quand vous vouliez cibler un serveur distant est une erreur frequente.

## Points cles a retenir

- La MMC est un conteneur pour les snap-ins d'administration
- Chaque snap-in gere un aspect specifique (DNS, DHCP, AD, disques, etc.)
- Les fichiers `.msc` sont des raccourcis vers des consoles pre-configurees
- Vous pouvez creer des consoles personnalisees multi-serveurs

## Pour aller plus loin

- [RSAT](rsat.md) - les memes snap-ins sur un poste client
- [Server Manager](server-manager.md) - vue d'ensemble centralisee

