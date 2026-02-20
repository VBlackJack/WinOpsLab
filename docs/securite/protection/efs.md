---
title: "EFS - Encrypting File System"
description: "Chiffrement de fichiers et dossiers avec EFS sur Windows Server 2022 : fonctionnement, certificats, agent de recuperation et comparaison avec BitLocker."
tags:
  - securite
  - protection
  - efs
  - chiffrement
  - certificats
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

# EFS - Encrypting File System

<span class="level-advanced">Avance</span> Â· Temps estime : 30 minutes

EFS (Encrypting File System) est une fonctionnalite de chiffrement integree a NTFS qui protege des fichiers et dossiers individuels. Contrairement a BitLocker qui chiffre des volumes entiers, EFS offre une granularite au niveau du fichier.

---

## Principe de fonctionnement

!!! example "Analogie"

    EFS fonctionne comme un casier personnel dans un espace de travail partage. Chaque employe a son propre cadenas (certificat EFS) pour verrouiller ses documents dans le casier. Les collegues voient que le casier existe, mais ne peuvent pas l'ouvrir. Le DRA (Data Recovery Agent), c'est le passe-partout du gardien, range dans un coffre-fort, utilisable uniquement si un employe perd sa cle.

EFS utilise un chiffrement hybride combinant cryptographie symetrique (pour la performance) et asymetrique (pour la gestion des cles) :

```mermaid
flowchart TD
    A[Fichier a chiffrer] --> B[Generation d'une cle FEK<br/>File Encryption Key - AES 256]
    B --> C[Chiffrement du fichier avec la FEK]
    B --> D[Chiffrement de la FEK avec<br/>la cle publique de l'utilisateur]
    B --> E[Chiffrement de la FEK avec<br/>la cle publique du DRA]
    C --> F[Fichier chiffre sur le disque]
    D --> G[FEK chiffree stockee<br/>dans les metadonnees du fichier]
    E --> H[Copie FEK pour recuperation]
```

### Processus detaille

1. Une **FEK** (File Encryption Key) AES 256 bits est generee pour chaque fichier
2. Le fichier est chiffre avec cette FEK (chiffrement symetrique rapide)
3. La FEK est chiffree avec la **cle publique EFS** de l'utilisateur
4. La FEK chiffree est stockee dans les **attributs NTFS** du fichier (champ `$EFS`)
5. Si un **DRA** (Data Recovery Agent) est configure, une copie de la FEK est egalement chiffree avec sa cle publique

---

## Utilisation d'EFS

### Chiffrer un fichier ou un dossier

```powershell
# Encrypt a single file
cipher /e "D:\Donnees\confidentiel.docx"

# Encrypt a folder (all new files in the folder will be encrypted)
cipher /e /s:"D:\Donnees\Confidentiel"

# Using .NET API
(Get-Item "D:\Donnees\confidentiel.docx").Encrypt()
```

Resultat :

```text
Encrypting files in D:\Donnees\
confidentiel.docx         [OK]
1 file(s) [or directorie(s)] within 1 directorie(s) were encrypted.

Encrypting files in D:\Donnees\Confidentiel\
rapport-Q1.xlsx           [OK]
notes-reunion.docx        [OK]
2 file(s) [or directorie(s)] within 1 directorie(s) were encrypted.
```

### Dechiffrer un fichier ou un dossier

```powershell
# Decrypt a single file
cipher /d "D:\Donnees\confidentiel.docx"

# Decrypt a folder and its contents
cipher /d /s:"D:\Donnees\Confidentiel"

# Using .NET API
(Get-Item "D:\Donnees\confidentiel.docx").Decrypt()
```

Resultat :

```text
Decrypting files in D:\Donnees\
confidentiel.docx         [OK]
1 file(s) [or directorie(s)] within 1 directorie(s) were decrypted.
```

### Verifier l'etat de chiffrement

