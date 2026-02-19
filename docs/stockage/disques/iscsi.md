---
title: iSCSI
description: Comprendre et configurer le stockage iSCSI sous Windows Server 2022 - initiateur, cible, IQN, LUN et MPIO.
tags:
  - stockage
  - disques
  - iscsi
  - intermediaire
---

# iSCSI

!!! info "Niveau : Intermediaire"

    Temps estime : 35 minutes

## Concepts fondamentaux

### Qu'est-ce que iSCSI ?

iSCSI (Internet Small Computer Systems Interface) est un protocole qui encapsule des commandes SCSI dans des paquets TCP/IP. Il permet d'acceder a du stockage distant via le reseau Ethernet standard, comme si les disques etaient connectes localement.

```mermaid
graph LR
    A[Serveur<br/>Initiateur iSCSI] -->|Reseau TCP/IP| B[Baie de stockage<br/>Cible iSCSI]
    B --> C[LUN 0<br/>100 Go]
    B --> D[LUN 1<br/>200 Go]
    B --> E[LUN 2<br/>500 Go]
```

### Terminologie cle

| Terme | Description |
|-------|-------------|
| **Initiateur** (Initiator) | Le client qui accede au stockage distant. Composant logiciel ou carte HBA dediee. |
| **Cible** (Target) | Le serveur qui expose le stockage. Peut etre un SAN, un NAS ou un serveur Windows. |
| **IQN** (iSCSI Qualified Name) | Identifiant unique et mondial d'un noeud iSCSI (format : `iqn.yyyy-mm.com.domaine:nom`) |
| **LUN** (Logical Unit Number) | Numero logique identifiant un disque virtuel expose par la cible |
| **Portail** (Portal) | Adresse IP et port TCP (par defaut 3260) de la cible |
| **Session** | Connexion etablie entre un initiateur et une cible |
| **CHAP** | Challenge Handshake Authentication Protocol, pour l'authentification |
| **MPIO** | Multipath I/O, pour la redundance et l'equilibrage de charge reseau |

### Architecture typique

```mermaid
graph TB
    subgraph "Reseau de stockage (iSCSI)"
        direction LR
        T1[Cible iSCSI<br/>Windows Server]
    end

    subgraph "Serveurs applicatifs"
        S1[Serveur Web<br/>Initiateur]
        S2[Serveur SQL<br/>Initiateur]
        S3[Cluster Hyper-V<br/>Initiateur]
    end

    S1 -->|LUN 0| T1
    S2 -->|LUN 1| T1
    S3 -->|LUN 2 & 3| T1
```

!!! tip "Reseau dedie"

    En production, isolez le trafic iSCSI sur un VLAN ou un reseau physique dedie pour eviter la contention avec le trafic applicatif. Utilisez des trames Jumbo (MTU 9000) pour de meilleures performances.

## Configurer une cible iSCSI (iSCSI Target)

Le role **iSCSI Target Server** transforme un serveur Windows en cible de stockage.

### Installer le role

```powershell
# Install the iSCSI Target Server role
Install-WindowsFeature FS-iSCSITarget-Server -IncludeManagementTools
```

### Creer un disque virtuel iSCSI

Un disque virtuel iSCSI est un fichier VHDX stocke sur le serveur cible :

```powershell
# Create a 100 GB iSCSI virtual disk (thin provisioned by default)
New-IscsiVirtualDisk -Path "E:\iSCSIDisks\SQL-Data.vhdx" -SizeBytes 100GB
```

Options de provisioning :

```powershell
# Fixed size (pre-allocated, better performance)
New-IscsiVirtualDisk -Path "E:\iSCSIDisks\SQL-Data.vhdx" -SizeBytes 100GB -UseFixed

# Dynamic (thin provisioned, saves space)
New-IscsiVirtualDisk -Path "E:\iSCSIDisks\SQL-Logs.vhdx" -SizeBytes 50GB
```

### Creer une cible iSCSI

La cible definit quels initiateurs sont autorises a se connecter :

```powershell
# Create an iSCSI target allowing a specific initiator by IQN
New-IscsiServerTarget -TargetName "SQL-Target" `
    -InitiatorIds @("IQN:iqn.1991-05.com.microsoft:srv-sql.lab.local")
