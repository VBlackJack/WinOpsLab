---
title: "Configuration des interfaces reseau"
description: "Configurer les interfaces reseau sous Windows Server 2022 via PowerShell, netsh et l'interface graphique."
tags:
  - reseau
  - powershell
  - configuration
  - windows-server
---

# Configuration des interfaces reseau

## Introduction

La configuration correcte des interfaces reseau (NIC - Network Interface Card) est une etape fondamentale dans la mise en place d'un serveur Windows. Que ce soit pour attribuer une adresse IP statique, configurer les serveurs DNS ou modifier les proprietes d'une carte reseau, Windows Server 2022 offre plusieurs approches : PowerShell, `netsh` et l'interface graphique.

!!! tip "Bonne pratique"

    Sur un serveur, il est recommande d'utiliser une adresse IP **statique** plutot que DHCP. Cela garantit la stabilite des services (DNS, AD DS, DHCP) qui dependent d'une adresse fixe.

---

## Decouvrir les interfaces disponibles

### Lister les cartes reseau

```powershell
# List all network adapters with their status
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, LinkSpeed

# List only connected adapters
Get-NetAdapter | Where-Object Status -eq "Up"
```

### Afficher la configuration IP actuelle

```powershell
# Display complete IP configuration for all interfaces
Get-NetIPConfiguration

# Display IP configuration for a specific interface
Get-NetIPConfiguration -InterfaceAlias "Ethernet0"

# Detailed view with DNS and gateway
Get-NetIPConfiguration -InterfaceAlias "Ethernet0" -Detailed
```

### Equivalent classique

```powershell
# Traditional command (still functional)
ipconfig /all
```

---

## Configuration via PowerShell (recommande)

### Attribuer une adresse IPv4 statique

```powershell
# Remove existing DHCP address first (if applicable)
Remove-NetIPAddress -InterfaceAlias "Ethernet0" -Confirm:$false

# Disable DHCP on the interface
Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Disabled

# Assign a static IPv4 address
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "192.168.1.10" `
    -PrefixLength 24 `
    -DefaultGateway "192.168.1.1"
```

### Configurer les serveurs DNS

```powershell
# Set primary and secondary DNS servers
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" `
    -ServerAddresses "192.168.1.1", "8.8.8.8"

# Verify DNS configuration
Get-DnsClientServerAddress -InterfaceAlias "Ethernet0" -AddressFamily IPv4
```

### Modifier une adresse existante

```powershell
# Change an existing IP address
Set-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "192.168.1.10" `
    -PrefixLength 24

# Change the default gateway
Remove-NetRoute -InterfaceAlias "Ethernet0" -DestinationPrefix "0.0.0.0/0" -Confirm:$false
New-NetRoute -InterfaceAlias "Ethernet0" `
    -DestinationPrefix "0.0.0.0/0" `
    -NextHop "192.168.1.254"
```

### Revenir en DHCP

```powershell
# Re-enable DHCP on the interface
Set-NetIPInterface -InterfaceAlias "Ethernet0" -Dhcp Enabled

# Reset DNS to automatic (DHCP-provided)
Set-DnsClientServerAddress -InterfaceAlias "Ethernet0" -ResetServerAddresses

# Force DHCP renewal
Invoke-Command -ScriptBlock {
    ipconfig /release
    ipconfig /renew
}
```

---

## Cmdlets PowerShell essentielles

### Get-NetIPAddress

```powershell
# Display all IPv4 addresses
Get-NetIPAddress -AddressFamily IPv4

# Display addresses for a specific interface
Get-NetIPAddress -InterfaceAlias "Ethernet0"

# Filter to show only manual (static) addresses
Get-NetIPAddress -AddressFamily IPv4 -PrefixOrigin Manual
```

### New-NetIPAddress

```powershell
# Add an additional IP address to an interface (IP aliasing)
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "192.168.1.20" `
    -PrefixLength 24

# Add an IPv6 address
New-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "2001:db8:1::10" `
    -PrefixLength 64
```

