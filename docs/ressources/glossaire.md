---
title: "Glossaire"
description: Glossaire des termes Windows Server - Active Directory, DNS, DHCP, GPO, FSMO et tous les acronymes essentiels.
tags:
  - ressources
  - reference
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

# Glossaire

<span class="level-beginner">Reference</span> · Temps estime : 5 minutes

---

Ce glossaire regroupe les termes et acronymes essentiels utilises dans l'administration Windows Server.

## A

| Terme | Definition |
|-------|-----------|
| **ACL** (Access Control List) | Liste de controle d'acces definissant les permissions sur un objet (fichier, dossier, cle de registre) |
| **AD CS** (Active Directory Certificate Services) | Role Windows Server fournissant une infrastructure de cles publiques (PKI) |
| **AD DS** (Active Directory Domain Services) | Service d'annuaire de Microsoft pour la gestion centralisee des identites et des ressources |
| **ADMT** (Active Directory Migration Tool) | Outil Microsoft pour la migration d'objets entre domaines AD |
| **AGDLP** | Modele de permissions : Account > Global Group > Domain Local Group > Permission |
| **AMA** (Azure Monitor Agent) | Agent unifie de collecte de telemetrie pour Azure Monitor |
| **Azure Arc** | Service Azure projetant des ressources on-premises dans le plan de gestion Azure |

## B

| Terme | Definition |
|-------|-----------|
| **Baseline** | Reference de performance etablie en conditions normales, utilisee pour detecter les anomalies |
| **BLG** | Format de fichier binaire des logs de performance Windows |
| **BSOD** (Blue Screen of Death) | Ecran bleu indiquant une erreur critique du noyau Windows |

## C

| Terme | Definition |
|-------|-----------|
| **CA** (Certificate Authority) | Autorite de certification delivrant et gerant les certificats numeriques |
| **CAL** (Client Access License) | Licence d'acces client requise pour se connecter a un serveur Windows |
| **CRL** (Certificate Revocation List) | Liste des certificats revoques publiee par une CA |
| **CSR** (Certificate Signing Request) | Demande de signature de certificat envoyee a une CA |

## D

| Terme | Definition |
|-------|-----------|
| **DC** (Domain Controller) | Controleur de domaine : serveur hebergeant une copie de la base AD DS |
| **DHCP** (Dynamic Host Configuration Protocol) | Protocole d'attribution automatique des adresses IP et configurations reseau |
| **DFS** (Distributed File System) | Systeme de fichiers distribue permettant la replication et les espaces de noms unifies |
| **DISM** (Deployment Image Servicing and Management) | Outil de maintenance des images Windows |
| **DNS** (Domain Name System) | Systeme de resolution de noms de domaine en adresses IP |
| **DSRM** (Directory Services Restore Mode) | Mode de demarrage special pour la restauration d'Active Directory |

## E-F

| Terme | Definition |
|-------|-----------|
| **ETW** (Event Tracing for Windows) | Infrastructure de tracage d'evenements haute performance du noyau Windows |
| **FSMO** (Flexible Single Master Operations) | Cinq roles AD operant en maitre unique : Schema Master, Domain Naming Master, PDC Emulator, RID Master, Infrastructure Master |

## G

| Terme | Definition |
|-------|-----------|
| **GC** (Global Catalog) | Catalogue global AD contenant un sous-ensemble d'attributs de tous les objets de la foret |
| **GPO** (Group Policy Object) | Objet de strategie de groupe definissant des parametres de configuration pour les utilisateurs et ordinateurs |
| **GUI** (Graphical User Interface) | Interface graphique (par opposition a Server Core) |

## H-I

| Terme | Definition |
|-------|-----------|
| **HA** (High Availability) | Haute disponibilite : conception minimisant les temps d'arret |
| **HSTS** (HTTP Strict Transport Security) | Politique de securite forcant l'utilisation de HTTPS |
| **Hyper-V** | Hyperviseur natif de Microsoft pour la virtualisation |
| **IIS** (Internet Information Services) | Serveur web integre a Windows Server |
| **IPAM** (IP Address Management) | Outil de gestion centralisee des adresses IP |
| **iSCSI** | Protocole de stockage en reseau transmettant des commandes SCSI sur TCP/IP |

## K-L

