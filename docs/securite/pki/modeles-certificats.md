---
title: "Modeles de certificats"
description: "Gestion des modeles de certificats AD CS : duplication, personnalisation, permissions, publication et modeles courants pour Windows Server 2022."
tags:
  - securite
  - pki
  - certificats
  - modeles
  - adcs
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

# Modeles de certificats

<span class="level-advanced">Avance</span> Â· Temps estime : 35 minutes

Les modeles de certificats (Certificate Templates) definissent les proprietes des certificats emis par une CA Enterprise. Ils controlent la duree de validite, l'algorithme de chiffrement, les usages autorises et les permissions d'inscription.

---

## Principe des modeles

!!! example "Analogie"

    Un modele de certificat est comparable a un formulaire administratif pre-rempli : le formulaire de passeport (modele) definit les champs obligatoires, les pieces justificatives et les droits du titulaire. Vous ne remplissez pas un formulaire de passeport pour demander un permis de conduire. De meme, chaque type de certificat a son propre modele adapte a son usage.

Un modele de certificat est un objet stocke dans Active Directory qui definit :

- Le **type de certificat** : serveur, utilisateur, code signing, etc.
- La **duree de validite** et la **periode de renouvellement**
- L'**algorithme et la taille de cle**
- Les **usages** (Key Usage et Enhanced Key Usage)
- Les **permissions** : qui peut s'inscrire, qui peut gerer le modele
- Le **mode de delivrance** : automatique ou avec approbation manuelle

```mermaid
flowchart LR
    A[Modele de certificat] --> B[Permissions d'inscription]
    A --> C[Proprietes cryptographiques]
    A --> D[Extensions et usages]
    A --> E[Conditions de delivrance]

    B --> B1[Utilisateurs / Groupes autorises]
    C --> C1[RSA 2048 / SHA256]
    D --> D1[Server Authentication]
    D --> D2[Client Authentication]
    E --> E1[Approbation automatique]
    E --> E2[Approbation par un gestionnaire]
```

---

## Modeles integres courants

Windows Server fournit des modeles pre-configures. Les plus utilises :

| Modele | Usage | Enhanced Key Usage |
|--------|-------|--------------------|
| **Web Server** | Certificat SSL/TLS pour serveurs web | Server Authentication |
| **Computer** | Authentification machine | Client Authentication, Server Authentication |
| **User** | Authentification utilisateur, email | Client Authentication, EFS, Secure Email |
| **Domain Controller** | Authentification des DC | Client Authentication, Server Authentication |
| **Code Signing** | Signature de code | Code Signing |
| **OCSP Response Signing** | Reponses OCSP | OCSP Signing |

!!! warning "Ne jamais modifier les modeles originaux"

    Les modeles integres ne doivent pas etre modifies directement. Creez toujours une **copie** (duplication) avant de personnaliser.

---

## Dupliquer un modele

La duplication est la methode standard pour creer un modele personnalise a partir d'un modele existant.

### Via la console certtmpl.msc

1. Ouvrir `certtmpl.msc` (Certificate Templates Console)
2. Clic droit sur le modele a dupliquer > **Duplicate Template**
3. Configurer les proprietes du nouveau modele
4. Nommer le modele (Display Name et Template Name)

### Via PowerShell

```powershell
# List all certificate templates in AD
$configContext = ([ADSI]"LDAP://RootDSE").configurationNamingContext
$templateContainer = "CN=Certificate Templates,CN=Public Key Services,CN=Services,$configContext"

Get-ADObject -SearchBase $templateContainer -Filter { objectClass -eq "pKICertificateTemplate" } |
    Select-Object Name, DistinguishedName |
    Sort-Object Name
```

Resultat :

```text
Name                    DistinguishedName
----                    -----------------
ClientAuth              CN=ClientAuth,CN=Certificate Templates,CN=Public Key Services,...
CodeSigning             CN=CodeSigning,CN=Certificate Templates,CN=Public Key Services,...
Computer                CN=Computer,CN=Certificate Templates,CN=Public Key Services,...
DomainController        CN=DomainController,CN=Certificate Templates,CN=Public Key Services,...
EFS                     CN=EFS,CN=Certificate Templates,CN=Public Key Services,...
Lab-WebServer           CN=Lab-WebServer,CN=Certificate Templates,CN=Public Key Services,...
User                    CN=User,CN=Certificate Templates,CN=Public Key Services,...
WebServer               CN=WebServer,CN=Certificate Templates,CN=Public Key Services,...
```

!!! tip "Versions de modele"

    - **Version 1** : compatibilite Windows 2000, fonctionnalites limitees
    - **Version 2** : Windows Server 2003+, auto-enrollment, archivage de cle
    - **Version 3** : Windows Server 2008+, CNG (Cryptography Next Generation)
    - **Version 4** : Windows Server 2012+, renouvellement avec meme cle, Key Attestation

    Privilegiez les **versions 3 ou 4** pour les nouveaux modeles.

---

