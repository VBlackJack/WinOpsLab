---
title: Installer le premier controleur de domaine
description: Deployer le premier controleur de domaine et creer une nouvelle foret Active Directory.
tags:
  - active-directory
  - adds
  - intermediaire
---

# Installer le premier controleur de domaine

<span class="level-intermediate">Intermediaire</span> · Temps estime : 30 minutes

## Processus d'installation

```mermaid
flowchart TD
    A["Preparer le serveur"] --> B{"Prerequis remplis ?"}
    B -- Non --> C["Configurer IP statique,\nnom, DNS, mises a jour"]
    C --> B
    B -- Oui --> D["Installer le role AD DS"]
    D --> E{"Premier DC\nde l'infrastructure ?"}
    E -- Oui --> F["Creer une nouvelle foret\n(Install-ADDSForest)"]
    E -- Non --> G["Rejoindre un domaine existant\n(Install-ADDSDomainController)"]
    F --> H["Redemarrage automatique"]
    G --> H
    H --> I["Verification post-promotion"]
    I --> J["Verifier NTDS, DNS,\nSYSVOL, FSMO, SRV records"]

    style A fill:#1565c0,color:#fff
    style F fill:#2e7d32,color:#fff
    style G fill:#2e7d32,color:#fff
    style J fill:#00838f,color:#fff
```

## Prerequis

Avant de promouvoir un serveur en controleur de domaine :

- [x] Serveur renomme (ex: `DC-01`)
- [x] Adresse IP statique configuree
- [x] DNS pointe vers lui-meme (il deviendra serveur DNS)
- [x] Fuseau horaire correct
- [x] Mises a jour installees

!!! warning "Choix du nom de domaine"

    Evitez les noms de domaine en `.local` dans un environnement de production
    (conflit possible avec mDNS/Bonjour). Preferez un sous-domaine de votre
    domaine public : `ad.monentreprise.com` ou `corp.monentreprise.com`.

    En lab, `lab.local` est acceptable.

## Etape 1 : Installer le role AD DS

```powershell
# Install AD DS role with management tools
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
```

!!! note "Le role DNS sera installe automatiquement"

    Lors de la promotion, si vous cochez l'option DNS Server (recommande),
    le role DNS sera installe automatiquement comme dependance.

## Etape 2 : Promouvoir en controleur de domaine

### Nouvelle foret (premier DC de l'infrastructure)

=== "PowerShell (recommande)"

    ```powershell
    # Import the ADDSDeployment module
    Import-Module ADDSDeployment

    # Promote to DC - new forest
    Install-ADDSForest `
        -DomainName "lab.local" `
        -DomainNetbiosName "LAB" `
        -ForestMode "WinThreshold" `
        -DomainMode "WinThreshold" `
        -InstallDns:$true `
        -DatabasePath "C:\Windows\NTDS" `
        -LogPath "C:\Windows\NTDS" `
        -SysvolPath "C:\Windows\SYSVOL" `
        -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd!DSRM" -AsPlainText -Force) `
        -Force:$true
    ```

    !!! danger "Mot de passe DSRM"

        Le mot de passe **SafeModeAdministratorPassword** (DSRM) est crucial.
        Il permet d'acceder au DC en mode restauration des services d'annuaire.
        Notez-le dans un endroit securise. Il ne peut pas etre recupere.

=== "Server Manager (GUI)"

    1. Ouvrir Server Manager
    2. Cliquer sur le drapeau de notification jaune
    3. Choisir **Promote this server to a domain controller**
    4. Selectionner **Add a new forest**
    5. Saisir le nom de domaine racine : `lab.local`
    6. Configurer le niveau fonctionnel de la foret et du domaine
    7. Cocher **DNS Server**
    8. Definir le mot de passe DSRM
    9. Ignorer la delegation DNS (normal pour le premier DC)
    10. Accepter le nom NetBIOS (`LAB`)
    11. Conserver les chemins par defaut
    12. Verifier les options et installer

Le serveur redemarrera automatiquement apres la promotion.

## Etape 3 : Verification post-promotion

```powershell
# Verify AD DS is running
Get-Service -Name "NTDS" | Select-Object Name, Status

# Verify DNS is running
Get-Service -Name "DNS" | Select-Object Name, Status

# Verify domain information
Get-ADDomain | Select-Object DNSRoot, NetBIOSName, DomainMode, PDCEmulator

# Verify forest information
Get-ADForest | Select-Object Name, ForestMode, SchemaMaster, DomainNamingMaster

# List domain controllers
Get-ADDomainController -Filter * | Select-Object Name, IPv4Address, IsGlobalCatalog

# Verify FSMO roles
netdom query fsmo

# Verify DNS zones
Get-DnsServerZone

# Verify SYSVOL share
Get-SmbShare | Where-Object Name -in "SYSVOL", "NETLOGON"

# Test DNS resolution
Resolve-DnsName -Name "lab.local" -Type SOA
Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.lab.local" -Type SRV
```

!!! tip "Test DNS SRV"

    La resolution de `_ldap._tcp.dc._msdcs.lab.local` en enregistrement SRV
    confirme que DNS et AD fonctionnent correctement ensemble.

## Etape 4 : Ajouter un second controleur de domaine

Pour la redondance, ajoutez toujours un second DC :

```powershell
# On the second server (SRV-DC-02), install AD DS
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

# Promote as additional DC in existing domain
Import-Module ADDSDeployment

Install-ADDSDomainController `
    -DomainName "lab.local" `
    -InstallDns:$true `
    -Credential (Get-Credential "LAB\Administrator") `
    -DatabasePath "C:\Windows\NTDS" `
    -LogPath "C:\Windows\NTDS" `
    -SysvolPath "C:\Windows\SYSVOL" `
    -SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd!DSRM" -AsPlainText -Force) `
    -Force:$true
```

## Ports reseau requis

Les ports suivants doivent etre ouverts entre les DC et les clients :

| Port | Protocole | Service |
|------|-----------|---------|
| 53 | TCP/UDP | DNS |
| 88 | TCP/UDP | Kerberos |
| 135 | TCP | RPC Endpoint Mapper |
| 389 | TCP/UDP | LDAP |
| 445 | TCP | SMB (SYSVOL, Netlogon) |
| 464 | TCP/UDP | Kerberos Password Change |
| 636 | TCP | LDAPS |
| 3268 | TCP | Global Catalog |
| 3269 | TCP | Global Catalog SSL |
| 49152-65535 | TCP | RPC Dynamic Ports |

## Points cles a retenir

- Le premier DC cree la foret et le domaine racine
- Le role DNS doit etre installe avec AD DS
- Le mot de passe DSRM est critique et doit etre conserve en securite
- Toujours deployer au moins 2 DC pour la redondance
- Verifier les enregistrements DNS SRV apres la promotion

## Pour aller plus loin

- [Structure des OU](structure-ou.md) - organiser l'annuaire
- [Utilisateurs et groupes](utilisateurs-et-groupes.md) - creer les premiers objets
- [DNS integre AD](../dns/zones-integrees-ad.md) - comprendre le DNS cree automatiquement
