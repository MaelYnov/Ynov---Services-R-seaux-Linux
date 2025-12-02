#  Debian
### Pourquoi ?

- Stable, très utilisé en production.

- Parfait pour les services web, DNS, sauvegarde, conteneurs (Docker/Podman).

- Large documentation + communauté gigantesque.

- Peu gourmand en ressources (pratique en environnement virtualisé).

- Idéal pour l’automatisation via Ansible, scripts Bash, Terraform, etc.

### Quand l’utiliser ?

- VM serveur web (Nginx/Apache + HTTPS)

- Serveur DNS (Bind9, Unbound…)

- Serveur de sauvegarde

- Host Docker/Podman

- Supervison (Prometheus, Grafana, Zabbix…)

##

# 🧱 ARCHITECTURE (4 VM + Docker)
### 🖥️ VM1 – SRV-CORE (Serveur principal)

- Debian 13

### Rôles :

- DNS primaire (Bind9)

- Serveur web HTTPS (Nginx)

- Reverse proxy pour les services en conteneurs

- Sauvegardes (borgbackup ou rsync)

- Ports :

- 22 (SSH)

- 53 (DNS)

- 80/443 (web)

### 🖥️ VM2 – SRV-DOCKER

- Debian 13

### Rôles :

- Docker + docker-compose

- Services conteneurisés :

- un site web simple

- une base de données (MariaDB/Postgres)

- un service supplémentaire (ex : Nextcloud, une app Flask, etc.)

- Déploiement “une commande” = docker compose up -d

### 🖥️ VM3 – SRV-BACKUP

- Debian 13

### Rôles :

- Serveur de stockage des sauvegardes

- BorgBackup Server / Rsync server

- Restauration automatisée

### 🖥️ VM4 – SRV-MONITORING

- Debian 13

### Rôles :

- Prometheus + Grafana

- Exporters sur les autres serveurs (node_exporter)

- Alertes (mail optionnel)