---
title: Depannage DNS
description: Methodologie de diagnostic DNS, outils, validation des enregistrements SRV et resolution des problemes courants.
tags:
  - active-directory
  - dns
  - avance
---

# Depannage DNS

<span class="level-advanced">Avance</span> · Temps estime : 40 minutes

## Methodologie de diagnostic

!!! example "Analogie"

    Le depannage DNS ressemble a la recherche d'une panne dans un systeme de distribution du courrier. On commence par verifier si le facteur peut physiquement acceder a la boite aux lettres (connectivite reseau), puis si l'adresse est correcte sur l'enveloppe (configuration DNS du client), ensuite si le bureau de poste est ouvert (service DNS), et enfin si le destinataire est bien repertorie dans l'annuaire (enregistrement DNS).

Face a un probleme de resolution DNS dans un environnement Active Directory, suivez cette approche methodique :

```mermaid
graph TD
    A[Probleme de resolution DNS] --> B{Le client peut-il<br/>pinguer le DC par IP ?}
    B -->|Non| C[Probleme reseau<br/>Verifier IP, passerelle, pare-feu]
    B -->|Oui| D{Le client pointe-t-il<br/>vers le bon DNS ?}
    D -->|Non| E[Corriger la config DNS<br/>du client]
    D -->|Oui| F{nslookup fonctionne-t-il<br/>depuis le client ?}
    F -->|Non| G{Le service DNS<br/>est-il demarre ?}
    G -->|Non| H[Demarrer le service DNS]
    G -->|Oui| I{La zone est-elle<br/>chargee ?}
    I -->|Non| J[Verifier l'etat de la zone<br/>et la replication AD]
    I -->|Oui| K{L'enregistrement<br/>existe-t-il ?}
    K -->|Non| L[Creer l'enregistrement<br/>ou verifier les mises a jour dynamiques]
    K -->|Oui| M[Verifier le cache DNS<br/>client et serveur]
    F -->|Oui| N[DNS fonctionne<br/>Verifier le service applicatif]
```

## Outils de diagnostic

### nslookup

L'outil classique de requete DNS, disponible sur tous les systemes Windows :

```powershell
# Basic query
nslookup srv-dc01.lab.local

# Query a specific DNS server
nslookup srv-dc01.lab.local 192.168.1.10

# Interactive mode - useful for multiple queries
nslookup
> server 192.168.1.10
> set type=SRV
> _ldap._tcp.lab.local
> set type=A
> srv-dc01.lab.local
> exit

# Query a specific record type
nslookup -type=MX lab.local
nslookup -type=SOA lab.local
nslookup -type=NS lab.local

# Reverse lookup
nslookup 192.168.1.10

# Enable debug output for detailed response analysis
nslookup -debug srv-dc01.lab.local
```

Resultat :

```text
PS> nslookup srv-dc01.lab.local
Server:   DC-01.lab.local
Address:  10.0.0.10

Name:     srv-dc01.lab.local
Address:  10.0.0.10

PS> nslookup -type=NS lab.local
Server:   DC-01.lab.local
Address:  10.0.0.10

lab.local
        primary name server = dc-01.lab.local
        responsible mail addr = hostmaster.lab.local
        serial  = 285
        refresh = 900 (15 mins)
        retry   = 600 (10 mins)
        expire  = 86400 (1 day)
        default TTL = 3600 (1 hour)
```

!!! tip "nslookup ignore le cache client"

    `nslookup` interroge directement le serveur DNS et ne consulte pas le cache
    du client Windows. Si `nslookup` fonctionne mais que `ping` par nom echoue,
    le probleme vient probablement du cache DNS local.

### Resolve-DnsName (PowerShell)

La cmdlet PowerShell moderne, plus flexible et puissante que nslookup :

```powershell
# Basic resolution
Resolve-DnsName -Name "srv-dc01.lab.local"

# Specify record type
Resolve-DnsName -Name "lab.local" -Type MX
Resolve-DnsName -Name "lab.local" -Type SOA
Resolve-DnsName -Name "lab.local" -Type NS

# Query a specific DNS server
Resolve-DnsName -Name "srv-dc01.lab.local" -Server "192.168.1.10"

# DNS only (skip LLMNR and NetBIOS)
Resolve-DnsName -Name "srv-dc01.lab.local" -DnsOnly

# TCP instead of UDP (useful for large responses)
Resolve-DnsName -Name "lab.local" -Type ANY -TcpOnly

# Reverse lookup
Resolve-DnsName -Name "192.168.1.10" -Type PTR

# Check SRV records for AD services
Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV
Resolve-DnsName -Name "_kerberos._tcp.lab.local" -Type SRV
Resolve-DnsName -Name "_gc._tcp.lab.local" -Type SRV
```

