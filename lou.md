# Jeune prise de note 

### 🌱 1. Pourquoi Debian est un excellent choix pour ton projet

Debian est l’un des systèmes Linux les plus utilisés dans le monde pour des infrastructures serveur.
Dans un contexte d’étudiant et de projet réseau, Debian apporte plusieurs avantages très concrets :

✔️ Stable

Debian privilégie la stabilité plutôt que les nouveautés toutes fraîches.
Résultat : très peu de bugs, comportement prévisible = idéal pour un projet académique qui doit fonctionner de manière fiable.

✔️ Documenté partout

Comme c’est l’un des OS les plus utilisés dans les cours d’administration système, tu trouveras des tutoriels, guides, forums, docs officielles partout.
C’est très utile quand tu bloques à une étape.

✔️ Paquets très propres

Les services que tu vas déployer (DNS, nginx, Apache, PostgreSQL, Docker, etc.) sont très bien packagés dans Debian.
L'installation est simple, standardisée, et sans surprise.

✔️ Idéal pour l’automatisation

Ansible, scripts Bash, cloud-init… Tous ces outils fonctionnent parfaitement sur Debian.
Et il y a moins de variations entre versions que sur Ubuntu.

✔️ Très utilisé en entreprise

C’est exactement ce qui est visé dans ton projet : “vous êtes administrateur système dans une entreprise”.
Debian est un choix professionnel, crédible et aligné avec ce qui se fait dans la vraie vie.

✔️ Léger

Pour VirtualBox c’est parfait : peu gourmand en ressources → tu peux lancer plusieurs VMs sans exploser la RAM.

👉 Bref : Debian c’est stable, simple, documenté et pro.
Pour un projet formation → parfait.


### Plan de structures 
```
VM1 – SRV-CORE (Serveur principal)

Debian 13

Rôles :

DNS primaire (Bind9)

Serveur web HTTPS (Nginx)

Reverse proxy pour les services en conteneurs

Sauvegardes (borgbackup ou rsync)

Ports :

22 (SSH)

53 (DNS)

80/443 (web)

🖥️ VM2 – SRV-DOCKER

Debian 13

Rôles :

Docker + docker-compose

Services conteneurisés :

un site web simple

une base de données (MariaDB/Postgres)

un service supplémentaire (ex : Nextcloud, une app Flask, etc.)

Déploiement “une commande” = docker compose up -d

🖥️ VM3 – SRV-BACKUP

Debian 13

Rôles :

Serveur de stockage des sauvegardes

BorgBackup Server / Rsync server

Restauration automatisée

🖥️ VM4 – SRV-MONITORING

Debian 13

Rôles :

Prometheus + Grafana

Exporters sur les autres serveurs (node_exporter)

Alertes (mail optionnel)
 ```

 ### Explication :

 🌱 Pourquoi ton architecture est excellente
🖥️ VM1 — SRV-CORE

➡️ Serveur principal – cœur réseau et services critiques

Rôles :

DNS primaire (Bind9) → couvre clairement le point "Service DNS" du projet.

Serveur web HTTPS (Nginx) → couvre "Service web en HTTPS".

Reverse proxy → permet d'exposer proprement les services conteneurisés de la VM Docker.

Sauvegardes vers SRV-BACKUP → permet :

workflow clean : les VMs se sauvegardent vers SRV-BACKUP

gestion des clés

stratégie centralisée

Pourquoi c’est intelligent :

Tu sépares le front-end et l'infra "sensibles" du serveur Docker.

Tu gardes toutes les fonctions les plus critiques dans un seul serveur robuste et facile à sécuriser (Nginx + DNS + reverse proxy).

Tu contrôles l’accès réseau en centralisant les flux.

🖥️ VM2 — SRV-DOCKER

➡️ Serveur applicatif / conteneurisation complète

Rôles :

Docker + docker-compose

Services conteneurisés :

Ton site web (ou API)

Une base de données

Un service supplémentaire (Nextcloud, app Flask, Mattermost…)

Pourquoi c’est malin :

Tu réponds parfaitement à l’exigence “Conteneurisation”.

Tu gardes les applications dans une machine isolée du système “core” (c’est professionnel).

docker compose up -d → démo parfaite en soutenance côté automisation.

Tu peux versionner ton docker-compose.yml dans Git : point bonus.

Ce que tu monteras en soutenance :

Lancement automatique du stack

Communication via le reverse proxy du SRV-CORE

Base accessible uniquement depuis l’intérieur du réseau → sécurité top

🖥️ VM3 — SRV-BACKUP

➡️ Serveur de sauvegardes dédié

Rôles :

BorgBackup ou rsync-server

Restauration automatisée

Ce point est très important pour l’évaluation mi-parcours !

Pourquoi c’est un choix malin :

Tu réponds complètement à “sauvegarde et restauration automatisées”.

BorgBackup est un excellent choix : chiffré, déduplication, facile à démontrer.

Avoir un serveur dédié :

c’est très réaliste (séparation des responsabilités)

ça renforce la sécurité (on ne sauvegarde pas sur la machine elle-même)

ça simplifie la démonstration

En soutenance, tu montreras par exemple :

1️⃣ Je supprime un fichier de config sur SRV-CORE
2️⃣ Je lance borg extract
3️⃣ Le service repart

👉 T’as gagné des points direct.

🖥️ VM4 — SRV-MONITORING

➡️ Super idée, très propre, très pro.

Rôles :

Prometheus

Grafana

node_exporter sur tous les serveurs

Alertes (option mail ou Discord)

Pourquoi c’est une excellente idée :

Ça répond à “Surveillance” du projet.

C’est très professionnel : toutes les entreprises utilisent Prometheus/Grafana.

Tu vas impressionner en soutenance avec un dashboard animé (CPU, RAM, services up/down).

Les alertes (ex : “Service Docker Offline”) montrent ta maîtrise.

Petit bonus facile :

Ajouter Blackbox exporter → monitoring HTTPS du reverse proxy

Ajouter Promtail + Loki → logs centralisés

Mais ce n’est pas obligatoire.

### Schéma réseau 

```
🌐 Schéma réseau (VirtualBox)

Réseau interne “LAN-ENTREPRISE” : 192.168.10.0/24

Machine    IP
SRV-CORE    192.168.10.10
SRV-DOCKER    192.168.10.11
SRV-BACKUP    192.168.10.12
SRV-MONITORING    192.168.10.13
CLIENT    192.168.10.20
```



🔧 1️⃣ Principe général (VirtualBox)

Chaque VM aura 2 cartes réseau :

🟦 Adaptateur 1 (NAT)

✔ Permet d’avoir Internet dans la VM
✔ Permet de faire apt update, installer des paquets, etc.
✔ Ne participe PAS au réseau interne entre les VMs

→ Type : NAT
→ Aucune config d’IP à faire, Debian reçoit une IP 10.x automatiquement.

🟪 Adaptateur 2 (réseau interne LAN-ENTREPRISE)

✔ C’est TON vrai réseau d’entreprise
✔ Toutes les machines communiquent ensemble
✔ C’est là que tu mets les IP fixes 192.168.10.x
✔ Aucun accès Internet → réseau isolé (réel, propre)

→ Type : Réseau interne
→ Nom : LAN-ENTREPRISE

Ainsi :

Adaptateur 1 = Internet

Adaptateur 2 = Réseau interne du projet