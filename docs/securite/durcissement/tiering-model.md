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
title: Modele de Tiering (Red Forest)
description: Comprendre et implementer le modele de Tiering Microsoft pour proteger Active Directory (Tier 0, 1, 2).
tags:
  - securite
  - tiering
  - active-directory
  - red-forest
---

# Modele de Tiering (Enterprise Access Model)

<span class="level-advanced">Avance</span> · Temps estime : 45 minutes

## Pourquoi le Tiering ?

Dans une attaque classique (type Ransomware), l'attaquant commence par compromettre un poste de travail (Phishing), vole les identifiants en memoire (LSASS), et rebondit de machine en machine (Lateral Movement) jusqu'a trouver un compte Administrateur de Domaine.

Le **Modele de Tiering** brise cette chaine en creant des **silos etanches**.

## Architecture des Tiers

![Modele de Tiering](../../diagrams/ad-tiering.drawio#Tiering_Model)

### Tier 0 : Identite (Zone Rouge)
C'est le cÅ“ur du pouvoir. Quiconque controle le Tier 0 controle toute l'entreprise.
*   **Ressources :** Controleurs de Domaine, PKI, ADFS, Azure AD Connect.
*   **Admins :** Enterprise Admins, Domain Admins.
*   **Regle d'Or :** Un Admin Tier 0 ne se connecte **JAMAIS** sur un serveur Tier 1 ou 2.

### Tier 1 : Applications (Zone Jaune)
Les serveurs qui hebergent les donnees et applications.
*   **Ressources :** Serveurs de Fichiers, SQL, Exchange, IIS.
*   **Admins :** Server Admins.
*   **Regle :** Un Admin Tier 1 ne gere pas les postes de travail (Tier 2).

### Tier 2 : Postes de Travail (Zone Verte)
La zone la plus exposee aux attaques (Internet, Emails).
*   **Ressources :** Windows 10/11, Imprimantes.
*   **Admins :** Helpdesk, Workstation Admins.

## Mise en Å“uvre technique

Pour implementer le Tiering, il ne suffit pas de le dire, il faut l'imposer via **GPO**.

### 1. Separation des comptes
Chaque administrateur doit avoir plusieurs comptes.
*   `j.dupont` : Compte standard (Email, Web) -> **Tier 2**
*   `adm-j.dupont` : Administration Serveurs -> **Tier 1**
*   `admin-j.dupont` : Administration AD (si necessaire) -> **Tier 0**

### 2. Restriction des connexions (Logon Restrictions)
On utilise le droit "Deny log on locally" et "Deny log on as a service" via GPO.

| GPO Tier 0 (Liee aux DC) | GPO Tier 1 (Liee aux Serveurs) | GPO Tier 2 (Liee aux PC) |
| :--- | :--- | :--- |
| **Refuser la connexion :**<br>- Admins Tier 1<br>- Admins Tier 2<br>- Comptes Standard | **Refuser la connexion :**<br>- Admins Tier 0<br>- Admins Tier 2 | **Refuser la connexion :**<br>- Admins Tier 0<br>- Admins Tier 1 |

### 3. PAW (Privileged Access Workstation)
Les Admins Tier 0 ne doivent administrer l'AD que depuis une station de travail ultra-securisee et dediee (PAW), qui ne navigue pas sur Internet.

## TP : Empecher un Domain Admin de se connecter sur un PC

**Objectif :** Proteger les identifiants Tier 0 d'un vol sur un poste compromis.

1.  Creez une GPO `SEC - Tier 2 - Restrictions`.
2.  Liez-la a l'OU contenant vos postes de travail.
3.  Allez dans : `Computer Configuration > Policies > Windows Settings > Security Settings > Local Policies > User Rights Assignment`.
4.  Configurez **Deny log on locally** et **Deny log on through Remote Desktop Services**.
5.  Ajoutez le groupe `Domain Admins` et `Enterprise Admins`.

**Resultat :** Si vous tentez d'ouvrir une session sur un PC Windows 10 avec le compte Administrateur du domaine, vous recevrez un message "La methode de connexion n'est pas autorisee". C'est une victoire pour la securite !

---

!!! example "Scenario pratique"

    **Contexte :** Un administrateur Tier 1 utilise son compte Tier 0 (Domain Admin) pour se connecter a un serveur applicatif Tier 1, violant le modele. L'equipe securite soupconne que des credentials privilegies ont ete exposes sur un systeme non protege.

    **Symptomes :**

    - Le SIEM detecte une connexion interactive d'un compte `T0-admin` sur `SRV-APP-01`
    - L'alerte indique un Logon Type 10 (RDP) depuis une adresse hors du reseau PAW

    **Diagnostic :**

    ```powershell
    # Verify if a Tier 0 account connected to a Tier 1 server
    Get-WinEvent -ComputerName "SRV-APP-01" -FilterHashtable @{
        LogName = 'Security'; Id = 4624
    } -MaxEvents 50 | ForEach-Object {
        $xml = [xml]$_.ToXml()
        $user = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'TargetUserName' } | Select-Object -ExpandProperty '#text'
        if ($user -like "T0-*") {
            [PSCustomObject]@{
                Time      = $_.TimeCreated
                Account   = $user
                LogonType = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'LogonType' } | Select-Object -ExpandProperty '#text'
                SourceIP  = $xml.Event.EventData.Data | Where-Object { $_.Name -eq 'IpAddress' } | Select-Object -ExpandProperty '#text'
            }
        }
    }
    ```

    Resultat :

    ```text
    Time                  Account     LogonType  SourceIP
    ----                  -------     ---------  --------
    2026-02-20 14:32:15   T0-admin    10         10.0.2.55
    2026-02-20 14:28:03   T0-admin    10         10.0.2.55
    ```

    Le compte Tier 0 `T0-admin` a ouvert deux sessions RDP depuis `10.0.2.55`, une adresse qui n'appartient pas au VLAN PAW (10.0.0.x). La violation du modele est confirmee.

    **Solution :**

    ```powershell
    # Configure Authentication Policy to restrict Tier 0 accounts to DCs only
    New-ADAuthenticationPolicy -Name "Tier0-RestrictToDC" `
        -UserAllowedToAuthenticateFrom "O:SYG:SYD:(XA;OICI;CR;;;WD;(@USER.ad://ext/AuthenticationSilo == `"Tier0-Silo`"))" `
        -Enforce $true

    # Create authentication policy silo for Tier 0 accounts
    New-ADAuthenticationPolicySilo -Name "Tier0-Silo" `
        -UserAuthenticationPolicy "Tier0-RestrictToDC" `
        -Enforce $true

    # Assign the Tier 0 admin to the silo
    Set-ADUser "T0-admin" -AuthenticationPolicySilo "Tier0-Silo"
    ```

    Resultat :

    ```text
    PS C:\> New-ADAuthenticationPolicy -Name "Tier0-RestrictToDC" ...
    PS C:\> New-ADAuthenticationPolicySilo -Name "Tier0-Silo" ...
    PS C:\> Set-ADUser "T0-admin" -AuthenticationPolicySilo "Tier0-Silo"
    ```

    Desormais, toute tentative de connexion du compte `T0-admin` sur un serveur non-DC est bloquee par la politique d'authentification AD. L'administrateur doit configurer une alerte sur l'Event ID 4820 pour detecter les futures violations.
