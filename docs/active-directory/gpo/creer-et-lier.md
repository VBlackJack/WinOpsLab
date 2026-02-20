---
title: Creer et lier une GPO
description: Guide pratique pour creer, lier et configurer des GPO via la console GPMC et PowerShell sous Windows Server 2022.
tags:
  - gpo
  - active-directory
  - gpmc
  - powershell
---

# Creer et lier une GPO

<span class="level-intermediate">Intermediaire</span> · Temps estime : 50 minutes

## Cycle de vie d'une GPO

!!! example "Analogie"

    Creer et lier une GPO est comparable a rediger un reglement interieur puis l'afficher dans un service precis de l'entreprise. Tant que le document reste dans le tiroir du directeur (non lie), personne ne le lit. Il faut l'afficher au bon etage (lier a la bonne OU) pour qu'il prenne effet.

La gestion d'une GPO suit un cycle en quatre etapes :

```mermaid
flowchart LR
    A["1. Creer<br/>la GPO"] --> B["2. Configurer<br/>les parametres"]
    B --> C["3. Lier<br/>a un conteneur"]
    C --> D["4. Tester<br/>et valider"]
    D -->|Ajustements| B

    style A fill:#1565c0,color:#fff
    style B fill:#1976d2,color:#fff
    style C fill:#1e88e5,color:#fff
    style D fill:#42a5f5,color:#fff
```

!!! tip "Bonne pratique : creer avant de lier"

    Creez la GPO dans le conteneur **Group Policy Objects** et configurez-la
    entierement **avant** de la lier a une OU. Cela evite d'appliquer des
    parametres incomplets aux utilisateurs ou ordinateurs.

---

## Creer une nouvelle GPO

=== "PowerShell"

    ```powershell
    # Create a new GPO (not yet linked)
    New-GPO -Name "SEC - Password Policy" -Comment "Enforce strong password requirements"

    # Create and immediately link to an OU
    New-GPO -Name "CFG - Mapped Drives" -Comment "Map network drives for all users" |
        New-GPLink -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local"
    ```

    Resultat :

    ```text
    DisplayName          : SEC - Password Policy
    DomainName           : lab.local
    Owner                : LAB\Domain Admins
    Id                   : d4e5f6a7-8b9c-0d1e-2f3a-4b5c6d7e8f9a
    GpoStatus            : AllSettingsEnabled
    Description          : Enforce strong password requirements
    CreationTime         : 20/02/2026 10:15:30
    ModificationTime     : 20/02/2026 10:15:30
    ```

=== "GUI (gpmc.msc)"

    1. Ouvrir **Group Policy Management** (`gpmc.msc`)
    2. Developper **Forest** > **Domains** > votre domaine
    3. Cliquer droit sur **Group Policy Objects** > **New**
    4. Saisir un nom descriptif et cliquer **OK**

!!! tip "Convention de nommage"

    Adoptez un prefixe pour identifier rapidement le type de GPO :

    | Prefixe | Usage                        | Exemple                         |
    | :------ | :--------------------------- | :------------------------------ |
    | `SEC`   | Securite                     | `SEC - Account Lockout Policy`  |
    | `CFG`   | Configuration poste/user     | `CFG - Desktop Restrictions`    |
    | `SW`    | Deploiement logiciel         | `SW - 7-Zip Deployment`         |
    | `SCR`   | Scripts                      | `SCR - Logon Drive Mapping`     |
    | `PRF`   | Preferences                  | `PRF - Printer Deployment`      |

---

## Lier une GPO a un conteneur

Une GPO doit etre **liee** a un ou plusieurs conteneurs pour prendre effet.
Les conteneurs valides sont : **Site**, **Domaine** ou **OU**.

=== "PowerShell"

    ```powershell
    # Link an existing GPO to an OU
    New-GPLink -Name "SEC - Password Policy" `
        -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local"

    # Link to the domain root
    New-GPLink -Name "SEC - Password Policy" `
        -Target "DC=lab,DC=local"

    # Link to a site
    New-GPLink -Name "CFG - Branch Office Settings" `
        -Target "CN=Lyon,CN=Sites,CN=Configuration,DC=lab,DC=local"

    # Disable a link (GPO remains but does not apply)
    Set-GPLink -Name "CFG - Mapped Drives" `
        -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local" `
        -LinkEnabled No

    # Remove a link (does not delete the GPO)
    Remove-GPLink -Name "CFG - Mapped Drives" `
        -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local"
    ```

    Resultat :

    ```text
    GpoId       : d4e5f6a7-8b9c-0d1e-2f3a-4b5c6d7e8f9a
    DisplayName : SEC - Password Policy
    Enabled     : True
    Enforced    : False
    Target      : OU=Utilisateurs,OU=Siege,DC=lab,DC=local
    Order       : 1
    ```

