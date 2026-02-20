---
title: "Extensions Windows Admin Center"
description: Installer et gerer les extensions Windows Admin Center - catalogue de feeds, extensions populaires et mise a jour.
tags:
  - gestion-moderne
  - wac
  - debutant
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

# Extensions Windows Admin Center

<span class="level-beginner">Debutant</span> Â· Temps estime : 15 minutes

## Presentation

Windows Admin Center est modulaire : ses fonctionnalites sont etendues via des **extensions**. Microsoft et des partenaires publient regulierement de nouvelles extensions pour ajouter des outils de gestion, des integrations cloud et des fonctionnalites de surveillance.

!!! example "Analogie"

    Les extensions WAC fonctionnent comme les applications d'un telephone. Le telephone de base permet de passer des appels, mais c'est l'ajout d'applications (GPS, messagerie, banque) qui lui donne toute sa valeur. WAC est le telephone, les extensions sont les applications.

## Types d'extensions

| Type | Description | Exemple |
|------|-------------|---------|
| **Tool** | Ajoute un outil dans le volet gauche d'une connexion serveur | Active Directory, DNS Manager |
| **Solution** | Experience de gestion autonome (page dediee) | Azure Hybrid Services |
| **Gateway Plugin** | Fonctionnalite au niveau de la passerelle WAC | Audit des connexions |

## Gerer les extensions

### Acceder au gestionnaire d'extensions

1. Ouvrir WAC dans le navigateur
2. Cliquer sur l'icone **Parametres** (engrenage) en haut a droite
3. Naviguer vers **Extensions**
4. La liste des extensions disponibles et installees s'affiche

### Installer une extension

1. Dans **Extensions** > onglet **Extensions disponibles**
2. Rechercher l'extension souhaitee
3. Cliquer sur **Installer**
4. L'extension est telechargee et activee automatiquement

### Mettre a jour les extensions

1. Dans **Extensions** > onglet **Extensions installees**
2. Les extensions avec une mise a jour disponible affichent un indicateur
3. Cliquer sur **Mettre a jour** pour chaque extension

!!! tip "Mise a jour automatique"

    Activez la mise a jour automatique des extensions dans **Parametres** > **Extensions** >
    **Mettre a jour automatiquement les extensions**. Cela garantit que vous disposez
    toujours des dernieres fonctionnalites et corrections.

### Desinstaller une extension

1. Dans **Extensions** > onglet **Extensions installees**
2. Selectionner l'extension
3. Cliquer sur **Desinstaller**

## Extensions populaires

### Extensions Microsoft

| Extension | Description |
|-----------|-------------|
| **Active Directory** | Gestion des utilisateurs, groupes, OU directement dans WAC |
| **DNS** | Gestion des zones et enregistrements DNS |
| **DHCP** | Gestion des etendues et reservations DHCP |
| **GPU Management** | Gestion des GPU (DDA, GPU-P) pour Hyper-V |
| **Azure Hybrid Services** | Integration Azure Arc, Azure Backup, Azure Monitor |
| **Azure File Sync** | Configuration de la synchronisation de fichiers Azure |
| **Security** | Recommandations de securite et conformite |
| **Storage Migration Service** | Migration de donnees entre serveurs |
| **Storage Replica** | Configuration de la replication du stockage |
| **System Insights** | Analyses predictives basees sur l'apprentissage automatique |
| **Remote Desktop** | Connexion RDP directement dans le navigateur |

### Extensions partenaires

Des editeurs tiers proposent egalement des extensions :

| Extension | Editeur | Description |
|-----------|---------|-------------|
| **DataON MUST** | DataON | Gestion du stockage HCI |
| **Fujitsu ServerView** | Fujitsu | Gestion materielle des serveurs Fujitsu |
| **Lenovo XClarity** | Lenovo | Integration avec le management Lenovo |

## Configurer les feeds d'extensions

WAC utilise des **feeds NuGet** pour distribuer les extensions.

### Feed par defaut

Le feed officiel Microsoft est configure par defaut et propose les extensions Microsoft et partenaires certifiees.

### Ajouter un feed personnalise

Pour les extensions internes ou de test :

1. **Parametres** > **Extensions** > **Feeds**
2. Cliquer sur **Ajouter**
3. Saisir l'URL du feed NuGet
4. Cliquer sur **Ajouter**

```
# Example NuGet feed URL format
https://mycompany.pkgs.visualstudio.com/_packaging/wac-extensions/nuget/v3/index.json
```

!!! warning "Securite"

    N'ajoutez que des feeds de confiance. Les extensions ont acces complet aux serveurs
    geres via WAC. Verifiez toujours l'editeur et la source des extensions.