```powershell
# Check encryption status of files in a directory
cipher /s:"D:\Donnees"

# Output legend:
# E = Encrypted
# U = Unencrypted

# List encrypted files via PowerShell
Get-ChildItem -Path "D:\Donnees" -Recurse |
    Where-Object { $_.Attributes -match "Encrypted" } |
    Select-Object FullName, Length, LastWriteTime
```

Resultat :

```text
 Listing D:\Donnees\
 New files added to this directory will not be encrypted.

U budget-2025.xlsx
U presentation.pptx

 Listing D:\Donnees\Confidentiel\
 New files added to this directory will be encrypted.

E rapport-Q1.xlsx
E notes-reunion.docx
E salaires-2025.xlsx

FullName                                 Length    LastWriteTime
--------                                 ------    -------------
D:\Donnees\Confidentiel\rapport-Q1.xlsx  45312     2025-02-18 14:30
D:\Donnees\Confidentiel\notes-reunion... 23104     2025-02-19 09:15
D:\Donnees\Confidentiel\salaires-2025... 18560     2025-02-20 11:00
```

---

## Certificats EFS

### Certificat utilisateur EFS

Chaque utilisateur qui chiffre un fichier avec EFS utilise un certificat numerique. Si aucun certificat n'existe, Windows en genere un automatiquement (auto-signe).

```powershell
# View the current user's EFS certificate
cipher /r:backup

# List EFS certificates in the user's store
Get-ChildItem -Path Cert:\CurrentUser\My |
    Where-Object { $_.EnhancedKeyUsageList.FriendlyName -contains "Encrypting File System" } |
    Select-Object Subject, Thumbprint, NotAfter
```

Resultat :

```text
Please type the password to protect your .PFX file:
Please retype the password to confirm:
Your .CER file was created successfully.
Your .PFX file was created successfully.

Subject                    Thumbprint                                NotAfter
-------                    ----------                                --------
CN=jdupont, OU=Users,...   C3D4E5F6A7B8C9D0E1F2A3B4C5D6E7F8A9B0C1D2 2026-02-15 10:00:00
```

### Utiliser des certificats d'une PKI d'entreprise

Pour un deploiement en entreprise, les certificats EFS doivent etre emis par la **CA d'entreprise** (pas auto-signes) :

1. Creer un modele de certificat EFS dans la CA
2. Publier le modele sur la CA Enterprise
3. Configurer l'auto-enrollment pour les utilisateurs

!!! tip "Certificats PKI vs auto-signes"

    Les certificats auto-signes EFS posent un probleme majeur : si l'utilisateur perd son profil ou sa cle privee, **les fichiers sont irrecuperables** (sauf si un DRA est configure). Avec une PKI, les cles peuvent etre archivees et recuperees.

---

## Agent de recuperation (DRA)

Le **Data Recovery Agent** est un compte capable de dechiffrer tous les fichiers EFS d'un domaine, meme sans connaitre le mot de passe de l'utilisateur original.

### Configurer un DRA

```powershell
# Generate a DRA certificate (self-signed for lab, PKI for production)
cipher /r:DRA-EFS

# This creates:
# - DRA-EFS.cer (public certificate)
# - DRA-EFS.pfx (certificate with private key - PROTECT THIS!)
```

Resultat :

```text
Please type the password to protect your .PFX file:
Please retype the password to confirm:
Your .CER file was created successfully.
Your .PFX file was created successfully.

    Directory: C:\Secure

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2025-02-20  10:30           1234   DRA-EFS.cer
-a----         2025-02-20  10:30           3456   DRA-EFS.pfx
```

### Deployer le DRA via GPO

```
Computer Configuration
  > Policies
    > Windows Settings
      > Security Settings
        > Public Key Policies
          > Encrypting File System
            > Right-click > Add Data Recovery Agent
            > Browse to the DRA-EFS.cer file
```

### Recuperer un fichier avec le DRA

```powershell
# 1. Import the DRA certificate with private key on the recovery machine
Import-PfxCertificate -FilePath "C:\Secure\DRA-EFS.pfx" `
    -CertStoreLocation Cert:\CurrentUser\My `
    -Password (Read-Host -AsSecureString "PFX password")

# 2. Access the encrypted file - it will be decrypted transparently
# (the DRA's private key allows automatic decryption)

# 3. Optionally decrypt the file permanently
cipher /d "D:\Donnees\confidentiel.docx"
```

