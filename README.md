# ansible-infra-automation
# Projet Ansible – Architecture Complète pour l'Enseignement

## Objectif

Construire un projet Ansible **complet, modulaire et pédagogique** qui permet de :
1. Provisionner un environnement Linux sécurisé
2. Installer et configurer Docker (avec sécurité renforcée)
3. Déployer une stack 3-tier WordPress (MySQL + WordPress + Nginx reverse proxy)
4. Automatiser les sauvegardes de la stack
5. Déployer un outil de gestion visuelle (Polemarch — interface Web pour l'exécution d'Ansible)

Le projet est conçu comme un **support pédagogique de production** pour illustrer tous les concepts clés d'Ansible.

---

## Concepts Ansible Couverts

| Concept | Où il est appliqué |
|---|---|
| Inventaires statiques & dynamiques | `inventory/` |
| Variables (`group_vars`, `host_vars`) | Configuration par environnement |
| Rôles (`roles/`) | Modularité et réutilisabilité |
| Handlers | Redémarrage de services sur changement |
| Templates Jinja2 | Configs dynamiques (nginx, docker-compose) |
| Vault (`ansible-vault`) | Secrets (mots de passe BDD, clés) |
| Tags | Exécution sélective des tâches |
| Modules clés | `apt`, `copy`, `template`, `service`, `docker_compose`, `ufw`, `cron`, `user` |
| Conditions (`when`) | Différenciation Debian/RedHat |
| Boucles (`loop`) | Installation de paquets en masse |
| `notify` & `flush_handlers` | Synchronisation des redémarrages |
| `block` / `rescue` / `always` | Gestion d'erreurs |
| Playbook orchestrateur ([site.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/site.yml)) | Point d'entrée unique |

---

## Architecture du Projet

```
__ANSIBLE/
├── ansible.cfg                    # Configuration globale Ansible
├── site.yml                       # Playbook orchestrateur principal
├── setup_linux.yml                # Provisionnement Linux de base
├── deploy_docker.yml              # Installation et sécurisation Docker
├── deploy_wordpress.yml           # Stack 3-tier WordPress
├── run_backup.yml                 # Sauvegardes automatisées
├── deploy_polemarch.yml           # Déploiement Polemarch UI
│
├── inventory/
│   ├── hosts.ini                  # Inventaire statique
│   └── group_vars/
│       ├── all.yml                # Variables globales
│       ├── webservers.yml         # Variables pour les serveurs web
│       └── dbservers.yml         # Variables pour les serveurs BDD
│
├── host_vars/
│   └── server1.yml                # Variables spécifiques à un hôte
│
├── roles/
│   ├── common/                    # Configuration de base Linux
│   ├── security/                  # Hardening sécurité
│   ├── docker/                    # Docker + Docker Compose
│   ├── wordpress/                 # Stack 3-tier
│   ├── backup/                    # Sauvegarde automatisée
│   └── polemarch/                 # Polemarch UI
│
├── files/                         # Fichiers statiques (scripts, certs)
├── templates/                     # Templates Jinja2 partagés
│
└── README.md                      # Documentation principale du projet
```

---

## Détail des Rôles

### Rôle `common`
- Configuration hostname, timezone, locale
- Création d'un utilisateur système dédié (`ansible_user`)
- Configuration SSH hardening (`/etc/ssh/sshd_config`)
- Installation des paquets de base (`curl`, `vim`, `htop`, `git`, `python3-pip`)
- Configuration du banner SSH

### Rôle `security`
- Installation et configuration UFW (firewall)
  - Ports autorisés : 22 (SSH), 80 (HTTP), 443 (HTTPS), 8080 (Polemarch)
- Installation et configuration Fail2Ban
- Mises à jour automatiques (`unattended-upgrades`)
- Désactivation de l'authentification root SSH
- Audit des permissions SUID/SGID

### Rôle `docker`
- Installation Docker Engine (via le dépôt officiel)
- Installation Docker Compose v2
- Configuration daemon Docker sécurisé (`/etc/docker/daemon.json`)
  - Limitation de logging
  - Activation de `userland-proxy: false`
  - `no-new-privileges: true`
- Ajout de l'utilisateur au groupe `docker`
- Activation et démarrage du service Docker

### Rôle `wordpress`
**Stack 3-tier via Docker Compose :**
- **Couche 1 – Base de données** : MySQL 8.0 (conteneur isolé, réseau privé)
- **Couche 2 – Application** : WordPress (php-fpm)
- **Couche 3 – Reverse Proxy** : Nginx (avec configuration SSL-ready)
- Template Jinja2 pour `docker-compose.yml` (variables injectées)
- Template Nginx pour le vhost WordPress
- Gestion des secrets via `ansible-vault`

