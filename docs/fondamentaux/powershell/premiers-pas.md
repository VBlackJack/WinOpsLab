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
title: Premiers pas avec PowerShell
description: Introduction a PowerShell - lancement, navigation et concepts de base.
tags:
  - fondamentaux
  - powershell
  - debutant
---

# Premiers pas avec PowerShell

<span class="level-beginner">Debutant</span> Â· Temps estime : 20 minutes

## Qu'est-ce que PowerShell ?

!!! example "Analogie"

    Imaginez `cmd.exe` comme un telephone fixe : il fait le minimum, appeler et raccrocher. PowerShell, c'est un smartphone â€” il fait tout ce que fait le telephone fixe, mais aussi naviguer, envoyer des messages, prendre des photos. C'est un outil complet qui ouvre un monde de possibilites.

PowerShell est le shell et langage de script de Microsoft pour l'administration systeme. Contrairement a `cmd.exe`, PowerShell manipule des **objets .NET** plutot que du texte brut, ce qui le rend bien plus puissant.

| Caracteristique | cmd.exe | PowerShell |
|-----------------|---------|------------|
| Type de donnees | Texte | Objets .NET |
| Extensibilite | Limitee | Modules illimites |
| Scripting | Batch (.bat) | Scripts (.ps1) |
| Administration a distance | Limitee | Remoting integre |
| Autocompletion | Basique | Avancee (++tab++) |

## Lancer PowerShell

### Sur Windows Server

=== "Desktop Experience"

    - Clic droit sur le menu Demarrer > **Windows PowerShell (Admin)**
    - Ou taper `powershell` dans la barre de recherche

=== "Server Core"

    PowerShell se lance automatiquement apres la connexion.

    ```powershell
    # If you're in cmd.exe, switch to PowerShell
    powershell
    ```

### PowerShell vs Windows PowerShell

| Version | Executable | Framework | Versions |
|---------|-----------|-----------|----------|
| Windows PowerShell | `powershell.exe` | .NET Framework | 5.1 (inclus dans Windows) |
| PowerShell (Core) | `pwsh.exe` | .NET (cross-platform) | 7.x (a installer separement) |

!!! tip "Quelle version utiliser ?"

    Sur Windows Server 2022, utilisez **Windows PowerShell 5.1** (inclus par defaut).
    C'est la version officielle pour l'administration Windows Server.
    PowerShell 7 est utile pour le scripting cross-platform.

```powershell
# Check the current PowerShell version
$PSVersionTable.PSVersion
```

Resultat :

```text
Major  Minor  Build  Revision
-----  -----  -----  --------
5      1      20348  2227
```

## Concepts fondamentaux

### La console

La console PowerShell est interactive : vous tapez une commande et obtenez un resultat immediat.

```powershell
# Display the current date
Get-Date

# Display the hostname
hostname

# Display the current user
whoami

# Display the current directory
Get-Location
```

Resultat :

```text
PS C:\Users\Administrator> Get-Date

mercredi 19 fevrier 2026 14:32:07

PS C:\Users\Administrator> hostname
SRV-DC01

PS C:\Users\Administrator> whoami
lab\administrator

PS C:\Users\Administrator> Get-Location

Path
----
C:\Users\Administrator
```

### Les commandes s'appellent des cmdlets

!!! example "Analogie"

    Le format **Verbe-Nom** fonctionne comme une instruction donnee a un assistant : "**Apporte**-moi le **courrier**", "**Ouvre**-la **porte**", "**Supprime**-ce **fichier**". L'action est toujours claire car vous dites d'abord *quoi faire*, puis *sur quoi*.

Une **cmdlet** (command-let) suit toujours le format **Verbe-Nom** :

```
Get-Process       # Verbe: Get, Nom: Process
Stop-Service      # Verbe: Stop, Nom: Service
New-Item          # Verbe: New, Nom: Item
Remove-NetRoute   # Verbe: Remove, Nom: NetRoute
```

Les verbes les plus courants :

| Verbe | Action | Exemple |
|-------|--------|---------|
| `Get` | Lire / Obtenir | `Get-Service` |
| `Set` | Modifier | `Set-TimeZone` |
| `New` | Creer | `New-Item` |
| `Remove` | Supprimer | `Remove-Item` |
| `Start` | Demarrer | `Start-Service` |
| `Stop` | Arreter | `Stop-Process` |
| `Restart` | Redemarrer | `Restart-Computer` |
| `Test` | Tester | `Test-Connection` |
| `Enable` | Activer | `Enable-PSRemoting` |
| `Disable` | Desactiver | `Disable-NetAdapter` |
| `Install` | Installer | `Install-WindowsFeature` |

### Les alias

PowerShell fournit des alias (raccourcis) pour les commandes courantes :

```powershell
# These are equivalent
Get-ChildItem
dir
ls

# These are equivalent
Set-Location C:\Windows
cd C:\Windows

# These are equivalent
Clear-Host
cls

# List all aliases
Get-Alias
```

Resultat (extrait de `Get-Alias`) :

```text
CommandType     Name                         Version    Source
-----------     ----                         -------    ------
Alias           cd -> Set-Location
Alias           cls -> Clear-Host
Alias           copy -> Copy-Item
Alias           del -> Remove-Item
Alias           dir -> Get-ChildItem
Alias           ls -> Get-ChildItem
Alias           mv -> Move-Item
Alias           pwd -> Get-Location
Alias           rm -> Remove-Item
```

