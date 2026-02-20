---
title: "Templates ARM et Bicep"
description: "Deployer des machines virtuelles Windows Server sur Azure avec les templates ARM et Bicep : structure, parametres, deploiement et bonnes pratiques."
tags:
  - automatisation
  - iac
  - arm
  - bicep
  - azure
  - windows-server
---

# Templates ARM et Bicep

<span class="level-advanced">Avance</span> · Temps estime : 50 minutes

## Introduction

Les templates **ARM** (Azure Resource Manager) et **Bicep** sont les outils d'**Infrastructure as Code** (IaC) natifs d'Azure. Ils permettent de definir l'infrastructure de maniere declarative et de la deployer de facon reproductible. Bicep est le successeur de ARM JSON : il offre une syntaxe plus lisible tout en compilant vers ARM en arriere-plan.

## ARM vs Bicep

| Critere | ARM (JSON) | Bicep |
|---|---|---|
| Format | JSON verbeux | Syntaxe concise et lisible |
| Courbe d'apprentissage | Raide (JSON imbrique) | Douce (syntaxe dediee) |
| IntelliSense | Limite | Excellent (VS Code extension) |
| Compilation | Direct | Compile vers ARM JSON |
| Modules | Linked templates (complexe) | Modules natifs (simple) |
| Etat | Mature, stable | Recommande par Microsoft |

!!! tip "Bicep est le choix recommande"

    Microsoft recommande Bicep pour tous les nouveaux projets IaC Azure. Bicep compile vers ARM JSON et offre toutes les memes fonctionnalites avec une syntaxe nettement plus lisible.

!!! example "Analogie"

    Un template ARM ou Bicep, c'est comme un plan d'architecte pour une maison. Le plan decrit precisement chaque piece, chaque porte, chaque fenetre. N'importe quel constructeur peut prendre ce plan et construire une maison identique, que ce soit en France, en Belgique ou au Canada. Et si vous voulez construire deux maisons identiques, vous utilisez exactement le meme plan — pas besoin de re-inventer les fondations.

## Structure d'un template ARM

Un template ARM est un fichier JSON avec quatre sections principales :

```mermaid
flowchart TD
    A[Template ARM] --> B[parameters<br/>Variables d'entree]
    A --> C[variables<br/>Valeurs calculees]
    A --> D[resources<br/>Ressources a deployer]
    A --> E[outputs<br/>Valeurs de sortie]
```

### Exemple ARM : VM Windows Server

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string",
      "metadata": {
        "description": "Name of the virtual machine"
      }
    },
    "adminUsername": {
      "type": "string",
      "metadata": {
        "description": "Admin username for the VM"
      }
    },
    "adminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Admin password for the VM"
      }
    },
    "vmSize": {
      "type": "string",
      "defaultValue": "Standard_D2s_v5",
      "allowedValues": [
        "Standard_D2s_v5",
        "Standard_D4s_v5",
        "Standard_D8s_v5"
      ]
    }
  },
  "variables": {
    "nicName": "[concat(parameters('vmName'), '-nic')]",
    "vnetName": "[concat(parameters('vmName'), '-vnet')]",
    "subnetName": "default",
    "publicIPName": "[concat(parameters('vmName'), '-pip')]"
  },
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachines",
      "apiVersion": "2023-03-01",
      "name": "[parameters('vmName')]",
      "location": "[resourceGroup().location]",
      "properties": {
        "hardwareProfile": {
          "vmSize": "[parameters('vmSize')]"
        },
        "osProfile": {
          "computerName": "[parameters('vmName')]",
          "adminUsername": "[parameters('adminUsername')]",
          "adminPassword": "[parameters('adminPassword')]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "MicrosoftWindowsServer",
            "offer": "WindowsServer",
            "sku": "2022-datacenter-g2",
            "version": "latest"
          },
          "osDisk": {
            "createOption": "FromImage",
            "managedDisk": {
              "storageAccountType": "Premium_LRS"
            }
          }
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
            }
          ]
        }
      },
      "dependsOn": [
        "[resourceId('Microsoft.Network/networkInterfaces', variables('nicName'))]"
      ]
    }
  ],
  "outputs": {
    "vmResourceId": {
      "type": "string",
      "value": "[resourceId('Microsoft.Compute/virtualMachines', parameters('vmName'))]"
    }
  }
}
```

## Structure d'un template Bicep

La meme VM en Bicep est nettement plus concise :

### Exemple Bicep : VM Windows Server

```bicep
// parameters
@description('Name of the virtual machine')
param vmName string

@description('Admin username for the VM')
param adminUsername string

@description('Admin password for the VM')
@secure()
param adminPassword string

@description('VM size')
@allowed([
  'Standard_D2s_v5'
  'Standard_D4s_v5'
  'Standard_D8s_v5'
])
param vmSize string = 'Standard_D2s_v5'

@description('Azure region for the deployment')
param location string = resourceGroup().location

// variables
var nicName = '${vmName}-nic'
var vnetName = '${vmName}-vnet'
var subnetName = 'default'
var publicIPName = '${vmName}-pip'