!!! danger "Protection de la cle DRA"

    La cle privee du DRA donne acces a **tous les fichiers EFS** du domaine. Elle doit etre :

    - Stockee hors ligne (cle USB dans un coffre-fort)
    - Protegee par un mot de passe fort
    - Accessible uniquement a un nombre restreint de personnes
    - Jamais stockee sur un poste connecte au reseau

---

## Limites d'EFS

| Limitation | Detail |
|------------|--------|
| **NTFS requis** | EFS ne fonctionne pas sur FAT32 ou ReFS |
| **Pas de chiffrement en transit** | EFS protege uniquement les donnees au repos sur le disque local |
| **Copie/deplacment** | Un fichier copie sur un volume non-NTFS perd son chiffrement |
| **Sauvegarde** | Les outils de backup standard sauvegardent les fichiers en clair si le compte de backup a acces |
| **Dossiers systeme** | Impossible de chiffrer les fichiers systeme Windows |
| **Fichiers comprimes** | Un fichier ne peut pas etre a la fois compresse NTFS et chiffre EFS |
| **Partage reseau** | Le chiffrement est applique sur le serveur, pas pendant le transfert |

---

## EFS vs BitLocker

| Critere | EFS | BitLocker |
|---------|-----|-----------|
| **Granularite** | Fichier / dossier | Volume entier |
| **Transparence** | Transparente pour l'utilisateur proprietaire | Transparente au demarrage (avec TPM) |
| **Protection contre** | Acces par un autre utilisateur du meme systeme | Vol physique du disque |
| **Gestion des cles** | Certificat par utilisateur | Cle de volume + protecteur (TPM, PIN, cle USB) |
| **Prerequis** | NTFS | TPM recommande |
| **Impact performance** | Faible (par fichier) | Negligeable (chiffrement AES materiel) |
| **Multiutilisateur** | Oui (ajout de cles par fichier) | Non applicable |
| **Cas d'usage serveur** | Donnees sensibles specifiques | Protection globale du volume |

!!! tip "Complementarite"

    EFS et BitLocker ne sont pas exclusifs. BitLocker protege contre le vol physique, EFS protege contre les acces non autorises entre utilisateurs du meme systeme. L'utilisation combinee offre une protection en profondeur.

---

## Audit des operations EFS

```powershell
# Enable auditing for EFS operations
auditpol /set /subcategory:"File System" /success:enable /failure:enable

# EFS events are logged under:
# Event ID 4656 : Handle requested for an encrypted file
# Event ID 4663 : Access attempt on an encrypted file

# Search for EFS-related events
Get-WinEvent -FilterHashtable @{
    LogName = 'Security'
    Id = @(4656, 4663)
} -MaxEvents 20 |
    Where-Object { $_.Message -match "Encrypt" } |
    Select-Object TimeCreated, Id, Message |
    Format-Table -Wrap
```

Resultat :

```text
TimeCreated            Id    Message
-----------            --    -------
2025-02-20 14:15:32    4656  A handle was requested to an object.
                              Object Name: D:\Donnees\Confidentiel\salaires-2025.xlsx
                              Process Name: C:\Windows\explorer.exe
                              Access Mask: 0x2
2025-02-20 14:10:18    4663  An attempt was made to access an object.
                              Object Name: D:\Donnees\Confidentiel\rapport-Q1.xlsx
                              Subject: LAB\jdupont
```