### Set-NetIPAddress

```powershell
# Modify an existing address (change prefix length only)
Set-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "192.168.1.10" `
    -PrefixLength 25
```

### Remove-NetIPAddress

```powershell
# Remove a specific IP address
Remove-NetIPAddress -InterfaceAlias "Ethernet0" `
    -IPAddress "192.168.1.20" `
    -Confirm:$false
```

---

## Gestion des proprietes de la carte reseau

### Renommer une interface

```powershell
# Rename a network adapter
Rename-NetAdapter -Name "Ethernet0" -NewName "LAN-Production"

# Verify the rename
Get-NetAdapter -Name "LAN-Production"
```

### Activer/desactiver une interface

```powershell
# Disable a network adapter
Disable-NetAdapter -Name "Ethernet0" -Confirm:$false

# Enable a network adapter
Enable-NetAdapter -Name "Ethernet0"
```

### Configurer les proprietes avancees

```powershell
# Display advanced properties of an adapter
Get-NetAdapterAdvancedProperty -Name "Ethernet0"

# Configure Jumbo Frames (example: 9014 bytes)
Set-NetAdapterAdvancedProperty -Name "Ethernet0" `
    -DisplayName "Jumbo Packet" -DisplayValue "9014"

# Configure VLAN ID
Set-NetAdapterAdvancedProperty -Name "Ethernet0" `
    -DisplayName "VLAN ID" -DisplayValue "100"

# Configure speed and duplex
Set-NetAdapterAdvancedProperty -Name "Ethernet0" `
    -DisplayName "Speed & Duplex" -DisplayValue "1.0 Gbps Full Duplex"
```

---

## Configuration via netsh

La commande `netsh` reste disponible et est parfois necessaire dans des scripts de compatibilite ou des environnements Server Core sans PowerShell complet.

### Afficher la configuration

```powershell
# Display IPv4 interface configuration
netsh interface ipv4 show config

# Display configuration for a specific interface
netsh interface ipv4 show config name="Ethernet0"

# Display all interfaces
netsh interface show interface
```

### Configurer une adresse statique

```powershell
# Set a static IP address
netsh interface ipv4 set address name="Ethernet0" static 192.168.1.10 255.255.255.0 192.168.1.1

# Set DNS servers
netsh interface ipv4 set dns name="Ethernet0" static 192.168.1.1 primary
netsh interface ipv4 add dns name="Ethernet0" 8.8.8.8 index=2
```

### Revenir en DHCP

```powershell
# Switch back to DHCP
netsh interface ipv4 set address name="Ethernet0" dhcp
netsh interface ipv4 set dns name="Ethernet0" dhcp
```

### Equivalences PowerShell / netsh

| Action                     | PowerShell                                        | netsh                                                  |
|----------------------------|---------------------------------------------------|--------------------------------------------------------|
| Afficher config IP         | `Get-NetIPConfiguration`                          | `netsh interface ipv4 show config`                     |
| Adresse statique           | `New-NetIPAddress`                                | `netsh interface ipv4 set address`                     |
| Configurer DNS             | `Set-DnsClientServerAddress`                      | `netsh interface ipv4 set dns`                         |
| Activer DHCP               | `Set-NetIPInterface -Dhcp Enabled`                | `netsh interface ipv4 set address dhcp`                |
| Lister interfaces          | `Get-NetAdapter`                                  | `netsh interface show interface`                       |

---

## Configuration via l'interface graphique

Pour les environnements avec interface Desktop Experience, la configuration se fait via le panneau **Connexions reseau** :

1. Ouvrir **Parametres reseau** : `ncpa.cpl` ou **Panneau de configuration > Reseau et Internet > Connexions reseau**
2. Clic droit sur l'interface > **Proprietes**
3. Selectionner **Protocole Internet version 4 (TCP/IPv4)** > **Proprietes**
4. Choisir **Utiliser l'adresse IP suivante** et renseigner :
    - Adresse IP
    - Masque de sous-reseau
    - Passerelle par defaut
