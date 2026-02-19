---
title: MMC et snap-ins
description: Microsoft Management Console et les snap-ins d'administration Windows Server.
tags:
  - fondamentaux
  - console
  - debutant
---

# MMC et snap-ins

!!! info "Niveau : Debutant"

    Temps estime : 10 minutes

## Qu'est-ce que la MMC ?

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

## Points cles a retenir

- La MMC est un conteneur pour les snap-ins d'administration
- Chaque snap-in gere un aspect specifique (DNS, DHCP, AD, disques, etc.)
- Les fichiers `.msc` sont des raccourcis vers des consoles pre-configurees
- Vous pouvez creer des consoles personnalisees multi-serveurs

## Pour aller plus loin

- [RSAT](rsat.md) - les memes snap-ins sur un poste client
- [Server Manager](server-manager.md) - vue d'ensemble centralisee