```

Autoriser un initiateur par adresse IP :

```powershell
# Allow connection by IP address
New-IscsiServerTarget -TargetName "HyperV-Target" `
    -InitiatorIds @("IPAddress:192.168.10.20", "IPAddress:192.168.10.21")
```

### Assigner un disque virtuel a une cible

```powershell
# Map the virtual disk to the target (assigns LUN automatically)
Add-IscsiVirtualDiskTargetMapping -TargetName "SQL-Target" `
    -Path "E:\iSCSIDisks\SQL-Data.vhdx"

# Map with a specific LUN number
Add-IscsiVirtualDiskTargetMapping -TargetName "SQL-Target" `
    -Path "E:\iSCSIDisks\SQL-Logs.vhdx" -Lun 1
```

### Verifier la configuration de la cible

```powershell
# List all iSCSI targets
Get-IscsiServerTarget | Format-Table TargetName, Status, InitiatorIds -AutoSize

# List all iSCSI virtual disks
Get-IscsiVirtualDisk | Format-Table Path,
    @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
    Status -AutoSize

# List target-to-disk mappings
Get-IscsiVirtualDiskTargetMapping | Format-Table TargetName, Path, Lun -AutoSize
```

## Configurer l'initiateur iSCSI

L'initiateur iSCSI est le composant cote client qui se connecte a la cible.

### Demarrer le service Initiateur

```powershell
# Start and enable the iSCSI Initiator service
Set-Service -Name MSiSCSI -StartupType Automatic
Start-Service MSiSCSI

# Verify the service is running
Get-Service MSiSCSI | Select-Object Name, Status, StartType
```

### Decouvrir une cible

```powershell
# Discover targets on the portal (IP of the target server)
New-IscsiTargetPortal -TargetPortalAddress 192.168.10.10

# List discovered targets
Get-IscsiTarget | Format-Table NodeAddress, IsConnected -AutoSize
```

### Se connecter a une cible

```powershell
# Connect to the target (non-persistent, lost after reboot)
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target"

# Connect with persistence (reconnects automatically after reboot)
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target" `
    -IsPersistent $true
```

### Connexion avec authentification CHAP

```powershell
# Connect with CHAP authentication
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target" `
    -AuthenticationType ONEWAYCHAP `
    -ChapUsername "sql-server" `
    -ChapSecret "P@ssw0rdCH@P123!" `
    -IsPersistent $true
```

!!! warning "Securite CHAP"

    CHAP fournit une authentification basique mais transmet le secret en clair a travers le reseau. Pour une securite accrue, combinez CHAP avec IPSec ou isolez le trafic iSCSI sur un reseau dedie.

### Verifier la connexion et utiliser le disque

Une fois connecte, le LUN distant apparait comme un disque local :

```powershell
# List iSCSI sessions
Get-IscsiSession | Format-Table InitiatorNodeAddress, TargetNodeAddress,
    IsConnected, IsPersistent -AutoSize

# The new disk appears as a local disk
Get-Disk | Where-Object { $_.BusType -eq "iSCSI" } |
    Format-Table Number, FriendlyName,
        @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
        OperationalStatus -AutoSize

# Initialize, partition, and format the iSCSI disk
$disk = Get-Disk | Where-Object { $_.BusType -eq "iSCSI" -and $_.PartitionStyle -eq "RAW" }
$disk | Initialize-Disk -PartitionStyle GPT -PassThru |
    New-Partition -UseMaximumSize -AssignDriveLetter |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "SQL-Data" -Confirm:$false
```

### Se deconnecter d'une cible

```powershell
# Disconnect from a target
Disconnect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target" -Confirm:$false

# Remove the target portal
Remove-IscsiTargetPortal -TargetPortalAddress 192.168.10.10 -Confirm:$false
```

## MPIO (Multipath I/O)

MPIO permet d'utiliser plusieurs chemins reseau vers une meme cible iSCSI, assurant la redundance et l'equilibrage de charge.

### Architecture MPIO

```mermaid
graph LR
    subgraph "Serveur initiateur"
        NIC1[NIC 1<br/>192.168.10.20]
        NIC2[NIC 2<br/>192.168.11.20]
    end

    subgraph "Cible iSCSI"
        P1[Portal 1<br/>192.168.10.10]
        P2[Portal 2<br/>192.168.11.10]
        LUN[LUN 0]
    end

    NIC1 -->|Chemin 1| P1
    NIC2 -->|Chemin 2| P2
    P1 --> LUN
    P2 --> LUN
```

