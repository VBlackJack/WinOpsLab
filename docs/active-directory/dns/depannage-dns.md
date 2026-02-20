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
title: Depannage DNS
description: Methodologie de diagnostic DNS, outils, validation des enregistrements SRV et resolution des problemes courants.
tags:
  - active-directory
  - dns
  - avance
  - powershell
---

# Depannage DNS

<span class="level-advanced">Avance</span> · Temps estime : 40 minutes

## Memento des Commandes de Survie

Avant de plonger dans les theories, voici les commandes indispensables pour survivre a un probleme DNS.

| Action | Commande PowerShell (Recommande) | Ancienne commande (CMD) |
|--------|----------------------------------|-------------------------|
| **Vider le cache client** | `Clear-DnsClientCache` | `ipconfig /flushdns` |
| **Tester la resolution** | `Resolve-DnsName google.fr` | `nslookup google.fr` |
| **Forcer l'enregistrement** | `Register-DnsClient` | `ipconfig /registerdns` |
| **Voir les serveurs DNS** | `Get-DnsClientServerAddress` | `ipconfig /all` |
| **Tester un port (TCP)** | `Test-NetConnection 10.0.0.1 -Port 53` | *N/A (telnet)* |

---

## Methodologie de diagnostic

Le depannage DNS doit suivre une logique implacable, du client vers le serveur, puis vers l'exterieur.

```mermaid
graph TD
    A[Probleme de resolution DNS] --> B{Ping 8.8.8.8 OK ?}
    B -- Non --> C[Probleme RESEAU<br/>(Cable, Routeur, IP)]
    B -- Oui --> D{Resolve-DnsName<br/>google.fr OK ?}
    D -- Oui --> E[Probleme specifique<br/>au domaine local]
    D -- Non --> F{Serveur DNS client<br/>= DC local ?}
    F -- Non --> G[Mauvaise Config IP<br/>Client pointe vers 8.8.8.8 ?]
    F -- Oui --> H{Service DNS du DC<br/>demarre ?}
    H -- Non --> I[Demarrer Service DNS]
    H -- Oui --> J[Verifier Forwarders<br/>sur le DC]
    
    E --> K{Enregistrements SRV<br/>presents ?}
    K -- Non --> L[Redemarrer Netlogon]
    K -- Oui --> M[Verifier Replication AD]

    style C fill:#f44336,color:#fff
    style G fill:#ff9800,color:#fff
    style I fill:#4caf50,color:#fff
    style L fill:#2196f3,color:#fff
```

## Outil d'Auto-Diagnostic (Script)

Ne perdez plus de temps a taper 10 commandes. Copiez ce script sur un contrôleur de domaine pour auditer la sante de votre DNS en 2 secondes.

```powershell
# --- Script d'Audit DNS WinOpsLab ---
$DomainName = (Get-ADDomain).DNSRoot
$Validation = [Ordered]@{}

Write-Host "Audit DNS pour le domaine : $DomainName" -ForegroundColor Cyan

# 1. Service DNS
$Service = Get-Service -Name DNS -ErrorAction SilentlyContinue
$Validation["Service DNS Actif"] = if ($Service.Status -eq 'Running') { "OK" } else { "ARRETE" }

# 2. Resolution Locale (A)
try {
    $SelfPing = Resolve-DnsName -Name $env:COMPUTERNAME -Type A -ErrorAction Stop
    $Validation["Resolution Locale (A)"] = "OK ($($SelfPing.IPAddress))"
} catch {
    $Validation["Resolution Locale (A)"] = "ECHEC"
}

# 3. Enregistrements SRV (Critique AD)
try {
    $SRV = Resolve-DnsName -Name "_ldap._tcp.dc._msdcs.$DomainName" -Type SRV -ErrorAction Stop
    $Validation["Enregistrements SRV (AD)"] = "OK ($($SRV.Count) trouves)"
} catch {
    $Validation["Enregistrements SRV (AD)"] = "MANQUANTS (Critique!)"
}

# 4. Resolution Externe (Forwarders)
try {
    $Ext = Resolve-DnsName -Name "google.com" -Type A -ErrorAction Stop
    $Validation["Resolution Internet"] = "OK"
} catch {
    $Validation["Resolution Internet"] = "ECHEC (Verifier Forwarders)"
}

# 5. Scavenging (Nettoyage)
$Scavenge = Get-DnsServerScavenging
$Validation["Nettoyage Auto (Scavenging)"] = if ($Scavenge.ScavengingState) { "ACTIF" } else { "INACTIF (Risque de pollution)" }

$Validation | Format-Table -AutoSize
```

---

## Challenge "Break/Fix" : Panne Internet Totale

!!! failure "Le cas pratique"

    **Contexte :** Vous gerez le lab `WinOpsLab`. Tout fonctionnait hier. Ce matin, plus aucun serveur ne peut acceder a Internet pour telecharger des mises a jour. Le reseau local fonctionne parfaitement (ping entre serveurs OK).

    **Symptôme :**
    Sur `DC-01` :
    - `Ping 8.8.8.8` -> Reponse : OK (Donc Internet physique fonctionne).
    - `Resolve-DnsName google.fr` -> Erreur : *DNS name does not exist*.

    **Diagnostic guide :**
    
    1.  Le serveur DNS local ne sait pas resoudre les noms publics.
    2.  Il n'est pas autoritaire pour `google.fr`, donc il doit transmettre la requete.
    3.  A qui transmet-il la requete ? Aux **Redirecteurs (Forwarders)** ou aux **Racines**.

    ??? tip "Solution (Cliquez pour reveler)"
        
        Le probleme vient probablement des **Redirecteurs**.

        **Verification :**
        ```powershell
        Get-DnsServerForwarder
        ```
        *Si la liste est vide ou contient des IP invalides, c'est la cause.*

        **Correction :**
        Ajouter les DNS de Google ou Cloudflare comme redirecteurs :
        ```powershell
        Set-DnsServerForwarder -IPAddress "1.1.1.1", "8.8.8.8"
        
        # Vider le cache pour tester immediatement
        Clear-DnsServerCache
        Resolve-DnsName google.fr
        ```

## Problemes courants et solutions

### 1. "Le domaine n'existe pas" (Jonction impossible)
*   **Cause :** Le client pointe vers sa Box ou `8.8.8.8` au lieu du DC.
*   **Fix :** `Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "10.0.0.10"`

### 2. Enregistrements SRV manquants
*   **Cause :** Le DC a demarre avant le service DNS, ou la zone n'accepte pas les mises a jour dynamiques.
*   **Fix :** 
    ```powershell
    # Redemarrer le service qui gere l'enregistrement
    Restart-Service Netlogon
    # Ou forcer via commande
    nltest /dsregdns
    ```

### 3. Pollution DNS (Vieux enregistrements)
*   **Cause :** Le "Scavenging" (nettoyage) n'est pas active. Les vieilles IP de PC disparus restent dans la zone.
*   **Fix :** Activer le Scavenging sur le Serveur ET sur la Zone. (Voir section *Concepts DNS*).

## Pour aller plus loin

- [Concepts DNS](concepts-dns.md) -- Comprendre la recursivite.
- [Memento Ports AD](../../ressources/memento-ports-ad.md) -- Verifier que le port 53 est ouvert.