## Configurer un modele Web Server personnalise

### Proprietes generales

| Parametre | Valeur recommandee |
|-----------|-------------------|
| **Template Display Name** | Lab-WebServer |
| **Template Name** | Lab-WebServer |
| **Validity Period** | 1 an |
| **Renewal Period** | 6 semaines |

### Proprietes cryptographiques

| Parametre | Valeur recommandee |
|-----------|-------------------|
| **Provider Category** | Key Storage Provider |
| **Algorithm Name** | RSA |
| **Minimum Key Size** | 2048 |
| **Hash Algorithm** | SHA256 |

### Request Handling

| Parametre | Valeur recommandee |
|-----------|-------------------|
| **Purpose** | Signature and Encryption |
| **Allow private key to be exported** | Oui (pour les certificats web) |

### Extensions - Key Usage

- Digital Signature
- Key Encipherment

### Extensions - Application Policies (EKU)

- Server Authentication (1.3.6.1.5.5.7.3.1)

### Subject Name

| Option | Choix |
|--------|-------|
| **Supply in the request** | Oui (le demandeur specifie le Subject) |

!!! danger "Securite du Subject Name"

    L'option **Supply in the request** permet au demandeur de choisir le nom du certificat. Cela peut etre exploite pour creer des certificats frauduleux (ex: un certificat pour le nom d'un autre serveur). Restreignez les permissions d'inscription et envisagez l'approbation manuelle par un gestionnaire de CA.

---

## Configurer les permissions

Les permissions d'un modele controlent qui peut demander un certificat base sur ce modele.

### Permissions essentielles

| Permission | Description |
|------------|-------------|
| **Read** | Voir le modele dans la liste |
| **Enroll** | Soumettre une demande de certificat |
| **Autoenroll** | Inscription automatique via GPO |
| **Write** | Modifier le modele |
| **Full Control** | Controle total sur le modele |

### Exemple de configuration

```powershell
# The template permissions are stored as ACLs on the AD object
# Use ADSI or the GUI to configure them

# Example: Grant Enroll permission to a group on a template
$templateName = "Lab-WebServer"
$configContext = ([ADSI]"LDAP://RootDSE").configurationNamingContext
$templateDN = "CN=$templateName,CN=Certificate Templates,CN=Public Key Services,CN=Services,$configContext"

# View current permissions
dsacls $templateDN

# Best practice: create dedicated security groups
# - "PKI-WebServer-Enroll" : servers allowed to request web certificates
# - "PKI-Template-Managers" : administrators who manage templates
```

Resultat :

```text
Access list:
  {Enroll}
    Allow LAB\PKI-WebServer-Enroll   SPECIAL ACCESS
                                       Enroll
                                       Autoenroll
    Allow LAB\Domain Admins          FULL CONTROL
    Allow LAB\Enterprise Admins      FULL CONTROL
    Allow NT AUTHORITY\Authenticated Users
                                       READ
```

### Recommandations

- Creer des **groupes de securite dedies** pour chaque modele (`PKI-<Template>-Enroll`)
- Ne jamais accorder `Enroll` a **Authenticated Users** sur des modeles sensibles
- Separer les droits `Enroll` (demande) et `Write` (modification)
- Utiliser `Autoenroll` uniquement pour les certificats machine deployes massivement

---

## Publier un modele sur la CA

Un modele cree dans AD n'est pas automatiquement disponible sur la CA. Il faut le **publier** explicitement.

### Via la console certsrv.msc

1. Ouvrir `certsrv.msc` (Certification Authority)
2. Developper l'arborescence de la CA
3. Clic droit sur **Certificate Templates** > **New** > **Certificate Template to Issue**
4. Selectionner le modele a publier

### Via PowerShell

```powershell
# Add a template to the CA's list of issued templates
$templateName = "Lab-WebServer"
$caName = "Lab-SUB-CA"

# Using certutil
certutil -SetCATemplates +$templateName

# Verify published templates
certutil -CATemplates
```

Resultat :

```text
CertUtil: -SetCATemplates command completed successfully.

Lab-WebServer -- Lab-WebServer
DomainController -- Domain Controller
Computer -- Computer
User -- User
WebServer -- Web Server
```

### Retirer un modele de la publication

```powershell
# Remove a template from the CA (stop issuing)
certutil -SetCATemplates -$templateName

# This does NOT delete the template from AD, only stops the CA from issuing it
```

---

## Modele pour certificat utilisateur

### Configuration typique

| Parametre | Valeur |
|-----------|--------|
| **Template Name** | Lab-User |
| **Validity** | 1 an |
| **Key Size** | 2048 |
| **Key Usage** | Digital Signature, Key Encipherment |
| **EKU** | Client Authentication, Secure Email, EFS |
| **Subject Name** | Build from AD information (E-mail, UPN) |
| **Autoenroll** | Oui (via GPO) |

### Configuration du Subject depuis AD

Pour les certificats utilisateur, le Subject Name doit etre **construit a partir d'Active Directory** (pas fourni dans la requete) :

