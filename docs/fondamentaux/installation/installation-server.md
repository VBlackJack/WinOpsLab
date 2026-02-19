---
title: Installation de Windows Server
description: Guide pas a pas pour installer Windows Server 2022.
tags:
  - fondamentaux
  - installation
  - debutant
---

# Installation de Windows Server

!!! info "Niveau : Debutant"

    Temps estime : 30 minutes | Lab associe : [Lab 01](../../labs/exercices/lab-01-installation.md)

## Introduction

L'installation de Windows Server 2022 est la premiere etape pour mettre en place une infrastructure serveur. Cette page couvre les differentes methodes d'installation et les choix a effectuer.

## Prerequis

- ISO Windows Server 2022 (telechargeable depuis le centre d'evaluation Microsoft)
- Machine virtuelle Hyper-V ou poste physique
- Minimum 512 Mo de RAM (2 Go recommande)
- Minimum 32 Go d'espace disque

## Methodes d'installation

=== "Installation interactive"

    L'installation interactive est la methode standard via l'assistant graphique.

    ### Etape 1 : Demarrer depuis l'ISO

    Configurez le BIOS/UEFI ou la VM pour demarrer sur l'ISO.

    ### Etape 2 : Choix de la langue

    Selectionnez la langue, le format horaire et la disposition du clavier.

    ### Etape 3 : Type d'installation

    Choisissez entre :

    - **Desktop Experience** : installation avec interface graphique complete
    - **Server Core** : installation minimale en ligne de commande

    !!! tip "Astuce"

        Server Core est recommande en production pour reduire la surface d'attaque
        et les besoins en mises a jour.

    ### Etape 4 : Partitionnement

    Selectionnez le disque et la partition cible. Pour un lab, acceptez les valeurs par defaut.

    ### Etape 5 : Configuration post-installation

    Apres le redemarrage, definissez le mot de passe administrateur.

=== "Installation automatisee"

    L'installation automatisee utilise un fichier `autounattend.xml`.

    ```xml
    <!-- Structure de base d'un fichier autounattend.xml -->
    <!-- Voir la documentation Microsoft pour les details -->
    ```

    !!! warning "Attention"

        La creation d'un fichier de reponse est un sujet avance.
        Voir la section [Automatisation](../../automatisation/index.md).

## Verification post-installation

```powershell
# Verify Windows Server version
Get-ComputerInfo | Select-Object WindowsProductName, OsVersion, OsBuildNumber

# Verify network configuration
Get-NetIPConfiguration

# Verify hostname
hostname
```

## Points cles a retenir

- Choisir entre Desktop Experience et Server Core selon l'usage
- Toujours definir un mot de passe administrateur fort
- Configurer le nom du serveur et l'adresse IP statique apres installation
- Installer les dernieres mises a jour Windows Update

## Pour aller plus loin

- [Configuration initiale](configuration-initiale.md) - etapes post-installation
- [Server Core vs GUI](server-core-vs-gui.md) - comparatif des deux modes
- [Lab 01 : Installation](../../labs/exercices/lab-01-installation.md) - exercice pratique
