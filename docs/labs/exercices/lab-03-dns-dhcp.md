---
title: "Lab 03 : DNS et DHCP"
description: Exercice pratique - configurer les zones DNS, creer une etendue DHCP et tester la resolution de noms et l'attribution d'adresses.
tags:
  - lab
  - dns
  - dhcp
  - debutant
---

# Lab 03 : DNS et DHCP

!!! abstract "Objectifs du lab"

    - [ ] Configurer une zone de recherche inversee DNS
    - [ ] Creer des enregistrements DNS manuels (A, CNAME, MX)
    - [ ] Installer et configurer le role DHCP
    - [ ] Creer une etendue DHCP avec options
    - [ ] Tester la resolution DNS et l'attribution DHCP depuis CLI-W11

## Scenario

Le domaine winopslab.local est en place. Vous devez maintenant configurer les services DNS et DHCP pour que les postes clients obtiennent automatiquement leur configuration reseau et resolvent les noms du domaine.

## Environnement requis

| Ressource | Specification |
|-----------|---------------|
| SRV-DC01 | DC + DNS (Lab 02 termine) |
| SRV-DC02 | DC secondaire + DNS |
| CLI-W11 | Client Windows 11 joint au domaine |
| Reseau | LabSwitch, 192.168.10.0/24 |

## Instructions

### Partie 1 : Zone de recherche inversee

1. Creer une zone de recherche inversee pour le reseau 192.168.10.0/24
2. Configurer la zone comme integree a AD et repliquee sur tous les DC DNS

??? success "Solution"

    ```powershell
    # On SRV-DC01: Create reverse lookup zone
    Add-DnsServerPrimaryZone -NetworkId "192.168.10.0/24" `
        -ReplicationScope "Forest" -DynamicUpdate "Secure"

    # Verify the zone
    Get-DnsServerZone | Where-Object { $_.IsReverseLookupZone }

    # Create PTR records for existing servers
    Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" `
        -Name "10" -PtrDomainName "SRV-DC01.winopslab.local"
    Add-DnsServerResourceRecordPtr -ZoneName "10.168.192.in-addr.arpa" `
        -Name "11" -PtrDomainName "SRV-DC02.winopslab.local"
    ```

### Partie 2 : Enregistrements DNS manuels

1. Creer un enregistrement A pour `intranet.winopslab.local` pointant vers SRV-WEB01 (192.168.10.30)
2. Creer un alias CNAME `www` pointant vers `intranet.winopslab.local`
3. Creer un enregistrement MX pour le domaine

??? success "Solution"

    ```powershell
    # Create A record
    Add-DnsServerResourceRecordA -ZoneName "winopslab.local" `
        -Name "intranet" -IPv4Address "192.168.10.30" -CreatePtr

    # Create CNAME record
    Add-DnsServerResourceRecordCName -ZoneName "winopslab.local" `
        -Name "www" -HostNameAlias "intranet.winopslab.local"

    # Create MX record (mail exchanger)
    Add-DnsServerResourceRecordMX -ZoneName "winopslab.local" `
        -Name "." -MailExchange "mail.winopslab.local" -Preference 10

    # Verify records
    Get-DnsServerResourceRecord -ZoneName "winopslab.local" |
        Where-Object { $_.HostName -in "intranet", "www" }

    # Test resolution
    Resolve-DnsName intranet.winopslab.local
    Resolve-DnsName www.winopslab.local
    ```

### Partie 3 : Installer le role DHCP

1. Installer le role DHCP sur SRV-DC01
2. Autoriser le serveur DHCP dans Active Directory

??? success "Solution"

    ```powershell
    # Install DHCP role
    Install-WindowsFeature -Name DHCP -IncludeManagementTools

    # Authorize DHCP server in Active Directory
    Add-DhcpServerInDC -DnsName "SRV-DC01.winopslab.local" -IPAddress 192.168.10.10

    # Verify authorization
    Get-DhcpServerInDC

    # Complete post-install configuration (security groups)
    Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\ServerManager\Roles\12" `
        -Name "ConfigurationState" -Value 2
    ```

### Partie 4 : Creer une etendue DHCP

1. Creer une etendue pour le reseau 192.168.10.0/24
2. Plage : 192.168.10.100 a 192.168.10.200
3. Exclure la plage 192.168.10.150 a 192.168.10.160 (reserve)
4. Configurer les options (passerelle, DNS, domaine)

??? success "Solution"

    ```powershell
    # Create the DHCP scope
    Add-DhcpServerv4Scope -Name "LAN-Lab" `
        -StartRange 192.168.10.100 -EndRange 192.168.10.200 `
        -SubnetMask 255.255.255.0 `
        -LeaseDuration (New-TimeSpan -Days 8) `
        -State Active

    # Add exclusion range
    Add-DhcpServerv4ExclusionRange -ScopeId 192.168.10.0 `
        -StartRange 192.168.10.150 -EndRange 192.168.10.160

    # Configure scope options
    Set-DhcpServerv4OptionValue -ScopeId 192.168.10.0 `
        -Router 192.168.10.1 `
        -DnsServer 192.168.10.10, 192.168.10.11 `
        -DnsDomain "winopslab.local"

    # Verify the scope
    Get-DhcpServerv4Scope
    Get-DhcpServerv4OptionValue -ScopeId 192.168.10.0
    ```

### Partie 5 : Tester depuis le client

1. Sur CLI-W11, configurer l'interface en DHCP
2. Verifier l'obtention d'une adresse IP
3. Tester la resolution DNS

??? success "Solution"

    ```powershell
    # On CLI-W11: Set interface to DHCP
    Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
    Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses

    # Force DHCP renewal
    ipconfig /release
    ipconfig /renew

    # Verify IP configuration
    ipconfig /all

    # Test DNS resolution
    Resolve-DnsName SRV-DC01.winopslab.local
    Resolve-DnsName intranet.winopslab.local
    nslookup www.winopslab.local

    # Test reverse DNS
    Resolve-DnsName 192.168.10.10

    # Test connectivity
    Test-NetConnection SRV-DC01 -Port 53
    ```

## Verification

!!! question "Questions de validation"

    1. Quelle est la difference entre une zone integree a AD et une zone fichier ?
    2. Pourquoi faut-il autoriser le serveur DHCP dans AD ?
    3. A quoi sert l'exclusion dans une etendue DHCP ?
    4. Quelle commande force le renouvellement d'un bail DHCP ?

??? success "Reponses"

    1. Une zone integree a AD stocke les enregistrements dans la base AD et beneficie
       de la replication AD multi-maitre. Une zone fichier est stockee dans un fichier
       .dns et necessite un transfert de zone pour la replication.
    2. L'autorisation dans AD empeche les serveurs DHCP non autorises de distribuer des
       adresses IP. C'est une mesure de securite contre les serveurs DHCP illegitimes.
    3. L'exclusion reserve des adresses dans la plage pour un usage specifique (serveurs,
       imprimantes) sans que le DHCP les distribue a des clients.
    4. `ipconfig /renew` (CMD) ou `Invoke-CimMethod -ClassName Win32_NetworkAdapterConfiguration -MethodName RenewDHCPLease` (PowerShell).

## Nettoyage

Conservez l'environnement pour les labs suivants.

## Prochaine etape

:material-arrow-right: [Lab 04 : Strategies de groupe (GPO)](lab-04-gpo.md)
