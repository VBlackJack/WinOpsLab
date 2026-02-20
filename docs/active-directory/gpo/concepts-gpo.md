---
title: Concepts GPO
description: Comprendre les objets de strategie de groupe (GPO), leur ordre de traitement LSDOU, le stockage SYSVOL et les commandes essentielles.
tags:
  - gpo
  - active-directory
  - strategie-de-groupe
  - sysvol
---

# Concepts des strategies de groupe (GPO)

<span class="level-intermediate">Intermediaire</span> · Temps estime : 45 minutes

## Qu'est-ce qu'un objet de strategie de groupe (GPO) ?

Un **Group Policy Object (GPO)** est un ensemble de parametres de configuration geres
de facon centralisee dans Active Directory. Les GPO permettent aux administrateurs de
controler l'environnement de travail des utilisateurs et des ordinateurs du domaine
sans intervention manuelle sur chaque poste.

Chaque GPO contient deux grandes sections :

| Section                      | Cible                       | Exemples de parametres                          |
| :--------------------------- | :-------------------------- | :---------------------------------------------- |
| **Computer Configuration**   | Compte ordinateur           | Scripts de demarrage, politique de securite, pare-feu |
| **User Configuration**       | Compte utilisateur          | Redirection de dossiers, lecteurs mappes, bureau |

!!! tip "Bonne pratique"

    Desactivez la section inutilisee d'une GPO (Computer ou User) pour accelerer
    le traitement. Cela se fait dans les proprietes de la GPO, onglet **Details**,
    champ **GPO Status**.

---

## Ordre de traitement : LSDOU

Les GPO sont appliquees dans un ordre precis appele **LSDOU** :

1. **L**ocal -- la strategie de groupe locale de la machine (`gpedit.msc`)
2. **S**ite -- GPO liees au site Active Directory
3. **D**omaine -- GPO liees au domaine
4. **OU** -- GPO liees aux unites organisationnelles (de la plus haute a la plus basse)

Le dernier parametre applique **l'emporte** en cas de conflit. Ainsi, une GPO liee a
une OU enfant prevaut sur une GPO liee au domaine pour le meme parametre.

```mermaid
flowchart TD
    A["Strategie locale<br/>(gpedit.msc)"] --> B["GPO de site<br/>(Site AD)"]
    B --> C["GPO de domaine<br/>(ex: Default Domain Policy)"]
    C --> D["GPO d'OU parente<br/>(ex: OU Siege)"]
    D --> E["GPO d'OU enfante<br/>(ex: OU Comptabilite)"]

    style A fill:#455a64,color:#fff
    style B fill:#546e7a,color:#fff
    style C fill:#1565c0,color:#fff
    style D fill:#1976d2,color:#fff
    style E fill:#1e88e5,color:#fff
```

!!! warning "Exceptions a LSDOU"

    Deux mecanismes peuvent modifier cet ordre :

    - **Enforced (Applique)** : force une GPO a s'appliquer meme si une GPO de
      niveau inferieur definit le meme parametre.
    - **Block Inheritance** : empeche les GPO parentes de s'appliquer a une OU,
      sauf si elles sont marquees Enforced.

    Ces mecanismes sont detailles dans la page
    [Filtrage et heritage](filtrage-et-heritage.md).

---

## Computer Configuration vs User Configuration

### Computer Configuration

Les parametres de la section **Computer Configuration** sont appliques au
**demarrage de la machine** et rafraichis periodiquement (toutes les 90 minutes
par defaut, avec un decalage aleatoire de 0 a 30 minutes).

Exemples courants :

- Politique de mot de passe et de verrouillage de compte
- Regles du pare-feu Windows
- Configuration de Windows Update
- Scripts de demarrage/arret
- Parametres de securite (audit, droits utilisateur)

### User Configuration

Les parametres de la section **User Configuration** sont appliques a
**l'ouverture de session** de l'utilisateur et rafraichis periodiquement.

Exemples courants :

- Redirection de dossiers (Bureau, Documents)
- Lecteurs reseau mappes
- Restrictions du menu Demarrer et du bureau
- Configuration des applications (Office, navigateurs)
- Scripts d'ouverture/fermeture de session

!!! info "Cycle de rafraichissement"

    Le rafraichissement automatique a lieu toutes les **90 minutes** (+ decalage
    aleatoire de 0 a 30 min) pour les postes de travail et serveurs membres.
    Pour les **controleurs de domaine**, le cycle est de **5 minutes**.

---

## SYSVOL et stockage des GPO

Les GPO sont stockees a deux endroits :

1. **Active Directory** (conteneur `CN=Policies,CN=System,DC=domaine,DC=local`) :
   contient les **metadonnees** de la GPO (liens, filtres de securite, version).

2. **SYSVOL** (`\\domaine\SYSVOL\domaine\Policies\{GUID}`) : contient les
   **fichiers de parametres** (fichiers `.pol`, scripts, modeles ADMX).

Chaque GPO est identifiee par un **GUID** unique. Le dossier SYSVOL correspondant
contient :

