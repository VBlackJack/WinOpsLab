# WinOpsLab

Base de connaissances **Windows Server 2022** — Parcours d'apprentissage du debutant a l'expert.

Site statique genere avec [MkDocs](https://www.mkdocs.org/) et le theme [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Contenu

| Section | Pages | Description |
|---------|:-----:|-------------|
| **Fondamentaux** | 13 | Installation, console de gestion, PowerShell, roles et fonctionnalites |
| **Active Directory** | 23 | AD DS, DNS, DHCP, Strategies de groupe (GPO) |
| **Reseau** | 15 | TCP/IP, pare-feu Windows, acces distant, services avances |
| **Stockage** | 12 | Disques, Storage Spaces, DFS, partage de fichiers |
| **Securite** | 13 | Durcissement, PKI, audit, protection des donnees |
| **Virtualisation** | 9 | Hyper-V, conteneurs Windows |
| **Haute disponibilite** | 11 | Cluster de basculement, sauvegarde, equilibrage de charge |
| **Automatisation** | 13 | PowerShell avance, DSC, taches planifiees, Infrastructure as Code |
| **Supervision** | 10 | Surveillance systeme, depannage, centralisation des logs |
| **Gestion moderne** | 10 | Windows Admin Center, Azure Hybrid, IIS |
| **Labs** | 16 | Environnement de lab, 10 exercices pratiques, 3 projets de synthese |
| **Ressources** | 4 | Glossaire, commandes essentielles, liens utiles, certifications |
| **Total** | **149** | |

## Prerequis

- Python 3.10+
- Git

## Installation

```bash
# Cloner le depot
git clone https://github.com/VBlackJack/WinOpsLab.git
cd WinOpsLab

# Creer l'environnement virtuel
python -m venv .venv

# Activer l'environnement virtuel
# Windows PowerShell :
.\.venv\Scripts\Activate.ps1
# Git Bash :
source .venv/Scripts/activate

# Installer les dependances
pip install -r requirements.txt
```

## Utilisation

```bash
# Lancer le serveur de developpement
mkdocs serve

# Acceder au site
# http://127.0.0.1:8000

# Construire le site statique
mkdocs build --strict
```

## Stack technique

- **Generateur** : MkDocs 1.6+
- **Theme** : Material for MkDocs 9.7+
- **Langue** : Francais
- **Extensions** : Admonitions, onglets, blocs de code, diagrammes Mermaid, taches, raccourcis clavier
- **Plugins** : Recherche (fr), date de revision Git, minification, GLightbox

## Structure du projet

```
WinOpsLab/
├── docs/                   # Contenu Markdown
│   ├── index.md            # Page d'accueil
│   ├── fondamentaux/       # Module 1 - Fondamentaux
│   ├── active-directory/   # Module 2 - Active Directory
│   ├── reseau/             # Module 3 - Reseau
│   ├── stockage/           # Module 4 - Stockage
│   ├── securite/           # Module 5 - Securite
│   ├── virtualisation/     # Module 6 - Virtualisation
│   ├── haute-disponibilite/# Module 7 - Haute disponibilite
│   ├── automatisation/     # Module 8 - Automatisation
│   ├── supervision/        # Module 9 - Supervision
│   ├── gestion-moderne/    # Module 10 - Gestion moderne
│   ├── labs/               # Module 11 - Labs pratiques
│   ├── ressources/         # Module 12 - Ressources
│   └── stylesheets/        # CSS personnalise
├── mkdocs.yml              # Configuration MkDocs
├── requirements.txt        # Dependances Python
└── .gitignore
```

## Licence

Copyright 2026 Julien Bombled

Distribue sous licence [Apache 2.0](http://www.apache.org/licenses/LICENSE-2.0).