| Terme | Definition |
|-------|-----------|
| **KQL** (Kusto Query Language) | Langage de requete utilise dans Azure Monitor et Log Analytics |
| **LAPS** (Local Administrator Password Solution) | Solution de gestion des mots de passe administrateur locaux via AD |
| **LDAP** (Lightweight Directory Access Protocol) | Protocole d'acces a l'annuaire Active Directory (port TCP 389) |
| **LDAPS** | LDAP securise par TLS (port TCP 636) |
| **LRS** (Locally Redundant Storage) | Stockage Azure replique localement (3 copies dans un datacenter) |

## M-N

| Terme | Definition |
|-------|-----------|
| **MARS** (Microsoft Azure Recovery Services) | Agent de sauvegarde Azure pour les serveurs on-premises |
| **MMC** (Microsoft Management Console) | Console d'administration modulaire hebergeant des snap-ins |
| **NLB** (Network Load Balancing) | Equilibrage de charge reseau Windows pour les services sans etat |
| **NTFS** (New Technology File System) | Systeme de fichiers principal de Windows avec support des permissions, chiffrement et compression |
| **NTLM** (NT LAN Manager) | Protocole d'authentification ancien, remplace par Kerberos dans les domaines AD |

## O-P

| Terme | Definition |
|-------|-----------|
| **OU** (Organizational Unit) | Unite d'organisation dans AD : conteneur logique pour organiser les objets |
| **PHS** (Password Hash Synchronization) | Mode de synchronisation des hash de mots de passe vers Azure AD |
| **PKI** (Public Key Infrastructure) | Infrastructure de gestion de cles publiques et certificats |
| **PRA** (Plan de Reprise d'Activite) | Plan definissant les procedures de reprise apres un sinistre |
| **PTA** (Pass-Through Authentication) | Mode d'authentification Azure AD delegant la validation a un agent on-premises |
| **PTR** (Pointer Record) | Enregistrement DNS pour la resolution inverse (IP vers nom) |

## Q-R

| Terme | Definition |
|-------|-----------|
| **Quorum** | Mecanisme de vote dans un cluster determinant sa capacite a fonctionner |
| **RBAC** (Role-Based Access Control) | Controle d'acces base sur les roles |
| **RDP** (Remote Desktop Protocol) | Protocole de bureau a distance (port TCP 3389) |
| **RPO** (Recovery Point Objective) | Perte de donnees maximale acceptable en cas de sinistre |
| **RSAT** (Remote Server Administration Tools) | Outils d'administration a distance pour Windows Server |
| **RTO** (Recovery Time Objective) | Duree maximale d'indisponibilite acceptable |

## S

| Terme | Definition |
|-------|-----------|
| **S2D** (Storage Spaces Direct) | Stockage defini par logiciel haute disponibilite de Microsoft |
| **SFC** (System File Checker) | Outil de verification et reparation des fichiers systeme Windows |
| **SID** (Security Identifier) | Identifiant unique attribue a chaque objet de securite (utilisateur, groupe, ordinateur) |
| **SMB** (Server Message Block) | Protocole de partage de fichiers Windows (ports TCP 445) |
| **SNI** (Server Name Indication) | Extension TLS permettant d'heberger plusieurs certificats sur une meme IP |
| **SPN** (Service Principal Name) | Identifiant unique d'un service dans Kerberos |
| **SRV** (Service Record) | Enregistrement DNS identifiant un service (ex. : _ldap._tcp) |
| **SSL/TLS** | Protocoles de chiffrement des communications reseau |
| **SYSVOL** | Partage replique entre les DC contenant les scripts de connexion et les GPO |

## T-V

| Terme | Definition |
|-------|-----------|
| **UPN** (User Principal Name) | Identifiant utilisateur au format email (prenom.nom@domaine.local) |
| **VHDX** | Format de disque dur virtuel Hyper-V (successeur du VHD) |
| **VLAN** (Virtual LAN) | Reseau local virtuel isolant le trafic au niveau 2 (Ethernet) |
| **VSS** (Volume Shadow Copy Service) | Service de copies instantanees de volumes pour les sauvegardes coherentes |

## W

| Terme | Definition |
|-------|-----------|
| **WAC** (Windows Admin Center) | Outil de gestion web moderne pour Windows Server |
| **WDS** (Windows Deployment Services) | Service de deploiement d'images Windows par le reseau (PXE) |
| **WEC** (Windows Event Collector) | Service collecteur d'evenements Windows |
| **WEF** (Windows Event Forwarding) | Mecanisme de transfert des evenements Windows entre serveurs |
| **WinRM** (Windows Remote Management) | Implementation Microsoft de WS-Management pour l'administration a distance (ports TCP 5985/5986) |
| **WSUS** (Windows Server Update Services) | Service de gestion centralisee des mises a jour Windows |