```
{GUID-de-la-GPO}/
    Machine/                  # Computer Configuration settings
        Registry.pol          # Registry-based policy settings
        Scripts/              # Startup / Shutdown scripts
    User/                     # User Configuration settings
        Registry.pol          # Registry-based policy settings
        Scripts/              # Logon / Logoff scripts
    GPT.INI                   # Version information
```

!!! danger "SYSVOL et replication"

    Le dossier SYSVOL est replique entre tous les controleurs de domaine via
    **DFS-R** (ou l'ancien FRS). Un probleme de replication SYSVOL entraine
    des GPO incoherentes entre les DC. Verifiez regulierement l'etat de
    replication avec `dcdiag /test:sysvolcheck`.

=== "PowerShell"

    ```powershell
    # List all GPOs in the domain
    Get-GPO -All | Select-Object DisplayName, Id, GpoStatus, CreationTime |
        Format-Table -AutoSize

    # View the SYSVOL path of a specific GPO
    $gpoId = (Get-GPO -Name "Default Domain Policy").Id
    $sysvolPath = "\\$env:USERDNSDOMAIN\SYSVOL\$env:USERDNSDOMAIN\Policies\{$gpoId}"
    Get-ChildItem -Path $sysvolPath -Recurse
    ```

=== "GUI (gpmc.msc)"

    1. Ouvrir **Group Policy Management** (`gpmc.msc`)
    2. Developper **Forest** > **Domains** > votre domaine > **Group Policy Objects**
    3. Cliquer droit sur une GPO > **Properties** > onglet **Details**
    4. Le champ **Unique ID** correspond au GUID du dossier SYSVOL

---

## Commande gpupdate

La commande `gpupdate` permet de forcer le rafraichissement immediat des
strategies de groupe sur un poste, sans attendre le cycle automatique.

=== "PowerShell"

    ```powershell
    # Standard refresh -- applies only new and changed settings
    gpupdate

    # Force full reapplication of all settings
    gpupdate /force

    # Refresh only Computer Configuration
    gpupdate /target:computer

    # Refresh only User Configuration
    gpupdate /target:user

    # Force refresh on a remote computer (requires PSRemoting)
    Invoke-GPUpdate -Computer "PC-COMPTA-01" -Force -RandomDelayInMinutes 0
    ```

=== "GUI (gpmc.msc)"

    1. Dans **Group Policy Management**, cliquer droit sur une OU
    2. Selectionner **Group Policy Update...**
    3. Confirmer pour forcer le rafraichissement sur tous les ordinateurs de l'OU

!!! tip "gpupdate vs gpupdate /force"

    - `gpupdate` : applique uniquement les parametres **nouveaux ou modifies**
      (plus rapide, recommande).
    - `gpupdate /force` : reapplique **tous** les parametres, y compris ceux
      deja en place. Utile pour le depannage, mais plus lourd.

---

## Les GPO par defaut

A la creation du domaine, deux GPO sont automatiquement creees :

| GPO                            | Liee a   | Role                                                    |
| :----------------------------- | :------- | :------------------------------------------------------ |
| **Default Domain Policy**      | Domaine  | Politique de mot de passe, verrouillage de compte, Kerberos |
| **Default Domain Controllers Policy** | OU Domain Controllers | Parametres de securite specifiques aux DC |

!!! warning "Ne pas modifier excessivement les GPO par defaut"

    Il est recommande de ne modifier les GPO par defaut **que** pour les
    parametres de politique de mot de passe et de verrouillage de compte.
    Pour tous les autres besoins, creez de **nouvelles GPO** dediees.
    Cela facilite le depannage et evite les effets de bord.

---

## Points cles a retenir

- Un **GPO** est un conteneur de parametres appliques de facon centralisee aux
  utilisateurs et/ou ordinateurs du domaine.
- L'ordre de traitement **LSDOU** (Local > Site > Domain > OU) determine quelle
  GPO l'emporte en cas de conflit : la derniere appliquee gagne.
- Les GPO contiennent deux sections independantes : **Computer Configuration**
  (appliquee au demarrage) et **User Configuration** (appliquee a l'ouverture de session).
- Les donnees des GPO sont stockees dans **Active Directory** (metadonnees) et
  **SYSVOL** (fichiers de parametres).
- La commande `gpupdate` force le rafraichissement immediat ; `gpupdate /force`
  reapplique tous les parametres.
- Ne modifiez les GPO par defaut que pour les politiques de mot de passe ; creez
  des GPO dediees pour tout le reste.

---

## Pour aller plus loin

- [Creer et lier une GPO](creer-et-lier.md) -- mise en pratique
- [Filtrage et heritage](filtrage-et-heritage.md) -- controle fin de l'application
- [GPResult et depannage](gpresult-et-depannage.md) -- verifier ce qui s'applique
- [Modeles ADMX](modeles-admx.md) -- etendre les parametres disponibles
- [Structure des OU](../adds/structure-ou.md) -- organiser les objets pour les GPO
- [Sites et replication](../adds/sites-et-replication.md) -- impact sur la replication SYSVOL