// resources
resource publicIP 'Microsoft.Network/publicIPAddresses@2023-04-01' = {
  name: publicIPName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource vnet 'Microsoft.Network/virtualNetworks@2023-04-01' = {
  name: vnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.0.0.0/16'
      ]
    }
    subnets: [
      {
        name: subnetName
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
    ]
  }
}

resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = {
  name: nicName
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: vnet.properties.subnets[0].id
          }
          publicIPAddress: {
            id: publicIP.id
          }
        }
      }
    ]
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: {
      vmSize: vmSize
    }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: '2022-datacenter-g2'
        version: 'latest'
      }
      osDisk: {
        createOption: 'FromImage'
        managedDisk: {
          storageAccountType: 'Premium_LRS'
        }
      }
    }
    networkProfile: {
      networkInterfaces: [
        {
          id: nic.id
        }
      ]
    }
  }
}

// outputs
output vmResourceId string = vm.id
output publicIPAddress string = publicIP.properties.ipAddress
```

## Deploiement

### Deployer un template ARM

```powershell
# Deploy ARM template
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg-yourproject" `
    -TemplateFile ".\templates\vm-windows.json" `
    -TemplateParameterFile ".\parameters\vm-windows.parameters.json" `
    -Verbose
```

Resultat :

```text
VERBOSE: Performing the operation "Creating Deployment" on target "rg-yourproject".
VERBOSE: 11:42:03 - Template is valid.
VERBOSE: 11:42:05 - Creating the deployment 'vm-windows-20260220-114203'
VERBOSE: 11:42:05 - Resource Microsoft.Network/publicIPAddresses 'SRV-WEB01-pip' provisioning status is running
VERBOSE: 11:46:12 - Resource Microsoft.Compute/virtualMachines 'SRV-WEB01' provisioning status is succeeded

DeploymentName          : vm-windows-20260220-114203
ResourceGroupName       : rg-yourproject
ProvisioningState       : Succeeded
Timestamp               : 20/02/2026 10:46:12
```

### Deployer un template Bicep

```powershell
# Deploy Bicep template (Azure CLI)
az deployment group create `
    --resource-group "rg-yourproject" `
    --template-file ".\templates\vm-windows.bicep" `
    --parameters vmName="SRV-WEB01" adminUsername="youradmin"

# Deploy Bicep template (PowerShell Az module)
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg-yourproject" `
    -TemplateFile ".\templates\vm-windows.bicep" `
    -vmName "SRV-WEB01" `
    -adminUsername "youradmin" `
    -Verbose
```

### Fichier de parametres

```json
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "value": "SRV-WEB01"
    },
    "adminUsername": {
      "value": "youradmin"
    },
    "vmSize": {
      "value": "Standard_D2s_v5"
    }
  }
}
```

!!! danger "Mots de passe dans les fichiers de parametres"

    Ne stockez jamais de mots de passe en clair dans les fichiers de parametres. Utilisez un **Azure Key Vault** pour les secrets.

```json
{
  "adminPassword": {
    "reference": {
      "keyVault": {
        "id": "/subscriptions/YOUR_SUB/resourceGroups/YOUR_RG/providers/Microsoft.KeyVault/vaults/YOUR_KV"
      },
      "secretName": "vmAdminPassword"
    }
  }
}
```

### Mode What-If (simulation)

```powershell
# Preview changes without deploying
New-AzResourceGroupDeployment `
    -ResourceGroupName "rg-yourproject" `
    -TemplateFile ".\templates\vm-windows.bicep" `
    -vmName "SRV-WEB01" `
    -WhatIf
```

Resultat :

```text
Resource and property changes are indicated with these symbols:
  + Create
  ~ Modify

The deployment will update the following scope:

Scope: /subscriptions/xxxx/resourceGroups/rg-yourproject

  + Microsoft.Network/publicIPAddresses/SRV-WEB01-pip [2023-04-01]
  + Microsoft.Network/virtualNetworks/SRV-WEB01-vnet [2023-04-01]
  + Microsoft.Network/networkInterfaces/SRV-WEB01-nic [2023-04-01]
  + Microsoft.Compute/virtualMachines/SRV-WEB01 [2023-03-01]

Resource changes: 4 to create.
```

## Modules Bicep

Les modules permettent de reutiliser des composants d'infrastructure.

```bicep
// modules/windows-vm.bicep
@description('Deploy a Windows Server VM')
param vmName string
param location string
param vmSize string
param adminUsername string
@secure()
param adminPassword string
param subnetId string

resource nic 'Microsoft.Network/networkInterfaces@2023-04-01' = {
  name: '${vmName}-nic'
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: subnetId
          }
        }
      }
    ]
  }
}

resource vm 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: vmName
  location: location
  properties: {
    hardwareProfile: { vmSize: vmSize }
    osProfile: {
      computerName: vmName
      adminUsername: adminUsername
      adminPassword: adminPassword
    }
    storageProfile: {
      imageReference: {
        publisher: 'MicrosoftWindowsServer'
        offer: 'WindowsServer'
        sku: '2022-datacenter-g2'
        version: 'latest'
      }
      osDisk: { createOption: 'FromImage' }
    }
    networkProfile: {
      networkInterfaces: [ { id: nic.id } ]
    }
  }
}

output vmId string = vm.id
```