### dcdiag /test:dns

L'outil de diagnostic Active Directory inclut un test DNS complet qui verifie l'ensemble de la configuration DNS necessaire au bon fonctionnement d'AD :

```powershell
# Full DNS test on the local DC
dcdiag /test:dns /v

# DNS test targeting a specific DC
dcdiag /test:dns /s:SRV-DC01 /v

# Test specific DNS sub-checks
dcdiag /test:dns /DnsBasic /s:SRV-DC01
dcdiag /test:dns /DnsForwarders /s:SRV-DC01
dcdiag /test:dns /DnsDelegation /s:SRV-DC01
dcdiag /test:dns /DnsDynamicUpdate /s:SRV-DC01
dcdiag /test:dns /DnsRecordRegistration /s:SRV-DC01

# DNS test on all DCs in the domain
dcdiag /test:dns /v /e
```

Resultat :

```text
Domain Controller Diagnosis

Performing initial setup:
   Trying to find home server...
   Home Server = DC-01
   * Identified AD Forest.
   Done gathering initial info.

Doing initial required tests
   Testing server: Default-First-Site-Name\DC-01
      Starting test: Connectivity
         ......................... DC-01 passed test Connectivity

Doing primary tests
   Testing server: Default-First-Site-Name\DC-01
      Starting test: DNS
         DNS Tests are running and not hung. Please wait a few minutes...
         ......................... DC-01 passed test DNS

         Summary of DNS test results:
                                          Auth Basc Forw Del  Dyn  RReg Ext
         _________________________________________________________________
         Domain: lab.local
           DC-01                          PASS PASS PASS PASS PASS PASS n/a
```

Les sous-tests DNS de dcdiag :

| Sous-test | Verification |
|-----------|-------------|
| **DnsBasic** | Connectivite DNS et configuration de base |
| **DnsForwarders** | Redirecteurs et resolution externe |
| **DnsDelegation** | Delegation DNS correcte |
| **DnsDynamicUpdate** | Mises a jour dynamiques fonctionnelles |
| **DnsRecordRegistration** | Enregistrements SRV et A du DC |
| **DnsResolveExtName** | Resolution de noms externes |
| **DnsAll** | Tous les sous-tests |

### Autres outils utiles

```powershell
# Test DNS resolution AND connectivity
Test-Connection -ComputerName "srv-dc01.lab.local" -Count 2

# View the client DNS configuration
Get-DnsClientServerAddress | Format-Table InterfaceAlias, ServerAddresses

# View DNS suffix search list
Get-DnsClient | Select-Object InterfaceAlias, ConnectionSpecificSuffix, UseSuffixWhenRegistering

# Test port 53 connectivity to a DNS server
Test-NetConnection -ComputerName "192.168.1.10" -Port 53

# Check the DNS server service status
Get-Service -Name DNS -ComputerName "SRV-DC01"

# View DNS server event log
Get-WinEvent -LogName "DNS Server" -ComputerName "SRV-DC01" -MaxEvents 20

# Check DNS server statistics
Get-DnsServerStatistics -ComputerName "SRV-DC01"
```

Resultat :

```text
PS> Test-NetConnection -ComputerName "10.0.0.10" -Port 53

ComputerName     : 10.0.0.10
RemoteAddress    : 10.0.0.10
RemotePort       : 53
InterfaceAlias   : Ethernet
TcpTestSucceeded : True

PS> Get-Service -Name DNS -ComputerName "DC-01"

Status   Name     DisplayName
------   ----     -----------
Running  DNS      DNS Server

PS> Get-DnsClientServerAddress | Format-Table InterfaceAlias, ServerAddresses

InterfaceAlias  ServerAddresses
--------------  ---------------
Ethernet        {10.0.0.10, 10.0.0.11}
Loopback        {}
```

## Gestion du cache DNS

### Cache client

Chaque poste Windows conserve un cache DNS local pour accelerer les resolutions repetees :

```powershell
# View the DNS client cache
Get-DnsClientCache

# View cache entries for a specific name
Get-DnsClientCache | Where-Object { $_.Entry -like "*srv-dc01*" }

# Flush the entire DNS client cache
Clear-DnsClientCache

# Register the client's DNS records (force dynamic update)
Register-DnsClient
```