=== "GUI (gpmc.msc)"

    1. Dans **Group Policy Management**, cliquer droit sur l'OU cible
    2. Selectionner **Link an Existing GPO...**
    3. Choisir la GPO dans la liste et cliquer **OK**

    Pour **desactiver** un lien : cliquer droit sur le lien > decocher
    **Link Enabled**.

!!! warning "GPO liee vs GPO presente"

    Une GPO existe dans le conteneur **Group Policy Objects** meme sans lien.
    Supprimer un **lien** ne supprime pas la GPO. Supprimer la **GPO** elle-meme
    supprime aussi tous ses liens.

---

## Configurer les parametres d'une GPO

=== "PowerShell"

    ```powershell
    # Set a registry-based policy value
    # Example: disable the Windows Store for all computers
    Set-GPRegistryValue -Name "CFG - Desktop Restrictions" `
        -Key "HKLM\SOFTWARE\Policies\Microsoft\WindowsStore" `
        -ValueName "RemoveWindowsStore" `
        -Type DWord -Value 1

    # Remove a specific policy value
    Remove-GPRegistryValue -Name "CFG - Desktop Restrictions" `
        -Key "HKLM\SOFTWARE\Policies\Microsoft\WindowsStore" `
        -ValueName "RemoveWindowsStore"

    # Generate a report of all settings in a GPO
    Get-GPOReport -Name "SEC - Password Policy" -ReportType Html `
        -Path "$env:USERPROFILE\Desktop\GPO-Report.html"
    ```

    Resultat :

    ```text
    DisplayName     : CFG - Desktop Restrictions
    Key             : HKLM\SOFTWARE\Policies\Microsoft\WindowsStore
    ValueName       : RemoveWindowsStore
    Value           : 1
    Type            : DWord
    ```

=== "GUI (gpmc.msc)"

    1. Dans **Group Policy Objects**, cliquer droit sur la GPO > **Edit**
    2. L'editeur de strategie de groupe s'ouvre (GPME)
    3. Naviguer dans **Computer Configuration** ou **User Configuration**
    4. Modifier les parametres souhaites

---

## Exemples de parametres courants

### Politique de mot de passe

!!! info "Emplacement"

    **Computer Configuration** > **Policies** > **Windows Settings** >
    **Security Settings** > **Account Policies** > **Password Policy**

=== "PowerShell"

    ```powershell
    # Configure password policy via the Default Domain Policy
    # Note: Fine-grained password policies (PSOs) are preferred for per-group settings
    Set-GPRegistryValue -Name "Default Domain Policy" `
        -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
        -ValueName "MinimumPasswordLength" `
        -Type DWord -Value 12
    ```

=== "GUI (gpmc.msc)"

    | Parametre                          | Valeur recommandee |
    | :--------------------------------- | :----------------- |
    | Minimum password length            | 12 caracteres      |
    | Password must meet complexity      | Enabled            |
    | Maximum password age               | 90 jours           |
    | Minimum password age               | 1 jour             |
    | Enforce password history            | 24 mots de passe   |

!!! warning "Politique de mot de passe et domaine"

    La politique de mot de passe definie par GPO ne s'applique qu'au niveau
    **domaine** (Default Domain Policy). Une GPO de mot de passe liee a une OU
    n'aura **aucun effet** sur les comptes du domaine. Pour des politiques
    differentes par groupe, utilisez les **Fine-Grained Password Policies (PSO)**.

### Verrouillage du bureau

!!! info "Emplacement"

    **User Configuration** > **Policies** > **Administrative Templates** >
    **Desktop** / **Start Menu and Taskbar**

```powershell
# Disable access to the Control Panel and PC Settings
Set-GPRegistryValue -Name "CFG - Desktop Restrictions" `
    -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
    -ValueName "NoControlPanel" `
    -Type DWord -Value 1

# Remove Run command from Start Menu
Set-GPRegistryValue -Name "CFG - Desktop Restrictions" `
    -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
    -ValueName "NoRun" `
    -Type DWord -Value 1
```

