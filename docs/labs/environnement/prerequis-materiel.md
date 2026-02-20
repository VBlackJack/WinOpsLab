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
title: "Prerequis materiel"
description: Configuration materielle requise pour le lab Windows Server 2022 - CPU, RAM, disque et virtualisation imbriquee.
tags:
  - labs
  - environnement
  - debutant
---

# Prerequis materiel

<span class="level-beginner">Debutant</span> Â· Temps estime : 10 minutes

## Presentation

Ce lab utilise Hyper-V pour creer un environnement complet de formation Windows Server 2022. La machine hote (votre PC) doit disposer de ressources suffisantes pour executer plusieurs machines virtuelles simultanement.

## Configuration minimale

| Composant | Minimum | Recommande | Ideal |
|-----------|---------|------------|-------|
| **CPU** | 4 coeurs (VT-x/AMD-V) | 6 coeurs | 8+ coeurs |
| **RAM** | 16 Go | 32 Go | 64 Go |
| **Stockage** | 256 Go SSD | 512 Go NVMe | 1 To NVMe |
| **OS hote** | Windows 10/11 Pro | Windows 11 Pro | Windows 11 Pro |
| **Hyper-V** | Active | Active | Active |

!!! warning "Edition Windows"

    Hyper-V n'est pas disponible sur les editions **Home** de Windows 10/11.
    Vous devez utiliser l'edition **Pro**, **Enterprise** ou **Education**.

## Verifier la compatibilite du processeur

```powershell
# Check CPU virtualization support
Get-ComputerInfo | Select-Object CsProcessors,
    @{N='VirtualizationEnabled';E={
        (Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled
    }}

# Detailed CPU info
Get-CimInstance Win32_Processor | Select-Object Name, NumberOfCores,
    NumberOfLogicalProcessors, VirtualizationFirmwareEnabled

# Check if Hyper-V is available
Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V | Select-Object State
```

Le processeur doit supporter :

- **Intel VT-x** ou **AMD-V** : virtualisation materielle
- **SLAT** (Second Level Address Translation) : Intel EPT ou AMD RVI
- **DEP** (Data Execution Prevention) : bit NX/XD

## Activer Hyper-V

```powershell
# Enable Hyper-V role on Windows 10/11
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart

# Reboot required
Restart-Computer
```

## Virtualisation imbriquee (Nested Virtualization)

Si vous utilisez deja une VM comme machine hote (par exemple, dans un cloud), vous devez activer la virtualisation imbriquee.

```powershell
# Enable nested virtualization on a VM (run on the Hyper-V HOST, not inside the VM)
Set-VMProcessor -VMName "LabHost" -ExposeVirtualizationExtensions $true
```

!!! tip "Quand est-ce necessaire ?"

    La virtualisation imbriquee est utile si :

    - Votre machine hote est elle-meme une VM (Azure, VMware, etc.)
    - Vous voulez faire le Lab 06 (Hyper-V) a l'interieur du lab

## Estimation des ressources par VM

| VM | Role | vCPU | RAM | Disque |
|----|------|------|-----|--------|
| **SRV-DC01** | DC principal + DNS + DHCP | 2 | 4 Go | 60 Go |
| **SRV-DC02** | DC secondaire + DNS | 2 | 4 Go | 60 Go |
| **SRV-FILE01** | Serveur de fichiers | 2 | 4 Go | 80 Go |
| **SRV-WEB01** | Serveur web (IIS) | 2 | 4 Go | 60 Go |
| **SRV-CORE01** | Server Core (divers roles) | 2 | 2 Go | 40 Go |
| **CLI-W11** | Client Windows 11 | 2 | 4 Go | 60 Go |
| **Total** | | **12** | **22 Go** | **360 Go** |

!!! tip "Memoire dynamique"

    Activez la **memoire dynamique** sur les VMs pour optimiser l'utilisation de la RAM.
    Definissez la RAM de demarrage au minimum et laissez Hyper-V ajuster selon la charge.

## Stockage

### Recommandations

| Aspect | Recommandation |
|--------|---------------|
| **Type** | SSD NVMe (eviter les HDD mecaniques) |
| **Emplacement des VMs** | Disque separe du systeme si possible |
| **Format VHDX** | Dynamiquement extensible (economise l'espace) |
| **Organisation** | Un dossier par VM |

```powershell
# Create the directory structure for VMs
$vmBasePath = "G:\VMs"
$vms = @("SRV-DC01", "SRV-DC02", "SRV-FILE01", "SRV-WEB01", "SRV-CORE01", "CLI-W11")

foreach ($vm in $vms) {
    New-Item -Path "$vmBasePath\$vm" -ItemType Directory -Force
}
```

### ISO requises

| ISO | Usage | Source |
|-----|-------|--------|
| Windows Server 2022 Evaluation | Serveurs | Centre d'evaluation Microsoft |
| Windows 11 Enterprise Evaluation | Client | Centre d'evaluation Microsoft |

## Points cles a retenir

- Un minimum de **16 Go de RAM** est requis ; **32 Go** est recommande pour un lab confortable
- Un **SSD NVMe** est fortement recommande pour eviter les lenteurs de disque
- Hyper-V necessite Windows 10/11 **Pro** ou **Enterprise**
- La **memoire dynamique** permet d'optimiser l'utilisation de la RAM entre les VMs
- Prevoyez environ **360 Go** d'espace disque pour l'ensemble du lab
- La virtualisation imbriquee est necessaire uniquement si la machine hote est elle-meme une VM

## Pour aller plus loin

- [Architecture du lab](architecture-lab.md) pour le plan reseau et les roles des serveurs
- [Creation des VMs](creation-vms.md) pour automatiser le deploiement des machines virtuelles