### Cache serveur

Le serveur DNS conserve egalement un cache des requetes resolues pour d'autres domaines :

```powershell
# View the server-side DNS cache
Show-DnsServerCache -ComputerName "SRV-DC01"

# Clear the server-side DNS cache
Clear-DnsServerCache -ComputerName "SRV-DC01" -Force

# View cache settings (max TTL, etc.)
Get-DnsServerCache -ComputerName "SRV-DC01"

# Set maximum cache TTL (example: 1 day)
Set-DnsServerCache -MaxTtl 1.00:00:00 -ComputerName "SRV-DC01"
```

!!! warning "Vider le cache resout souvent les problemes de resolution"

    Quand un enregistrement DNS a ete modifie mais que la resolution retourne
    l'ancienne valeur, videz le cache client (`Clear-DnsClientCache`) puis le
    cache serveur (`Clear-DnsServerCache`) si necessaire. Le TTL controle
    la duree de vie des entrees en cache.

## Validation des enregistrements SRV

Les enregistrements SRV sont la pierre angulaire de la localisation des services AD. Leur absence ou leur erreur provoque des dysfonctionnements graves.

### Verification rapide

```powershell
# Verify all critical SRV records for the domain
$domain = "lab.local"
$srvRecords = @(
    "_ldap._tcp.$domain",
    "_kerberos._tcp.$domain",
    "_gc._tcp.$domain",
    "_kpasswd._tcp.$domain",
    "_ldap._tcp.dc._msdcs.$domain"
)

foreach ($record in $srvRecords) {
    Write-Host "`n--- $record ---" -ForegroundColor Cyan
    try {
        Resolve-DnsName -Name $record -Type SRV -DnsOnly -ErrorAction Stop |
            Format-Table Name, Type, NameTarget, Port, Priority -AutoSize
    }
    catch {
        Write-Host "  ERREUR : Enregistrement non trouve !" -ForegroundColor Red
    }
}
```

### Verification via le fichier Netlogon.dns

Chaque DC genere un fichier contenant les enregistrements SRV qu'il doit enregistrer :

```powershell
# View the expected SRV records for a DC
Get-Content "C:\Windows\System32\config\netlogon.dns" | Select-Object -First 30

# Compare with what is actually registered in DNS
$expectedRecords = Get-Content "C:\Windows\System32\config\netlogon.dns" |
    Where-Object { $_ -match "^_" }
Write-Host "Number of SRV records expected: $($expectedRecords.Count)"
```

### Re-enregistrer les SRV manquants

Si des enregistrements SRV sont absents, forcez leur re-enregistrement :

```powershell
# Restart the Netlogon service (re-registers all SRV records)
Restart-Service -Name Netlogon

# Or force re-registration via nltest
nltest /dsregdns

# Verify the registration was successful
Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -DnsOnly
```

Resultat :

```text
PS> nltest /dsregdns
The command completed successfully

PS> Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -DnsOnly

Name                                    Type   TTL   Section    NameTarget              Priority Weight Port
----                                    ----   ---   -------    ----------              -------- ------ ----
_ldap._tcp.lab.local                    SRV    600   Answer     DC-01.lab.local         0        100    389
_ldap._tcp.lab.local                    SRV    600   Answer     SRV-DC01.lab.local      0        100    389
```

!!! danger "Enregistrements SRV manquants"

    Si les enregistrements SRV ne se re-enregistrent pas apres un redemarrage de
    Netlogon, verifiez :

    - Que la zone DNS autorise les mises a jour dynamiques
    - Que le DC pointe vers lui-meme (ou un autre DC) comme serveur DNS
    - Que le service DNS est operationnel
    - Que la replication AD fonctionne correctement

## Problemes courants et solutions

### Probleme 1 : Le client ne peut pas joindre le domaine

**Symptome** : le message "Le domaine specifie n'existe pas ou n'est pas joignable" apparait lors de la jonction.

**Diagnostic** :

```powershell
# Check DNS configuration on the client
Get-DnsClientServerAddress -InterfaceAlias "Ethernet"

# Try to resolve the domain
Resolve-DnsName -Name "lab.local" -Type A -DnsOnly

