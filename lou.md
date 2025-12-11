# Jeune prise de note 

### Pourquoi utiliser des VM Ubuntu pour ce projet 💻🖥️

L’utilisation de machines virtuelles (VM) Ubuntu présente de nombreux avantages pour un projet de sauvegarde et restauration, tant pour l’apprentissage que pour des environnements de test ou de production. Voici les principaux points :

1. Isolation et sécurité 🔒

- Chaque VM fonctionne comme un système indépendant. Une erreur ou une mauvaise manipulation dans une VM n’affecte pas l’hôte ni les autres machines virtuelles.

- Cela permet de tester des scripts de sauvegarde et de restauration sans risquer de compromettre le système principal.

2. Reproductibilité et portabilité 🔄✈️

- Les VM peuvent être clonées, sauvegardées et restaurées facilement, ce qui facilite la réplication d’environnements pour les exercices pratiques.

- Les configurations sont portables : une VM Ubuntu configurée sur un PC peut être déplacée vers un autre ordinateur ou un serveur sans modification majeure.

3. Contrôle complet de l’environnement ⚙️

- Ubuntu offre un environnement Linux standard, avec accès complet au terminal, aux outils systèmes et aux permissions root.

- Cela permet de travailler avec tous les outils de sauvegarde étudiés : rsync, tar, BorgBackup, Restic, Duplicity, mysqldump, etc.

- Nous pouvons ainsi expérimenter tous les niveaux de sauvegarde (fichiers, bloc, image, bases de données) sans limitation.

4. Flexibilité pour la simulation de scénarios 🛠️🎭

- Les VM permettent de simuler des pannes, des corruptions de fichiers, des ransomwares ou des pertes de données de manière contrôlée.

- Cela rend possible la mise en place et le test de plans de restauration (DRP) sans conséquences réelles sur des systèmes de production.

5. Économie de ressources et gestion simplifiée 💾📦

- Plusieurs VM Ubuntu peuvent tourner sur un seul PC grâce à la virtualisation, ce qui permet de réaliser un laboratoire complet de sauvegarde multi-sites ou multi-niveaux sans multiplier les machines physiques.

- Les instantanés (snapshots) des VM permettent de revenir en arrière rapidement après un test, réduisant le temps nécessaire pour réinitialiser un environnement.

6. Compatibilité et documentation abondante 📚🌐

- Ubuntu est largement utilisé en entreprise et en formation, ce qui facilite la recherche de documentation et le support communautaire.

- Les outils de sauvegarde open source et propriétaires sont généralement testés et optimisés pour Linux, ce qui assure une compatibilité maximale.

7. Préparation au monde professionnel 🚀🏢

- La pratique sur VM Ubuntu reflète de réels environnements de production Linux, où les administrateurs mettent en place des stratégies de sauvegarde et de restauration.

- Les compétences acquises sont directement transférables à des infrastructures physiques ou cloud, ce qui est un vrai atout pour la formation.

💡 Conclusion :
L’utilisation de VM Ubuntu combine sécurité 🔒, flexibilité 🛠️, reproductibilité 🔄 et accessibilité 🌐. C’est un choix idéal pour un projet pédagogique sur la sauvegarde et la restauration, permettant aux étudiants de manipuler des systèmes complets et de tester différentes stratégies sans risque pour le matériel ou les données réelles.

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



### Faire des backup entre deux vm ###

1️⃣ Préparer la VM de backup
Mettre à jour le système :
```
sudo apt update && sudo apt upgrade -y
```
2️⃣ Installer BorgBackup
```
sudo apt install borgbackup -y
```
Vérifie la version :
```
borg --version
```
3️⃣ Créer un utilisateur pour les backups

C’est une bonne pratique pour la sécurité :
```
sudo adduser --system --group borgbackup
sudo mkdir -p /srv/borg-repo
sudo chown borgbackup:borgbackup /srv/borg-repo
```

/srv/borg-repo = emplacement où les backups seront stockés

4️⃣ Initialiser le dépôt Borg

Connecte-toi en tant qu’utilisateur borgbackup ou utilise sudo -u borgbackup :
```
sudo -u borgbackup borg init --encryption=repokey /srv/borg-repo
```

--encryption=repokey → chiffré avec la clé stockée dans le repo

Borg va créer le dépôt initial

5️⃣ Préparer l’accès SSH depuis le serveur principal

Pour que le serveur principal puisse envoyer des backups automatiquement :

Sur le serveur principal :
```
ssh-keygen -t ed25519 -C "backup-key" -f ~/.ssh/id_borgbackup
```

Copier la clé publique vers la VM backup :
```
ssh-copy-id -i ~/.ssh/id_borgbackup.pub borgbackup@192.168.x.y
```

192.168.x.y = IP de ta VM backup sur le réseau interne

L’utilisateur distant = borgbackup

6️⃣ Tester l’accès SSH

Depuis le serveur principal :
```
ssh -i ~/.ssh/id_borgbackup borgbackup@192.168.x.y
```

Tu dois arriver sur la VM de backup sans mot de passe

Vérifie que l’utilisateur borgbackup peut accéder au dépôt /srv/borg-repo

7️⃣ Créer un premier backup manuel (test)

Depuis le serveur principal :
```
export BORG_RSH="ssh -i ~/.ssh/id_borgbackup"
borg create borgbackup@192.168.x.y:/srv/borg-repo::test-backup ~/  # exemple : backup du home
```

- ::test-backup → nom du snapshot

- ~/ → dossier à sauvegarder (tu pourras mettre /etc, /var/lib/docker/volumes, etc.)

8️⃣ Automatiser avec un script

Exemple /usr/local/bin/backup.sh sur le serveur principal :
```
#!/bin/bash
export BORG_RSH="ssh -i ~/.ssh/id_borgbackup"
BACKUP_DIRS="/etc /var/lib/docker/volumes /home"
borg create borgbackup@192.168.10.12:/srv/borg-repo::$(date +%Y-%m-%d) $BACKUP_DIRS
borg prune -v --keep-daily=7 --keep-weekly=4 --keep-monthly=3 borgbackup@192.168.10.12:/srv/borg-repo
```

Prune = supprime les snapshots anciens selon ta politique

Mets le script exécutable :
```
chmod +x /usr/local/bin/backup.sh
```

Teste-le :
```
/usr/local/bin/backup.sh
```
9️⃣ Planification automatique (cron)

Exemple pour un backup quotidien à 2h00 du matin :
```
sudo crontab -e
```

Ajouter la ligne :
```
0 2 * * * /usr/local/bin/backup.sh >> /var/log/borgbackup.log 2>&1
```

Les logs sont conservés dans /var/log/borgbackup.log

🔟 Restauration d’un backup (test)

Depuis la VM backup ou le serveur principal :
```
borg list borgbackup@192.168.10.12:/srv/borg-repo   # voir les snapshots
borg extract borgbackup@192.168.10.12:/srv/borg-repo::2025-12-05 /restore/path
```

/restore/path = chemin temporaire pour vérifier que les fichiers sont corrects

✅ Résultat attendu :

- Le serveur principal peut sauvegarder automatiquement vers la VM backup
- Tu peux restaurer rapidement
- Tout est isolé sur le réseau interne (host-only)
- Prépare parfaitement ton projet pour la partie sauvegarde et restauration de l’énoncé