```bicep
// main.bicep - using the module
module webServer 'modules/windows-vm.bicep' = {
  name: 'deploy-web-server'
  params: {
    vmName: 'SRV-WEB01'
    location: resourceGroup().location
    vmSize: 'Standard_D2s_v5'
    adminUsername: 'youradmin'
    adminPassword: adminPassword
    subnetId: vnet.properties.subnets[0].id
  }
}
```

!!! example "Scenario pratique"

    **Contexte :** Antoine est ingenieur cloud dans une societe qui doit regulierement provisionner des environnements de test pour les developpeurs. Chaque environnement comprend un serveur Windows Server 2022 avec une IP publique. Actuellement, chaque environnement est cree manuellement dans le portail Azure, ce qui prend 30 a 45 minutes.

    **Probleme :** Les environnements manuels different les uns des autres (tailles VM incorrectes, regions differentes, noms inconsistants), et les developpeurs doivent attendre Antoine pour obtenir un environnement.

    **Solution :** Antoine cree un template Bicep parametre et le stocke dans Git. Les developpeurs peuvent maintenant se provisionner eux-memes :

    ```powershell
    # Step 1: Validate the Bicep template
    az bicep build --file .\templates\vm-windows.bicep
    ```

    ```text
    (La commande genere vm-windows.json sans erreur si le template est valide)
    ```

    ```powershell
    # Step 2: What-if before deploying
    New-AzResourceGroupDeployment `
        -ResourceGroupName "rg-dev-antoine-20260220" `
        -TemplateFile ".\templates\vm-windows.bicep" `
        -vmName "SRV-DEV01" `
        -adminUsername "labadmin" `
        -WhatIf
    ```

    ```text
    Resource changes: 4 to create.
      + SRV-DEV01-pip     (publicIPAddress)
      + SRV-DEV01-vnet    (virtualNetwork)
      + SRV-DEV01-nic     (networkInterface)
      + SRV-DEV01         (virtualMachine - Standard_D2s_v5)
    ```

    ```powershell
    # Step 3: Deploy (password from Key Vault)
    New-AzResourceGroupDeployment `
        -ResourceGroupName "rg-dev-antoine-20260220" `
        -TemplateFile ".\templates\vm-windows.bicep" `
        -vmName "SRV-DEV01" `
        -adminUsername "labadmin" `
        -TemplateParameterFile ".\parameters\dev.parameters.json"
    ```

    ```text
    DeploymentName    : vm-windows-20260220-143012
    ProvisioningState : Succeeded
    Timestamp         : 20/02/2026 12:34:45
    Outputs           :
      vmResourceId    : /subscriptions/.../virtualMachines/SRV-DEV01
      publicIPAddress : 51.105.22.47
    ```

    Le provisionnement complet prend maintenant 8 minutes au lieu de 45. Antoine partage le template dans le wiki interne et les developpeurs s'autonomisent.

!!! danger "Erreurs courantes"

    **Mot de passe en clair dans le fichier de parametres** — Ne stockez jamais `adminPassword` en clair dans un fichier `.parameters.json` commite dans Git. Utilisez une reference Key Vault ou passez le parametre interactivement via `-adminPassword (Read-Host -AsSecureString)`.

    **API version obsolete** — Azure retire regulierement les anciennes versions d'API. Si votre template utilise une `apiVersion` obsolete, le deploiement echoue. Verifiez les versions supportees dans la documentation ou via `az provider show`.

    **Deploiement sans mode What-If** — Deployer directement sans `-WhatIf` dans un environnement existant peut modifier ou supprimer des ressources non prevues. Utilisez systematiquement What-If pour les ressources en production.

    **Dependances implicites manquantes** — En ARM JSON, les dependances doivent etre declarcees explicitement avec `dependsOn`. En Bicep, les dependances symboliques sont automatiques (reference directe a la ressource), mais si vous copiez des blocs JSON vers Bicep, verifiez que les dependances sont bien gerees.

    **Region non supportee pour le type de VM** — Certaines tailles de VM (`Standard_D*v5`) ne sont pas disponibles dans toutes les regions Azure. Verifiez la disponibilite avant de deployer ou parametrez la region dynamiquement.

## Points cles a retenir

- **Bicep** est le choix recommande pour les nouveaux projets IaC Azure (compile vers ARM JSON)
- Les templates sont **declaratifs** et **idempotents** : ils peuvent etre re-deployes sans effet secondaire
- Utilisez **Azure Key Vault** pour les secrets (mots de passe, cles)
- Le mode **What-If** permet de previsualiser les changements avant deploiement
- Les **modules Bicep** favorisent la reutilisation et la maintenabilite
- Versionnez vos templates dans un depot Git pour la tracabilite

## Pour aller plus loin

- Packer pour les images : [Packer](packer-images.md)
- DSC pour la configuration : [Concepts DSC](../dsc/concepts-dsc.md)
- Documentation Microsoft : Bicep Documentation