# Check SRV records
Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -DnsOnly
```

**Solutions** :

- Verifier que le client utilise un serveur DNS du domaine (pas un DNS externe)
- Verifier que le suffixe DNS est correctement configure
- S'assurer que les enregistrements SRV existent dans la zone

### Probleme 2 : Replication AD en echec

**Symptome** : `repadmin /replsummary` montre des erreurs de replication avec des messages DNS.

**Diagnostic** :

```powershell
# Check replication status
repadmin /replsummary

# Test DNS between DCs
Resolve-DnsName -Name "srv-dc02.lab.local" -Server "SRV-DC01" -DnsOnly
Resolve-DnsName -Name "srv-dc01.lab.local" -Server "SRV-DC02" -DnsOnly

# Verify each DC can resolve the other's SRV records
Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -Server "SRV-DC01"
Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -Server "SRV-DC02"
```

**Solutions** :

- Verifier que chaque DC pointe vers un serveur DNS qui connait la zone AD
- Verifier que les enregistrements A de chaque DC sont presents
- Forcer le re-enregistrement DNS : `nltest /dsregdns` sur chaque DC

### Probleme 3 : Resolution lente ou intermittente

**Symptome** : les requetes DNS prennent plusieurs secondes ou echouent de maniere aleatoire.

**Diagnostic** :

```powershell
# Measure DNS resolution time
Measure-Command { Resolve-DnsName -Name "srv-dc01.lab.local" -DnsOnly }

# Check if the DNS server is overloaded
Get-DnsServerStatistics -ComputerName "SRV-DC01" |
    Select-Object -ExpandProperty QueryStatistics

# Check the DNS server event log for errors
Get-WinEvent -LogName "DNS Server" -ComputerName "SRV-DC01" -MaxEvents 50 |
    Where-Object { $_.LevelDisplayName -eq "Error" -or $_.LevelDisplayName -eq "Warning" }
```

**Solutions** :

- Verifier les redirecteurs (le serveur tente peut-etre des redirecteurs inaccessibles)
- Augmenter le nombre de serveurs DNS
- Verifier la charge du serveur (CPU, memoire, reseau)
- Vider le cache DNS si des enregistrements obsoletes causent des erreurs

### Probleme 4 : Enregistrements DNS en doublon ou obsoletes

**Symptome** : une adresse IP est associee a plusieurs noms, ou un nom pointe vers une ancienne adresse.

**Diagnostic** :

```powershell
# Find duplicate A records for the same name
Get-DnsServerResourceRecord -ZoneName "lab.local" -Name "srv-app01" -RRType A -ComputerName "SRV-DC01"

# Find all records pointing to a specific IP
Get-DnsServerResourceRecord -ZoneName "lab.local" -RRType A -ComputerName "SRV-DC01" |
    Where-Object { $_.RecordData.IPv4Address -eq "192.168.1.50" }

# Check scavenging status
Get-DnsServerScavenging -ComputerName "SRV-DC01"
Get-DnsServerZoneAging -Name "lab.local" -ComputerName "SRV-DC01"
```

**Solutions** :

- Supprimer les enregistrements obsoletes manuellement
- Activer le [scavenging](enregistrements.md#nettoyage-automatique-scavenging) sur le serveur et la zone
- Verifier que les clients DHCP enregistrent correctement leurs noms

### Probleme 5 : La zone DNS ne se charge pas

**Symptome** : la zone apparait avec une icone d'erreur dans DNS Manager ou `Get-DnsServerZone` retourne une erreur.

**Diagnostic** :

```powershell
# Check the zone status
Get-DnsServerZone -Name "lab.local" -ComputerName "SRV-DC01"

# Check DNS server events
Get-WinEvent -LogName "DNS Server" -ComputerName "SRV-DC01" -MaxEvents 20

# For AD-integrated zones, check AD replication
repadmin /replsummary
dcdiag /test:replications /s:SRV-DC01
```

**Solutions** :

- Pour une zone integree AD : verifier la replication AD et la partition DomainDnsZones
- Pour une zone fichier : verifier les permissions sur le fichier de zone dans `%SystemRoot%\System32\dns\`
- Redemarrer le service DNS : `Restart-Service DNS`

## Script de diagnostic complet

Ce script effectue une verification complete de l'etat DNS d'un environnement AD :

```powershell
# Comprehensive DNS health check script
param(
    [string]$DomainName = (Get-ADDomain).DNSRoot,
    [string]$DCName = $env:COMPUTERNAME
)