## Extension PowerShell

WAC expose une API PowerShell pour la gestion des extensions :

```powershell
# List installed extensions (via WAC API)
# Note: these commands must be run in the WAC PowerShell console or via the API

# Check extension status via the WAC settings page
# Settings > Extensions > Installed Extensions
```

Resultat :

```text
# Via Parametres > Extensions > Extensions installees dans l'interface WAC :

Nom                         Version   Statut
--------------------------  --------  ------------------
Active Directory            2.3.1     Installe
DNS                         2.1.4     Installe
DHCP                        1.9.2     Installe
Remote Desktop              1.5.0     Installe - Mise a jour disponible (1.6.1)
Azure Hybrid Services       3.2.0     Installe
Security                    2.0.5     Installe
```

## Developpement d'extensions personnalisees

Le SDK Windows Admin Center permet de creer des extensions personnalisees :

| Composant | Technologie |
|-----------|-------------|
| Frontend | Angular + TypeScript |
| Backend | PowerShell ou C# (.NET) |
| Distribution | Package NuGet |
| Documentation | Microsoft Learn - WAC SDK |

Les cas d'usage pour des extensions personnalisees :

- Tableaux de bord specifiques a l'entreprise
- Integration avec des outils internes
- Automatisation de taches recurrentes
- Gestion de logiciels tiers installes sur les serveurs

!!! example "Scenario pratique"

    **Context :** Laura, administratrice systeme, vient d'installer WAC. Elle souhaite pouvoir gerer Active Directory directement depuis WAC sans ouvrir ADUC sur son poste.

    **Etape 1 : Acceder au gestionnaire d'extensions**

    Laura ouvre `https://SRV-01.lab.local`, clique sur l'engrenage en haut a droite, puis sur **Extensions**.

    **Etape 2 : Installer l'extension Active Directory**

    Elle clique sur l'onglet **Extensions disponibles**, recherche "Active Directory", puis clique sur **Installer**. L'extension se telecharge et s'active automatiquement (30 secondes environ).

    **Etape 3 : Installer Remote Desktop pour administrer les serveurs Core**

    Toujours dans les extensions disponibles, Laura installe egalement **Remote Desktop**. Elle peut maintenant ouvrir une session RDP directement dans le navigateur sans client RDP installe sur son poste.

    **Etape 4 : Activer les mises a jour automatiques**

    Dans **Parametres** > **Extensions**, elle coche **Mettre a jour automatiquement les extensions**. Desormais, les nouvelles versions s'installent sans intervention manuelle.

    **Resultat :** Laura gere maintenant les utilisateurs AD, les GPO, le DNS et le DHCP entierement depuis WAC. Le nombre d'outils ouverts sur son poste a ete divise par cinq.

!!! danger "Erreurs courantes"

    **Installer des extensions depuis des feeds NuGet non verifies.** Les extensions ont acces complet aux serveurs geres. Une extension malveillante pourrait executer du code sur tous les serveurs administres via WAC. Utiliser uniquement le feed officiel Microsoft ou des sources internes validees.

    **Ne pas mettre a jour les extensions.** Les anciennes versions d'extensions peuvent contenir des vulnerabilites ou des bugs de compatibilite avec les nouvelles versions de WAC. Activer les mises a jour automatiques ou verifier manuellement chaque mois.

    **Confondre extension "Tool" et extension "Solution".** Une extension Tool apparait dans le volet gauche lors de la connexion a un serveur. Une extension Solution est une page autonome accessible depuis la page d'accueil de WAC. Si une extension n'apparait pas la ou on l'attend, verifier son type.

    **Desinstaller une extension en cours d'utilisation.** Si d'autres administrateurs utilisent WAC au meme moment, la desinstallation d'une extension peut provoquer des erreurs dans leurs sessions actives. Communiquer avant toute modification des extensions en production.

## Points cles a retenir

- Les extensions etendent les fonctionnalites de WAC avec de nouveaux outils et integrations
- Les extensions **Active Directory**, **DNS** et **DHCP** sont essentielles pour la gestion quotidienne
- L'extension **Azure Hybrid Services** permet l'integration avec les services cloud Azure
- Les mises a jour automatiques garantissent l'acces aux dernieres fonctionnalites
- N'installer que des extensions provenant de sources de confiance
- Le SDK WAC permet de developper des extensions personnalisees en Angular + TypeScript

## Pour aller plus loin

- [Installation de WAC](installation.md) pour la mise en place initiale
- [Gestion des serveurs via WAC](gestion-serveurs.md) pour decouvrir les outils integres
- [Azure Arc](../azure-hybrid/azure-arc.md) pour l'integration cloud hybride