Resultat :

```text
DisplayName     : CFG - Desktop Restrictions
Key             : HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
ValueName       : NoControlPanel
Value           : 1
Type            : DWord

DisplayName     : CFG - Desktop Restrictions
Key             : HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer
ValueName       : NoRun
Value           : 1
Type            : DWord
```

### Lecteurs reseau mappes

!!! info "Emplacement"

    **User Configuration** > **Preferences** > **Windows Settings** >
    **Drive Maps**

=== "PowerShell"

    ```powershell
    # Map drives via a logon script GPO (alternative to Preferences)
    # Create the script content
    $scriptContent = @'
    # Map shared drives
    New-PSDrive -Name "S" -PSProvider FileSystem -Root "\\SRV-FILE\Partage" -Persist
    New-PSDrive -Name "U" -PSProvider FileSystem -Root "\\SRV-FILE\Users\%USERNAME%" -Persist
    '@

    # Deploy via GPO logon script
    Set-GPRegistryValue -Name "SCR - Drive Mapping" `
        -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Group Policy\Scripts\Logon" `
        -ValueName "Script" `
        -Type String -Value "\\lab.local\NETLOGON\Map-Drives.ps1"
    ```

=== "GUI (gpmc.msc)"

    1. Editer la GPO > **User Configuration** > **Preferences** >
       **Windows Settings** > **Drive Maps**
    2. Cliquer droit > **New** > **Mapped Drive**
    3. Configurer :
        - **Action** : Create
        - **Location** : `\\SRV-FILE\Partage`
        - **Label as** : Partage commun
        - **Drive Letter** : `S:`
    4. Onglet **Common** : cocher **Run in logged-on user's security context**

!!! tip "Preferences vs Scripts"

    Les **Preferences Drive Maps** sont generalement preferees aux scripts de
    logon car elles offrent une interface graphique, un ciblage par element
    (item-level targeting) et une gestion plus propre. Voir la page
    [Preferences vs Policies](preferences-vs-policies.md) pour plus de details.

---

## Ordre de liaison et priorite

Lorsque plusieurs GPO sont liees au meme conteneur, leur **ordre de liaison**
(Link Order) determine la priorite. La GPO avec l'ordre de liaison **le plus bas
(1)** a la **priorite la plus elevee**.

=== "PowerShell"

    ```powershell
    # View link order on an OU
    (Get-GPInheritance -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local").GpoLinks |
        Select-Object DisplayName, Order, Enabled |
        Sort-Object Order

    # Change link order (move GPO to position 1 = highest priority)
    Set-GPLink -Name "SEC - Password Policy" `
        -Target "OU=Utilisateurs,OU=Siege,DC=lab,DC=local" `
        -Order 1
    ```

    Resultat :

    ```text
    DisplayName              Order Enabled
    -----------              ----- -------
    SEC - Password Policy        1    True
    CFG - Mapped Drives          2    True
    CFG - Desktop Restrictions   3    True
    ```

=== "GUI (gpmc.msc)"

    1. Selectionner l'OU dans GPMC
    2. Dans le volet droit, onglet **Linked Group Policy Objects**
    3. Utiliser les fleches haut/bas pour modifier l'ordre de liaison

---

## Sauvegarder et restaurer une GPO

=== "PowerShell"

    ```powershell
    # Backup a single GPO
    Backup-GPO -Name "SEC - Password Policy" `
        -Path "C:\GPO-Backups" -Comment "Before quarterly review"

    # Backup all GPOs
    Backup-GPO -All -Path "C:\GPO-Backups"

    # Restore a GPO from backup
    Restore-GPO -Name "SEC - Password Policy" -Path "C:\GPO-Backups"

    # Import settings from a backup into a different GPO
    Import-GPO -BackupGpoName "SEC - Password Policy" `
        -Path "C:\GPO-Backups" `
        -TargetName "SEC - Password Policy v2" `
        -CreateIfNeeded
    ```

    Resultat :

    ```text
    DisplayName   : SEC - Password Policy
    GpoId         : d4e5f6a7-8b9c-0d1e-2f3a-4b5c6d7e8f9a
    Id            : e5f6a7b8-9c0d-1e2f-3a4b-5c6d7e8f9a0b
    BackupDirectory : C:\GPO-Backups
    CreationTime  : 20/02/2026 14:30:00
    DomainName    : lab.local
    Comment       : Before quarterly review
    ```