Write-Host "=== DNS Health Check - $DomainName ===" -ForegroundColor Green
Write-Host "Target DC: $DCName`n"

# 1. DNS service status
Write-Host "--- DNS Service Status ---" -ForegroundColor Cyan
$dnsService = Get-Service -Name DNS -ComputerName $DCName
Write-Host "  Service: $($dnsService.Status)"

# 2. Zone status
Write-Host "`n--- Zone Status ---" -ForegroundColor Cyan
Get-DnsServerZone -ComputerName $DCName |
    Where-Object { $_.IsAutoCreated -eq $false } |
    Format-Table ZoneName, ZoneType, IsDsIntegrated, DynamicUpdate -AutoSize

# 3. Critical SRV records
Write-Host "--- SRV Record Validation ---" -ForegroundColor Cyan
$srvTests = @(
    @{ Name = "LDAP"; Query = "_ldap._tcp.$DomainName" },
    @{ Name = "Kerberos"; Query = "_kerberos._tcp.$DomainName" },
    @{ Name = "GC"; Query = "_gc._tcp.$DomainName" },
    @{ Name = "PDC"; Query = "_ldap._tcp.pdc._msdcs.$DomainName" }
)

foreach ($test in $srvTests) {
    try {
        $result = Resolve-DnsName -Name $test.Query -Type SRV -DnsOnly -ErrorAction Stop
        Write-Host "  [OK] $($test.Name): $($result.Count) record(s)" -ForegroundColor Green
    }
    catch {
        Write-Host "  [FAIL] $($test.Name): No records found" -ForegroundColor Red
    }
}

# 4. Forwarders
Write-Host "`n--- Forwarders ---" -ForegroundColor Cyan
$forwarders = Get-DnsServerForwarder -ComputerName $DCName
if ($forwarders.IPAddress) {
    foreach ($fw in $forwarders.IPAddress) {
        $reachable = Test-NetConnection -ComputerName $fw -Port 53 -WarningAction SilentlyContinue
        $status = if ($reachable.TcpTestSucceeded) { "[OK]" } else { "[FAIL]" }
        $color = if ($reachable.TcpTestSucceeded) { "Green" } else { "Red" }
        Write-Host "  $status $fw" -ForegroundColor $color
    }
} else {
    Write-Host "  No forwarders configured (using Root Hints)"
}

# 5. Scavenging status
Write-Host "`n--- Scavenging Status ---" -ForegroundColor Cyan
$scavenging = Get-DnsServerScavenging -ComputerName $DCName
Write-Host "  Enabled: $($scavenging.ScavengingState)"
Write-Host "  Interval: $($scavenging.ScavengingInterval)"

# 6. Recent DNS errors
Write-Host "`n--- Recent DNS Errors (last 24h) ---" -ForegroundColor Cyan
$errors = Get-WinEvent -LogName "DNS Server" -ComputerName $DCName -MaxEvents 100 |
    Where-Object { $_.LevelDisplayName -eq "Error" -and $_.TimeCreated -gt (Get-Date).AddDays(-1) }
if ($errors) {
    Write-Host "  $($errors.Count) error(s) found:" -ForegroundColor Yellow
    $errors | Select-Object -First 5 | ForEach-Object {
        Write-Host "    [$($_.TimeCreated)] $($_.Message.Substring(0, [Math]::Min(100, $_.Message.Length)))" -ForegroundColor Yellow
    }
} else {
    Write-Host "  No errors in the last 24 hours" -ForegroundColor Green
}