!!! warning "Eviter les alias dans les scripts"

    Utilisez toujours les noms complets des cmdlets dans vos scripts
    pour la lisibilite et la maintenabilite. Les alias sont pratiques
    uniquement en mode interactif.

## Navigation dans le systeme de fichiers

```powershell
# Display current location
Get-Location   # alias: pwd

# Change directory
Set-Location C:\Windows   # alias: cd

# List files and directories
Get-ChildItem   # alias: ls, dir

# List only files
Get-ChildItem -File

# List only directories
Get-ChildItem -Directory

# List recursively
Get-ChildItem -Recurse

# Create a directory
New-Item -ItemType Directory -Name "MonDossier"

# Create a file
New-Item -ItemType File -Name "test.txt"

# Copy a file
Copy-Item -Path "test.txt" -Destination "backup.txt"

# Move a file
Move-Item -Path "backup.txt" -Destination "C:\Temp\backup.txt"

# Delete a file
Remove-Item -Path "test.txt"
```

Resultat (exemple pour `Get-ChildItem -Directory`) :

```text
PS C:\> Get-ChildItem -Directory

    Directory: C:\

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         2/10/2026   8:15 AM                PerfLogs
d-r---         2/15/2026   3:22 PM                Program Files
d-r---         1/20/2026   9:00 AM                Program Files (x86)
d-----         2/19/2026  10:45 AM                Temp
d-r---         2/18/2026   2:30 PM                Users
d-----         2/19/2026  11:00 AM                Windows
```

## Execution de scripts

Par defaut, l'execution de scripts est restreinte. Verifiez et modifiez la politique :

```powershell
# Check current execution policy
Get-ExecutionPolicy

# Set execution policy to allow local scripts
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

# Run a script
.\mon-script.ps1
```

Resultat :

```text
PS C:\> Get-ExecutionPolicy
Restricted

PS C:\> Set-ExecutionPolicy RemoteSigned -Scope CurrentUser

Execution Policy Change
The execution policy helps protect you from scripts that you do not trust.
...
Do you want to change the execution policy?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): Y

PS C:\> Get-ExecutionPolicy
RemoteSigned
```

| Politique | Description |
|-----------|-------------|
| `Restricted` | Aucun script autorise (defaut) |
| `AllSigned` | Seuls les scripts signes sont autorises |
| `RemoteSigned` | Scripts locaux OK, scripts distants doivent etre signes |
| `Unrestricted` | Tous les scripts sont autorises (deconseille) |

!!! tip "Recommandation"

    Utilisez `RemoteSigned` pour un bon compromis securite/praticite en lab.
    En production, privilegiez `AllSigned`.

!!! example "Scenario pratique"

    **Contexte** : Sophie, administratrice junior, vient d'installer Windows Server 2022 sur un nouveau serveur `SRV-01`. Elle doit verifier que PowerShell fonctionne et preparer l'environnement pour executer des scripts.

    **Etape 1** : Verifier la version de PowerShell

    ```powershell
    $PSVersionTable.PSVersion
    ```

    ```text
    Major  Minor  Build  Revision
    -----  -----  -----  --------
    5      1      20348  2227
    ```

    **Etape 2** : Verifier la politique d'execution actuelle

    ```powershell
    Get-ExecutionPolicy
    ```

    ```text
    Restricted
    ```

    **Etape 3** : Passer en `RemoteSigned` pour autoriser les scripts locaux

    ```powershell
    Set-ExecutionPolicy RemoteSigned -Scope LocalMachine -Force
    ```

    **Etape 4** : Confirmer le changement

    ```powershell
    Get-ExecutionPolicy
    ```

    ```text
    RemoteSigned
    ```

    Sophie peut maintenant executer ses scripts `.ps1` locaux en toute securite.

!!! danger "Erreurs courantes"

    1. **Lancer PowerShell sans les droits administrateur** : de nombreuses cmdlets systeme (comme `Set-ExecutionPolicy` au niveau Machine) echouent sans elevation. Pensez a faire clic droit > "Executer en tant qu'administrateur".

    2. **Confondre `powershell.exe` et `pwsh.exe`** : sur Windows Server 2022, `powershell.exe` (v5.1) est installe par defaut. `pwsh.exe` (v7.x) doit etre installe separement. Les modules ne sont pas toujours compatibles entre les deux.

    3. **Utiliser des alias dans les scripts** : ecrire `ls` ou `cd` dans un fichier `.ps1` est tentant mais nuit a la lisibilite. Privilegiez `Get-ChildItem` et `Set-Location`.

    4. **Oublier les guillemets autour des chemins avec espaces** : `Set-Location C:\Program Files` provoque une erreur. Utilisez `Set-Location "C:\Program Files"`.

    5. **Passer la politique d'execution en `Unrestricted`** : c'est excessif et dangereux. `RemoteSigned` offre le bon equilibre entre securite et praticite.

## Points cles a retenir

- PowerShell manipule des objets, pas du texte
- Les cmdlets suivent le format Verbe-Nom
- Utilisez ++tab++ pour l'autocompletion
- `Get-` pour lire, `Set-` pour modifier, `New-` pour creer
- Les alias sont des raccourcis (pratiques en interactif, a eviter dans les scripts)

## Pour aller plus loin

- [Cmdlets essentielles](cmdlets-essentielles.md) - les commandes a connaitre
- [Pipeline et objets](pipeline-et-objets.md) - la puissance du pipeline
- [Aide et decouverte](aide-et-decouverte.md) - apprendre a se debrouiller seul