### Installer et configurer MPIO

```powershell
# Install the Multipath I/O feature
Install-WindowsFeature Multipath-IO -IncludeManagementTools

# A reboot is required after installation
Restart-Computer -Force
```

Apres le redemarrage :

```powershell
# Add iSCSI support to MPIO
Enable-MSDSMAutomaticClaim -BusType iSCSI

# A second reboot may be required
Restart-Computer -Force
```

### Configurer les chemins multiples

```powershell
# Connect to the target via the first path
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target" `
    -TargetPortalAddress 192.168.10.10 `
    -InitiatorPortalAddress 192.168.10.20 `
    -IsMultipathEnabled $true `
    -IsPersistent $true

# Connect via the second path
Connect-IscsiTarget -NodeAddress "iqn.1991-05.com.microsoft:srv-stor-sql-target" `
    -TargetPortalAddress 192.168.11.10 `
    -InitiatorPortalAddress 192.168.11.20 `
    -IsMultipathEnabled $true `
    -IsPersistent $true
```

### Politiques d'equilibrage de charge

```powershell
# View current MPIO load balance policy
Get-MSDSMGlobalDefaultLoadBalancePolicy

# Set Round Robin policy (recommended for iSCSI)
Set-MSDSMGlobalDefaultLoadBalancePolicy -Policy RR
```

Politiques disponibles :

| Politique | Description |
|-----------|-------------|
| **FOO** (Fail Over Only) | Un chemin actif, les autres en standby |
| **RR** (Round Robin) | Alternance entre les chemins (recommande) |
| **LQD** (Least Queue Depth) | Chemin avec le moins de requetes en attente |
| **WP** (Weighted Paths) | Poids attribue a chaque chemin |

### Verifier MPIO

```powershell
# Check MPIO configuration
Get-MPIOSetting

# List MPIO disks and their paths
Get-MSDSMSupportedHW | Format-Table VendorId, ProductId -AutoSize

# Verify paths for a specific disk (via Disk Management or mpclaim)
mpclaim -s -d
```

## Bonnes pratiques en production

### Reseau

- Dedie un VLAN ou un reseau physique au trafic iSCSI
- Activez les trames Jumbo (MTU 9000) sur les cartes reseau iSCSI
- Utilisez MPIO avec au minimum deux chemins physiques independants
- Desactivez le mode veille des cartes reseau iSCSI

### Securite

- Activez l'authentification CHAP ou CHAP mutuel
- Combinez avec IPSec si le reseau n'est pas physiquement isole
- Limitez l'acces aux cibles par IQN plutot que par adresse IP

### Performances

```powershell
# Disable power management on iSCSI network adapters
Get-NetAdapter | Where-Object { $_.Name -like "*iSCSI*" } |
    ForEach-Object {
        Set-NetAdapterPowerManagement -Name $_.Name -WakeOnMagicPacket Disabled
    }

# Set Jumbo Frames on iSCSI adapters
Set-NetAdapterAdvancedProperty -Name "iSCSI-NIC1" `
    -RegistryKeyword "*JumboPacket" -RegistryValue 9014
```

## Points cles a retenir

- **iSCSI** transporte des commandes SCSI sur TCP/IP, permettant l'acces a du stockage distant via Ethernet
- Le role **iSCSI Target Server** transforme un serveur Windows en baie de stockage
- L'**initiateur** se connecte aux cibles ; les LUN distants apparaissent comme des disques locaux
- Les connexions **persistantes** (`-IsPersistent $true`) survivent aux redemarrages
- **MPIO** assure la redundance et l'equilibrage de charge avec plusieurs chemins reseau
- En production, isolez le trafic iSCSI sur un reseau dedie avec trames Jumbo
- Utilisez **CHAP** pour l'authentification et **IPSec** pour le chiffrement

## Pour aller plus loin

- [Types de disques](types-de-disques.md)
- [Volumes et partitions](volumes-et-partitions.md)
- [Storage Spaces : concepts](../storage-spaces/concepts.md)
- [Storage Spaces Direct](../storage-spaces/storage-spaces-direct.md)