5. Renseigner les serveurs DNS prefere et auxiliaire
6. Cliquer sur **OK** pour valider

!!! info "Server Core"

    Sur une installation Server Core (sans interface graphique), seules les commandes PowerShell et netsh sont disponibles. L'utilitaire `sconfig` offre egalement un menu simplifie pour la configuration reseau de base.

### Utiliser sconfig (Server Core)

```powershell
# Launch the Server Configuration tool
sconfig
```

Le menu **8) Parametres reseau** de `sconfig` permet de :

- Selectionner une interface
- Configurer une adresse statique ou DHCP
- Definir les serveurs DNS

---

## Diagnostics reseau

### Tests de connectivite

```powershell
# Ping a remote host
Test-Connection -ComputerName "192.168.1.1" -Count 4

# Traceroute
Test-Connection -ComputerName "8.8.8.8" -Traceroute

# Test DNS resolution
Resolve-DnsName -Name "www.microsoft.com"

# Test a specific TCP port
Test-NetConnection -ComputerName "192.168.1.1" -Port 443
```

### Verifier la table de routage

```powershell
# Display the IPv4 routing table
Get-NetRoute -AddressFamily IPv4 | Format-Table -AutoSize

# Display only the default gateway
Get-NetRoute -DestinationPrefix "0.0.0.0/0"
```

### Diagnostiquer les problemes de DNS

```powershell
# Display current DNS cache
Get-DnsClientCache | Select-Object -First 20

# Flush DNS cache
Clear-DnsClientCache

# Verify DNS server configuration
Get-DnsClientServerAddress -AddressFamily IPv4
```

---

## Script complet de configuration

Exemple d'un script PowerShell pour configurer entierement une interface reseau de serveur :

```powershell
# --- Server Network Configuration Script ---

# Configuration variables
$interfaceName = "Ethernet0"
$newName = "LAN-Production"
$ipAddress = "192.168.1.10"
$prefixLength = 24
$gateway = "192.168.1.1"
$dnsServers = @("192.168.1.1", "192.168.1.2")

# Rename the interface
Rename-NetAdapter -Name $interfaceName -NewName $newName

# Disable DHCP
Set-NetIPInterface -InterfaceAlias $newName -Dhcp Disabled

# Remove any existing IP configuration
Remove-NetIPAddress -InterfaceAlias $newName -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute -InterfaceAlias $newName -Confirm:$false -ErrorAction SilentlyContinue

# Assign static IP address and default gateway
New-NetIPAddress -InterfaceAlias $newName `
    -IPAddress $ipAddress `
    -PrefixLength $prefixLength `
    -DefaultGateway $gateway

# Configure DNS servers
Set-DnsClientServerAddress -InterfaceAlias $newName -ServerAddresses $dnsServers

# Verify final configuration
Get-NetIPConfiguration -InterfaceAlias $newName
```

---

## Points cles a retenir

| Concept               | Detail                                                        |
|-----------------------|---------------------------------------------------------------|
| Adresse statique      | Recommandee pour tous les serveurs                            |
| PowerShell            | Methode privilegiee : `New-NetIPAddress`, `Set-DnsClientServerAddress` |
| netsh                 | Alternative pour la compatibilite et Server Core              |
| sconfig               | Menu simplifie pour Server Core                               |
| Diagnostics           | `Test-Connection`, `Test-NetConnection`, `Resolve-DnsName`    |
| Renommage             | Renommer les interfaces pour la clarte (`Rename-NetAdapter`)  |

---

## Pour aller plus loin

- Comprendre l'adressage IPv4 : voir la page [Adressage IPv4](adressage-ipv4.md)
- Decoupage en sous-reseaux : voir la page [Sous-reseaux](sous-reseaux.md)
- Configurer IPv6 : voir la page [IPv6 Fondamentaux](ipv6-fondamentaux.md)
- Gestion du pare-feu : voir la section [Pare-feu Windows](../pare-feu/wfas-concepts.md)