---

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Claire, du service RH, a quitte l'entreprise. Son responsable demande a l'administrateur David d'acceder aux fichiers EFS chiffres par Claire sur le dossier partage `D:\Partage\RH\Confidentiel` du serveur `SRV-01` (10.0.0.11). Claire n'est plus disponible pour fournir son mot de passe.

    **Diagnostic** :

    ```powershell
    # Verify the files are encrypted
    cipher /s:"D:\Partage\RH\Confidentiel"
    ```

    Resultat :

    ```text
     Listing D:\Partage\RH\Confidentiel\
     New files added to this directory will be encrypted.

    E contrats-2024.xlsx
    E evaluations-annuelles.docx
    E plan-formation.xlsx
    ```

    ```powershell
    # Check who can decrypt these files
    cipher /c "D:\Partage\RH\Confidentiel\contrats-2024.xlsx"
    ```

    Resultat :

    ```text
    Users who can decrypt:
      LAB\c.martin [claire.martin@lab.local]
        Certificate thumbprint: D4E5F6A7...

    Recovery Agents:
      DRA-EFS
        Certificate thumbprint: F6A7B8C9...
    ```

    **Resolution** : un DRA est configure. David importe la cle privee du DRA depuis le stockage securise :

    ```powershell
    # 1. Import the DRA private key
    Import-PfxCertificate -FilePath "E:\Secure\DRA-EFS.pfx" `
        -CertStoreLocation Cert:\CurrentUser\My `
        -Password (Read-Host -AsSecureString "PFX password")

    # 2. Decrypt the files
    cipher /d /s:"D:\Partage\RH\Confidentiel"
    ```

    Resultat :

    ```text
    Decrypting files in D:\Partage\RH\Confidentiel\
    contrats-2024.xlsx          [OK]
    evaluations-annuelles.docx  [OK]
    plan-formation.xlsx         [OK]
    3 file(s) [or directorie(s)] within 1 directorie(s) were decrypted.
    ```

    David supprime ensuite le certificat DRA de son profil et remet la cle USB au coffre-fort.

---

!!! danger "Erreurs courantes"

    1. **Ne pas configurer de DRA avant de deployer EFS** : sans Data Recovery Agent, la perte du profil utilisateur ou du certificat EFS entraine la perte definitive des fichiers chiffres. Configurez toujours un DRA via GPO avant d'autoriser l'utilisation d'EFS.

    2. **Stocker la cle privee du DRA sur un poste connecte au reseau** : la cle DRA donne acces a tous les fichiers EFS du domaine. Elle doit etre stockee hors ligne (cle USB dans un coffre-fort) et importee uniquement pour les operations de recuperation ponctuelles.

    3. **Confondre EFS et BitLocker** : EFS protege contre les acces inter-utilisateurs sur le meme systeme, BitLocker protege contre le vol physique. Copier un fichier EFS sur une cle USB FAT32 le dechiffre automatiquement car FAT32 ne supporte pas EFS.

    4. **Utiliser des certificats EFS auto-signes en production** : les certificats auto-signes n'offrent pas de mecanisme de recuperation de cle. Deployez une PKI d'entreprise avec archivage de cle pour permettre la recuperation en cas de perte du certificat utilisateur.

    5. **Oublier que les fichiers comprimes NTFS ne peuvent pas etre chiffres** : un fichier ne peut pas etre a la fois compresse et chiffre en NTFS. Le chiffrement EFS desactive automatiquement la compression, ce qui peut impacter l'espace disque.

---

## Points cles a retenir

- EFS chiffre les fichiers **individuellement** avec une FEK protegee par le certificat de l'utilisateur
- Un **Data Recovery Agent** (DRA) est indispensable : sans lui, la perte du certificat utilisateur entraine la perte definitive des donnees
- Les certificats EFS doivent provenir d'une **PKI d'entreprise** en production (pas auto-signes)
- EFS et BitLocker sont **complementaires** : BitLocker contre le vol physique, EFS contre les acces inter-utilisateurs
- EFS ne fonctionne que sur **NTFS** et ne protege pas les donnees en transit
- La cle privee du DRA doit etre stockee **hors ligne** dans un emplacement securise

---

## Pour aller plus loin

- BitLocker sur serveur (voir la page [BitLocker](bitlocker.md))
- Concepts PKI (voir la page [Concepts PKI](../pki/concepts-pki.md))
- Microsoft : Encrypting File System overview
- Microsoft : Best practices for EFS

