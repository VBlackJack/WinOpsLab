---
title: Active Directory
description: Services d'annuaire Active Directory - AD DS, DNS, DHCP et strategies de groupe.
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

# Active Directory

!!! abstract "Objectifs du module"

    A la fin de ce module, vous serez capable de :

    - [ ] Deployer et gerer un domaine Active Directory
    - [ ] Configurer les services DNS integres a AD
    - [ ] Mettre en place un serveur DHCP
    - [ ] Creer et appliquer des strategies de groupe (GPO)

## Contenu

<div class="grid cards" markdown>

- :material-folder-account: **AD DS**

    ---

    Concepts fondamentaux, deploiement du premier DC, OU, utilisateurs, groupes, sites et replication.

    [:octicons-arrow-right-24: Commencer](adds/index.md)

- :material-dns: **DNS**

    ---

    Zones integrees AD, enregistrements, resolution conditionnelle, depannage.

    [:octicons-arrow-right-24: Decouvrir](dns/index.md)

- :material-ip-network: **DHCP**

    ---

    Installation, etendues, options, reservations et basculement.

    [:octicons-arrow-right-24: Configurer](dhcp/index.md)

- :material-shield-lock: **Strategie de groupe (GPO)**

    ---

    Creation, liaison, filtrage, heritage, preferences, depannage.

    [:octicons-arrow-right-24: Appliquer](gpo/index.md)

- :material-shield-key: **AD FS**

    ---

    Authentification federee, Single Sign-On et federation d'identite avec AD FS.

    [:octicons-arrow-right-24: Decouvrir](adfs/index.md)

</div>

## Prerequis

- Module [Fondamentaux](../fondamentaux/index.md) complete
- Un controleur de domaine installe (voir [Lab 02](../labs/exercices/lab-02-ad-ds-premier-domaine.md))