=== "GUI (gpmc.msc)"

    1. Cliquer droit sur **Group Policy Objects** > **Manage Backups...**
    2. **Backup** : selectionner la GPO et cliquer **Back Up**
    3. **Restore** : selectionner la sauvegarde et cliquer **Restore**

!!! tip "Automatiser les sauvegardes"

    Planifiez un script PowerShell de sauvegarde GPO via le
    **Planificateur de taches** pour maintenir un historique regulier.
    Voir [Taches planifiees](../../automatisation/taches-planifiees/task-scheduler.md).

---

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Marc, administrateur systeme, doit deployer une GPO pour desactiver l'acces au Panneau de configuration pour tous les utilisateurs de l'OU `Comptabilite`, sauf les membres du groupe `GRP_Admins_Compta`.

    **Etapes** :

    1. Marc cree la GPO sans la lier immediatement :

        ```powershell
        New-GPO -Name "CFG - No Control Panel Compta" `
            -Comment "Disable Control Panel for Accounting users"
        ```

    2. Il configure le parametre souhaite :

        ```powershell
        Set-GPRegistryValue -Name "CFG - No Control Panel Compta" `
            -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
            -ValueName "NoControlPanel" -Type DWord -Value 1
        ```

    3. Il lie la GPO a l'OU cible :

        ```powershell
        New-GPLink -Name "CFG - No Control Panel Compta" `
            -Target "OU=Comptabilite,OU=Siege,DC=lab,DC=local"
        ```

    4. Il teste sur un poste de la comptabilite :

        ```powershell
        gpupdate /force
        gpresult /r /scope:user
        ```

        Resultat :

        ```text
        Applied Group Policy Objects
            CFG - No Control Panel Compta
            CFG - Desktop Settings
            Default Domain Policy
        ```

    5. Le Panneau de configuration est bien bloque. Marc pourra exclure `GRP_Admins_Compta` via le filtrage de securite (voir [Filtrage et heritage](filtrage-et-heritage.md)).

---

## Erreurs courantes

!!! danger "Erreurs courantes"

    1. **Lier la GPO avant de la configurer** : les postes recoivent une GPO vide ou partiellement configuree, ce qui peut causer des comportements inattendus. Toujours configurer d'abord, lier ensuite.

    2. **Supprimer la GPO au lieu du lien** : supprimer une GPO depuis le conteneur Group Policy Objects supprime aussi tous ses liens. Pour retirer l'effet d'une GPO sur une OU specifique, supprimer uniquement le **lien**.

    3. **Ignorer la convention de nommage** : sans prefixe ni logique de nommage, il devient tres difficile de retrouver une GPO parmi des dizaines. Adoptez une convention des le premier jour.

    4. **Ne pas sauvegarder les GPO** : une modification malencontreuse peut casser l'environnement de centaines d'utilisateurs. Un backup regulier avec `Backup-GPO` permet de revenir en arriere rapidement.

    5. **Confondre ordre de liaison et priorite** : l'ordre de liaison 1 signifie priorite **maximale**, pas minimale. Une GPO en position 1 l'emporte sur celle en position 2.

---

## Points cles a retenir

- Creez et configurez une GPO **avant** de la lier pour eviter d'appliquer des
  parametres incomplets.
- Adoptez une **convention de nommage** coherente (prefixes par type).
- Une GPO peut etre liee a plusieurs conteneurs (Site, Domaine, OU) ; chaque lien
  est independant.
- L'**ordre de liaison** determine la priorite quand plusieurs GPO sont liees au
  meme conteneur (1 = priorite maximale).
- La politique de mot de passe par GPO ne fonctionne qu'au niveau **domaine**.
- Sauvegardez regulierement vos GPO avec `Backup-GPO`.

---

## Pour aller plus loin

- [Concepts GPO](concepts-gpo.md) -- rappel sur LSDOU et le fonctionnement general
- [Filtrage et heritage](filtrage-et-heritage.md) -- controler finement qui recoit la GPO
- [Preferences vs Policies](preferences-vs-policies.md) -- choisir entre preference et strategie
- [GPResult et depannage](gpresult-et-depannage.md) -- verifier l'application effective
- [Structure des OU](../adds/structure-ou.md) -- concevoir une arborescence adaptee aux GPO
- [Utilisateurs et groupes](../adds/utilisateurs-et-groupes.md) -- gerer les cibles des GPO