### Rôle `backup`
- Script de sauvegarde des volumes Docker
- Export de la base de données MySQL (`mysqldump`)
- Compression et archivage (`tar.gz` avec horodatage)
- Rotation des sauvegardes (conservation des 7 dernières)
- Planification via cron Ansible

### Rôle `polemarch`
- Déploiement de **Polemarch** (interface web pour la gestion des exécutions Ansible)
  - Interface web pour gérer et lancer les playbooks Ansible (compatible CLI)
  - Support des projets, inventaires, tâches planifiées
- Installation via Docker Compose (Polemarch + MySQL + Redis)
- Configuration de l'accès sécurisé sur le port 8080

---

## Playbooks Principaux

| Playbook | Description |
|---|---|
| [site.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/site.yml) | Orchestrateur – implémente toute la stack dans l'ordre |
| [setup_linux.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/setup_linux.yml) | `common` + `security` |
| [deploy_docker.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_docker.yml) | Rôle `docker` uniquement |
| [deploy_wordpress.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_wordpress.yml) | Rôle `wordpress` uniquement |
| [run_backup.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/run_backup.yml) | Rôle `backup` (utilisable manuellement ou via Polemarch) |
| [deploy_polemarch.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_polemarch.yml) | Rôle `polemarch` uniquement |

---

## Choix Technologique : Polemarch vs AWX

> **Recommandation : Polemarch**

| Critère | Polemarch | AWX |
|---|---|---|
| Ressources requises | Faibles (Docker Compose) | Élevées (4+ Go RAM, Kubernetes) |
| Stabilité | ✅ Très stable | ⚠️ Complexe à maintenir |
| Installation | Simple (Docker Compose) | Complexe (Kubernetes) |
| Adapté aux débutants | ✅ Oui | ❌ Non |
| Interface moderne | ✅ Oui | ✅ Oui |

---

## Plan de Vérification

### Vérification statique (syntaxe)
```bash
# Depuis le répertoire du projet
ansible-lint site.yml
yamllint .
ansible-playbook site.yml --syntax-check
```

### Dry-run (simulation sans exécution)
```bash
ansible-playbook site.yml -i inventory/hosts.ini --check --diff
```

### Vérification de structure
```bash
# Vérifier que tous les rôles ont la structure standard
find roles/ -name "main.yml" | sort
```

### Test d'inventaire
```bash
ansible all -i inventory/hosts.ini --list-hosts
ansible all -i inventory/hosts.ini -m ping
```

### Validation manuelle (post-déploiement sur serveur cible)
1. Accéder à `http://<IP_SERVEUR>` → Page d'installation WordPress visible
2. Accéder à `http://<IP_SERVEUR>:8080` → Interface Polemarch accessible
3. Vérifier `ufw status` sur le serveur → règles actives
4. Vérifier `docker ps` → 3 conteneurs WordPress en cours d'exécution
5. Vérifier le dossier de backup → fichiers `.tar.gz` présents

---

## Prérequis (côté étudiant)

- **Contrôleur Ansible** : Machine Linux/WSL/macOS avec Ansible ≥ 2.14
- **Hôtes cibles** : 1 à 3 VMs Ubuntu 22.04 LTS (VMware, VirtualBox, Cloud)
- **Accès SSH** : Clé SSH configurée vers les hôtes cibles
- **Python 3** : Sur les hôtes cibles

---

## Fichiers à Créer

### Fichiers racine
- [NEW] [ansible.cfg](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/ansible.cfg)
- [NEW] [site.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/site.yml)
- [NEW] [setup_linux.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/setup_linux.yml)
- [NEW] [deploy_docker.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_docker.yml)
- [NEW] [deploy_wordpress.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_wordpress.yml)
- [NEW] [run_backup.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/run_backup.yml)
- [NEW] [deploy_polemarch.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/deploy_polemarch.yml)
- [NEW] `README.md`

### Inventaire
- [NEW] [inventory/hosts.ini](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/inventory/hosts.ini)
- [NEW] [inventory/group_vars/all.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/inventory/group_vars/all.yml)
- [NEW] [inventory/group_vars/webservers.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/inventory/group_vars/webservers.yml)
- [NEW] [inventory/group_vars/dbservers.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/inventory/group_vars/dbservers.yml)
- [NEW] [host_vars/server1.yml](file:///c:/Users/ulric/OneDrive%20-%20MSFT/__SUN%20DATA%20CONSULTING/__ANSIBLE/host_vars/server1.yml)

### Rôles (structure complète avec tasks, handlers, templates, vars, defaults, meta)
- [NEW] `roles/common/` — 6 fichiers
- [NEW] `roles/security/` — 6 fichiers
- [NEW] `roles/docker/` — 6 fichiers
- [NEW] `roles/wordpress/` — 8 fichiers (inclut templates Nginx + docker-compose)
- [NEW] `roles/backup/` — 5 fichiers
- [NEW] `roles/polemarch/` — 5 fichiers