Write-Host "`n=== Check Complete ===" -ForegroundColor Green
```

## Commandes de reference rapide

| Action | Commande |
|--------|----------|
| Tester la resolution | `Resolve-DnsName -Name "nom" -Type A` |
| Requete vers un serveur specifique | `Resolve-DnsName -Name "nom" -Server "IP"` |
| Vider le cache client | `Clear-DnsClientCache` |
| Vider le cache serveur | `Clear-DnsServerCache -ComputerName "DC"` |
| Forcer l'enregistrement DNS | `Register-DnsClient` |
| Re-enregistrer les SRV du DC | `nltest /dsregdns` |
| Test DNS complet du DC | `dcdiag /test:dns /v` |
| Verifier le service DNS | `Get-Service DNS -ComputerName "DC"` |
| Voir les evenements DNS | `Get-WinEvent -LogName "DNS Server"` |
| Voir la config DNS du client | `Get-DnsClientServerAddress` |

!!! example "Scenario pratique"

    **Situation** : Lundi matin, Pierre, technicien de support, recoit de nombreux appels : les utilisateurs du site de Paris ne peuvent plus se connecter au domaine. Les sessions existantes fonctionnent encore, mais les nouvelles connexions echouent avec "Aucun serveur d'authentification disponible".

    **Diagnostic** :

    ```powershell
    # Etape 1 : Depuis un poste client, verifier la config DNS
    Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
    ```

    Resultat : les postes pointent vers `10.0.0.10` (DC-01) et `10.0.0.11` (SRV-DC01).

    ```powershell
    # Etape 2 : Tester la resolution des SRV records
    Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -DnsOnly
    ```

    Resultat : erreur "DNS name does not exist". Les enregistrements SRV sont absents.

    ```powershell
    # Etape 3 : Verifier le service DNS sur les DC
    Get-Service -Name DNS -ComputerName "DC-01"
    Get-Service -Name DNS -ComputerName "SRV-DC01"
    ```

    Resultat : le service DNS est arrete sur les deux DC suite a une mise a jour Windows qui a echoue durant le week-end.

    **Solution** :

    ```powershell
    # Redemarrer le service DNS sur les deux DC
    Start-Service -Name DNS -ComputerName "DC-01"
    Start-Service -Name DNS -ComputerName "SRV-DC01"

    # Forcer le re-enregistrement des SRV
    Invoke-Command -ComputerName "DC-01","SRV-DC01" -ScriptBlock {
        Restart-Service -Name Netlogon
    }

    # Verifier que les SRV sont de nouveau presents
    Resolve-DnsName -Name "_ldap._tcp.lab.local" -Type SRV -DnsOnly

    # Valider avec dcdiag
    dcdiag /test:dns /DnsRecordRegistration /s:DC-01
    ```

    Les utilisateurs peuvent a nouveau ouvrir de nouvelles sessions. Pierre programme une surveillance automatique du service DNS pour detecter ce type de panne plus rapidement.

!!! danger "Erreurs courantes"

    - **Oublier de vider le cache client apres une modification DNS** : un enregistrement modifie sur le serveur peut rester en cache sur le client pendant la duree du TTL. Utilisez `Clear-DnsClientCache` sur le poste pour forcer la mise a jour immediate.
    - **Diagnostiquer avec ping au lieu de nslookup ou Resolve-DnsName** : `ping` utilise le cache DNS local et la resolution NetBIOS, ce qui peut fausser le diagnostic. Utilisez `Resolve-DnsName -DnsOnly` pour tester exclusivement la resolution DNS.
    - **Configurer les postes du domaine avec des DNS publics en premier** : si le DNS principal d'un poste est `8.8.8.8`, il ne trouvera jamais les enregistrements SRV de `lab.local`. Le DNS primaire doit toujours etre un DC du domaine.
    - **Ne pas verifier les deux sens de la resolution entre DC** : lors d'un probleme de replication, il faut verifier que chaque DC peut resoudre le nom de l'autre. Un seul sens fonctionnel ne suffit pas.
    - **Ignorer les journaux d'evenements DNS** : les erreurs recurrentes dans le journal "DNS Server" sont souvent le premier signe d'un probleme sous-jacent (zone corrompue, replication en echec, espace disque insuffisant).

## Points cles a retenir

- Suivez une methodologie systematique : reseau > configuration client > service DNS > zone > enregistrement
- `Resolve-DnsName` est preferable a `nslookup` pour le diagnostic en PowerShell
- `dcdiag /test:dns` verifie l'ensemble de la configuration DNS requise par AD
- Les enregistrements SRV sont critiques : utilisez `nltest /dsregdns` pour les re-enregistrer
- Videz le cache client ET serveur quand un changement DNS n'est pas pris en compte
- Consultez toujours le journal d'evenements DNS pour les erreurs recurrentes

## Pour aller plus loin

- [Concepts DNS](concepts-dns.md) -- revoir les fondamentaux du DNS
- [Enregistrements DNS](enregistrements.md) -- comprendre les types d'enregistrements et le scavenging
- [Resolution conditionnelle](resolution-conditionnelle.md) -- diagnostiquer les redirecteurs
- [Depannage GPO](../gpo/gpresult-et-depannage.md) -- de nombreux problemes GPO sont lies au DNS
- [Methodologie de depannage generale](../../supervision/depannage/methodologie.md) -- approche structuree du diagnostic