- **Subject name format** : Common Name
- **Include e-mail in subject name** : Oui
- **Include e-mail in subject alternative name** : Oui

---

## Demander un certificat manuellement

### Via PowerShell

```powershell
# Request a certificate using a specific template
$template = "Lab-WebServer"
$subjectName = "CN=www.lab.local"
$san = "dns=www.lab.local&dns=lab.local"

# Create the certificate request and submit it
$enrollment = New-Object -ComObject X509Enrollment.CX509Enrollment
$request = New-Object -ComObject X509Enrollment.CX509CertificateRequestCertificate

# Simplified method using certreq
# Create an INF file for the request
$inf = @"
[Version]
Signature="`$Windows NT`$"

[NewRequest]
Subject = "$subjectName"
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "$san"

[RequestAttributes]
CertificateTemplate = $template
"@

$inf | Out-File -FilePath "C:\Temp\certrequest.inf" -Encoding ASCII

# Generate the request
certreq -new "C:\Temp\certrequest.inf" "C:\Temp\certrequest.req"

# Submit to the CA
certreq -submit -config "LAB-SUB-CA\Lab-SUB-CA" "C:\Temp\certrequest.req" "C:\Temp\certresponse.cer"

# Install the certificate
certreq -accept "C:\Temp\certresponse.cer"
```

---

## Scenario pratique

!!! example "Scenario pratique"

    **Contexte** : Lucas, administrateur PKI, doit creer un modele de certificat pour les serveurs web internes. Les developpeurs doivent pouvoir demander des certificats SSL pour leurs applications hebergees sur IIS, mais il faut eviter qu'ils puissent generer des certificats pour des noms de domaine qu'ils ne gerent pas.

    **Solution** :

    1. Dupliquer le modele `Web Server` integre via `certtmpl.msc`
    2. Nommer le nouveau modele `Lab-WebServer`
    3. Configurer les proprietes :

    ```powershell
    # Verify the template exists after creation
    $configContext = ([ADSI]"LDAP://RootDSE").configurationNamingContext
    $templateDN = "CN=Lab-WebServer,CN=Certificate Templates,CN=Public Key Services,CN=Services,$configContext"
    Get-ADObject -Identity $templateDN -Properties DisplayName, msPKI-Cert-Template-OID
    ```

    Resultat :

    ```text
    DisplayName                : Lab-WebServer
    DistinguishedName          : CN=Lab-WebServer,CN=Certificate Templates,...
    msPKI-Cert-Template-OID    : 1.3.6.1.4.1.311.21.8.12345678.1234567.1234567
    ```

    4. Configurer la securite : seul le groupe `PKI-WebServer-Enroll` peut demander un certificat
    5. Activer l'approbation par un gestionnaire de CA pour les demandes avec `Supply in the request`
    6. Publier le modele sur la CA :

    ```powershell
    certutil -SetCATemplates +Lab-WebServer
    ```

    Lucas cree ensuite le groupe `PKI-WebServer-Enroll` dans AD et y ajoute les comptes machine des serveurs web autorises.

---

!!! danger "Erreurs courantes"

    1. **Modifier le modele original au lieu de le dupliquer** : les modeles integres peuvent etre reinitialises par des mises a jour Windows. Toute personnalisation serait perdue. Dupliquez toujours avant de modifier.

    2. **Accorder `Enroll` a `Authenticated Users` sur un modele avec `Supply in the request`** : n'importe quel utilisateur du domaine pourrait demander un certificat pour n'importe quel nom DNS, permettant des attaques de type man-in-the-middle. Restreignez les permissions et activez l'approbation manuelle.

    3. **Oublier de publier le modele sur la CA** : un modele cree dans AD n'est pas automatiquement disponible. Sans la publication via `certsrv.msc` ou `certutil -SetCATemplates`, les demandes echoueront avec l'erreur "template not found".

    4. **Utiliser des modeles version 1 pour de nouvelles installations** : les modeles v1 ne supportent ni l'auto-enrollment, ni l'archivage de cle, ni les algorithmes CNG modernes. Privilegiez les versions 3 ou 4.

---

## Points cles a retenir

- Toujours **dupliquer** un modele existant avant de le personnaliser, ne jamais modifier les originaux
- Les permissions d'inscription doivent etre **restreintes** via des groupes de securite dedies
- L'option **Supply in the request** pour le Subject Name necessite des controles supplementaires
- Un modele doit etre **publie** sur la CA pour etre disponible a l'inscription
- Privilegier les modeles **version 3 ou 4** avec des algorithmes modernes (RSA 2048+, SHA-256)
- Separer les modeles par usage : web, utilisateur, machine, code signing

---

## Pour aller plus loin

- Inscription automatique via GPO (voir la page [Inscription automatique](inscription-automatique.md))
- Concepts PKI (voir la page [Concepts PKI](concepts-pki.md))
- Microsoft : Certificate Templates overview
- Microsoft : Configuring certificate auto-enrollment

