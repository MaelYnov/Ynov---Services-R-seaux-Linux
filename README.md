
# 📚 Rapport de Projet Infrastructure et Sauvegarde

Ce rapport détaille les choix techniques effectués et la mise en œuvre des exercices de virtualisation, de sauvegarde, et de sécurité réseau, basés sur une infrastructure Linux.

-----

## 💡 Choix de l'OS : Debian

Le choix de **Debian** comme système d'exploitation de base pour les machines virtuelles (VMs) repose sur plusieurs facteurs cruciaux pour un environnement de laboratoire ou de production stable :

  * **Stabilité et Fiabilité :** Debian est réputé pour sa rigueur dans la sélection des paquets. La branche "Stable" assure une robustesse essentielle pour les environnements de serveur.
  * **Sécurité :** L'équipe de sécurité de Debian est très active, fournissant des mises à jour rapides pour les vulnérabilités.
  * **Minimalisme :** Debian permet une installation minimale, ce qui réduit la surface d'attaque et optimise l'utilisation des ressources de la VM.

-----
## 🛠️ Mise en place des VM

Tout d'abord, j'ai créer 3 VM, un serveur contenant toutes les données, une vm backup et une vm client. J'ai tout d'abord configuré mes interfaces (enp0s3 réseau interne, enp0s8 Nat) : 

sudo nano /etc/network/interfaces

auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
    address 192.168.50.102  # 102 pour ser, 103 client, 104 backup.
    netmask 255.255.255.0

auto enp0s8
iface enp0s8 inet dhcp

sudo systemctl restart networking.service
sudo systemctl status networking.service

On est censé voir actif si bien config et avec ip a on peut voir notre adresse ip configuré.


-----
## 🛠️ Exercice 1 : Mise en Place de la Sauvegarde Rsync

L'objectif de cet exercice était de mettre en place une sauvegarde simple et incrémentale des données locales vers une VM distante via SSH, en utilisant **Rsync**.

Rsync est utilisé ici pour sa capacité à transférer uniquement les différences entre les fichiers, ce qui est rapide et efficace pour une première copie de travail.

### Script de Sauvegarde (`/usr/local/bin/backup_script.sh`)

Ce script automatise la copie de répertoires spécifiques (`/home` et `/var/www`) vers un dépôt de sauvegarde distant. Pour cela, on établit une connexion ssh depuis le serveur vers ma vm backup. 

```bash
USER="hugo"
HOST="192.168.56.102"
# Chemin du répertoire racine des sauvegardes sur la destination
REMOTE_BACKUP_PATH="/mnt/backups/rsync" 
# Email de notification
EMAIL_TO="hugo.cabanes@ynov.com" 
# Fichier de log local
LOG_FILE="$HOME/rsync_backup_$(date +%Y%m%d).log"
# Nombre de jours de rétention (7 jours)
RETENTION=7 
# -----------------

# Redirection de toutes les sorties (stdout et stderr) vers le fichier de log
exec > "$LOG_FILE" 2>&1

echo "--- Rsync Backup started on $(date) ---"
echo "----------------------------------------"

# 1. ROTATION DES SAUVEGARDES (sur la VM de destination)
# On se connecte en SSH pour gérer les répertoires distants
echo "1. Rotation des sauvegardes..."
SSH_CMD="ssh ${USER}@${HOST}"

# Supprime la plus ancienne (au-delà de la rétention)
$SSH_CMD "rm -rf ${REMOTE_BACKUP_PATH}/daily.${RETENTION}"

# Décale les sauvegardes existantes (ex: daily.6 -> daily.7, daily.5 -> daily.6>
for i in $(seq $((RETENTION - 1)) -1 0); do
    if $SSH_CMD "[ -d ${REMOTE_BACKUP_PATH}/daily.$i ]"; then
        echo "Déplacement de daily.$i vers daily.$((i+1))"
        $SSH_CMD "mv ${REMOTE_BACKUP_PATH}/daily.$i ${REMOTE_BACKUP_PATH}/daily>
    fi
done

# Crée le nouveau répertoire pour la sauvegarde du jour (daily.0)
$SSH_CMD "mkdir -p ${REMOTE_BACKUP_PATH}/daily.0"
 Vérification de l'existence de la sauvegarde précédente pour --link-dest
PREVIOUS_BACKUP_DIR="${REMOTE_BACKUP_PATH}/daily.1"
LINK_DEST_OPTION=""
if $SSH_CMD "[ -d ${PREVIOUS_BACKUP_DIR} ]"; then
    # Utilise le répertoire de la veille pour créer des liens physiques pour le>
    LINK_DEST_OPTION="--link-dest=${REMOTE_BACKUP_PATH}/daily.1"
    echo "La sauvegarde précédente (daily.1) existe, utilisation de --link-dest>
else
    echo "La sauvegarde précédente n'existe pas, la première sauvegarde sera co>
fi

# 2. COMMANDE RSYNC
RSYNC_OPTIONS="-avz --delete" # a: archive, v: verbose, z: compression, --delet>
EXCLUDE_OPTIONS="--exclude '*/cache*' --exclude '*.log' --exclude 'lost+found'"

# Source 1 : /home
echo "2. Sauvegarde de /home..."
rsync $RSYNC_OPTIONS $EXCLUDE_OPTIONS $LINK_DEST_OPTION \
    /home/ \
    ${USER}@${HOST}:${REMOTE_BACKUP_PATH}/daily.0/home
# Vérification du statut de la commande rsync pour /home
if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: Rsync pour /home a échoué. Envoi d'une notification >
    echo "----------------------------------------"
    mail -s "[CRITIQUE] Echec Rsync VM - HOME" "$EMAIL_TO" < "$LOG_FILE"
    exit 1
fi

# Source 2 : /var/www
echo "3. Sauvegarde de /var/www..."
/usr/bin/rsync $RSYNC_OPTIONS $EXCLUDE_OPTIONS $LINK_DEST_OPTION \
    /var/www/ \
    ${USER}@${HOST}:${REMOTE_BACKUP_PATH}/daily.0/var_www

# Vérification du statut de la commande rsync pour /var/www
if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: Rsync pour /var/www a échoué. Envoi d'une notificati>
    echo "----------------------------------------"
    /usr/bin/mail -s "[CRITIQUE] Echec Rsync VM - VAR/WWW" "$EMAIL_TO" < "$LOG_>
    exit 1
fi

# 4. Finalisation
echo "----------------------------------------"
echo "Sauvegarde rsync terminée avec succès le $(date)."
# Optionnel: Envoyer un email de succès (décommenter si désiré)
# mail -s "[INFO] Rsync VM - Sauvegarde OK" "$EMAIL_TO" < "$LOG_FILE"

exit 0
```

### Planification Automatique (Cron)

La sauvegarde est lancée toutes les nuits à 03h00.

1.  **Permissions du script :**
    ```bash
    sudo chmod +x /usr/local/bin/backup_script.sh
    ```
2.  **Ajout de la tâche Cron :**
    ```bash
    sudo crontab -e
    ```
    Ajouter la ligne :
    ```cron
    0 3 * * * /usr/local/bin/backup_script.sh >> /var/log/rsync_backup.log 2>&1
    ```
Attention : quand vous faites un script qui sauvegarde vers une machine distante en ssh, bien vérifiez que les dossier existent où ont bien été créer ainsi qu'avoir les autorisations nécessaire. Sinon à chaque tentative d'exécution du script, il vous demandera un mdp root. 
+ quand on demande d'envoyer un mail, bien vérifier que le package mail est installé. Pour les chemins rsync et mail si problème recontré comme dans mon cas, nécessité de faire un which rsync et which mail pour avoir la localisation précise des repository.

-----

## 🗃️ Exercice 2 : Sauvegarde de Base de Données (MariaDB/MySQL)

Une sauvegarde web complète nécessite d'inclure les données dynamiques du site, généralement stockées dans une base de données.

### Script de Sauvegarde (`/usr/local/bin/backup_mariadb.sh`)

Tout d'abord, il faut créer la db : 

sudo mariadb

-- Création de la base de données de test
CREATE DATABASE db_test;

-- Création de l'utilisateur de sauvegarde et attribution d'un mot de passe fort
CREATE USER 'user_dump'@'localhost' IDENTIFIED BY 'VotreMotDePasseTresFort';

-- Accorder la permission de LECTURE à l'utilisateur pour cette base (suffisant pour le dump)
GRANT SELECT, LOCK TABLES ON db_test.* TO 'user_dump'@'localhost';

-- Appliquer les changements
FLUSH PRIVILEGES;

-- Quitter
EXIT;



### Script de Sauvegarde (`/usr/local/bin/backup_mariadb.sh`)

Ce script utilise l'outil `mysqldump` pour créer une copie cohérente de la base de données.

```bash
#!/bin/bash

# -----------------
# Configuration MariaDB
# -----------------
DB_USER="user_dump"
DB_PASS="VotreMotDePasseTresFort" # L'utilisateur créé ci-dessus
DB_NAME="db_test"

# -----------------
# Configuration Sauvegarde (Destination)
# -----------------
USER_SSH="hugo"
HOST_SSH="192.168.56.104"
REMOTE_BACKUP_PATH="/mnt/backups/mariadb" # Nouveau répertoire sur la destinati>
RETENTION=30 # Nombre de jours de rétention
DATE_STR=$(date +%Y%m%d)
LOG_FILE="$HOME/mariadb_backup_${DATE_STR}.log"

# Redirection de la sortie vers le fichier de log
exec > "$LOG_FILE" 2>&1
echo "--- MariaDB Backup started on $(date) ---"

# 1. ROTATION SUR LA DESTINATION (via SSH)
echo "1. Rotation des sauvegardes..."
SSH_CMD="ssh ${USER_SSH}@${HOST_SSH}"

# Supprime la plus ancienne (jour 30)
$SSH_CMD "rm -f ${REMOTE_BACKUP_PATH}/dump_${DB_NAME}.$(date --date='30 days ag>

# 2. CRÉATION DU DUMP LOCAL ET COMPRESSION GZIP
echo "2. Création du dump local de la base ${DB_NAME} et compression Gzip..."
LOCAL_DUMP_FILE="/tmp/dump_${DB_NAME}_${DATE_STR}.sql.gz"

# La commande mysqldump crée le dump, qui est immédiatement compressé par gzip
mysqldump -u ${DB_USER} -p${DB_PASS} ${DB_NAME} | gzip > ${LOCAL_DUMP_FILE}

if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: La création du dump local a échoué. Arrêt."
    # Logique d'envoi d'email ici si besoin
    exit 1
fi

# 3. TRANSFERT VERS LA DESTINATION
echo "3. Transfert vers la VM de destination..."
RSYNC_OPTIONS="-avz"
REMOTE_TARGET="${USER_SSH}@${HOST_SSH}:${REMOTE_BACKUP_PATH}/"

/usr/bin/rsync $RSYNC_OPTIONS ${LOCAL_DUMP_FILE} ${REMOTE_TARGET}

if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: Le transfert Rsync a échoué. Arrêt."
    # Logique d'envoi d'email ici si besoin
    exit 1
fi

# 4. NETTOYAGE ET FINALISATION
echo "4. Nettoyage du fichier local..."
rm -f ${LOCAL_DUMP_FILE}

echo "--- MariaDB Backup finished successfully on $(date) ---"
exit 0
```

### Planification Automatique (Cron)

La sauvegarde est lancée toutes les nuits à 03h00.

1.  **Permissions du script :**
    ```bash
    sudo chmod +x /usr/local/bin/backup_script.sh
    ```
2.  **Ajout de la tâche Cron :**
    ```bash
    sudo crontab -e
    ```
    Ajouter la ligne :
    ```cron
    0 3 * * * /usr/local/bin/backup_mariadb.sh
    ```

Voici les étapes concrètes pour effectuer une restauration de la base de données de test (`db_test`) depuis la VM de sauvegarde vers la VM source.

-----

## 🚀 Étapes de Restauration de MariaDB

### 1\. Localiser et Transférer le Dump

Connectez-vous à votre **VM source** et utilisez `scp` pour transférer le fichier dump compressé du jour (ou de la veille) depuis la VM de destination.

**Pré-requis :** Trouvez le nom du fichier de dump sur la VM de destination. Par exemple : `/mnt/backups/mariadb/dump_db_test.20251204.sql.gz`.

```bash
scp backupuser@192.168.50.2:/mnt/backups/mariadb/dump_db_test.20251204.sql.gz /tmp/
```

Le fichier dump est maintenant dans `/tmp/` sur votre VM source.

### 2\. Décompresser le Dump

Utilisez `gunzip` pour décompresser le fichier.

```bash
gunzip /tmp/dump_db_test.20251204.sql.gz
# Le fichier s'appelle maintenant : /tmp/dump_db_test.20251204.sql
```

### 3\. Importer la Base de Données

Vous pouvez maintenant ré-importer le contenu du fichier SQL directement dans MariaDB.

**A. Vider ou Recréer la Base (Sécurité)**
Pour garantir un état propre, vous pouvez vous connecter à MariaDB en tant que `root` pour recréer la base :

```sql
# Connectez-vous à MariaDB en tant que root
sudo mariadb

-- Suppression de l'ancienne base (si elle existe)
DROP DATABASE IF EXISTS db_test;

-- Recréation de la base
CREATE DATABASE db_test;

-- Quitter
EXIT;
```

**B. Exécuter l'Importation**
Utilisez la commande `mariadb` pour lire le fichier SQL et l'injecter dans la base de données :

```bash
# -u root utilise l'utilisateur root de MariaDB. 
# Si vous préférez utiliser l'utilisateur 'user_dump', ajustez le -u et le -p

sudo mariadb -u root db_test < /tmp/dump_db_test.20251204.sql
```

Le contenu de votre base de données est alors restauré à l'état exact du moment où le dump a été créé.

-----

## 🔑 Exercice 3 : Mise en Place de BorgBackup

**BorgBackup** est choisi pour l'étape de sauvegarde **chiffrée** et **dédupliquée**. Il est idéal pour la sauvegarde hors site ou sur des dépôts de grande taille.

### Configuration Borg

D'abord vous devez créer le repo de dépot sur la vm destination et lui donner les droits d'accès. Il faut installer borg sur la vm source et sur la destination sinon vous risquez d'avoir des messages d'erreur car il ne comprends pas les packets.

VM destination : 
sudo mkdir -p /mnt/backups/borg_repo
sudo chown backupuser:backupuser /mnt/backups/borg_repo

VM source :
sudo apt install borgbackup
borg init --encryption=repokey backupuser@192.168.50.2:/mnt/backups/borg_repo

### Script de Sauvegarde Borg (`/usr/local/bin/backup_borg.sh`)

```bash
#!/bin/bash

# -----------------
# Configuration
# -----------------
# REMPLACEZ VOTRE PHRASE DE PASSE ICI (Obligatoire pour l'automatisation)
# ATTENTION: Cette méthode est moins sécurisée que l'utilisation d'un fichier c>
# Pour les tests, cela fonctionne. Pour la production, utilisez BORG_PASSPHRASE>
export BORG_PASSPHRASE="Pz75OiKlm/,Ks5+"

# Chemin du dépôt
REPO="hugo@192.168.56.104:/mnt/backups/borg_repo"

# Nom de l'archive (format : 2025-12-04T10:30:00)
ARCHIVE_NAME="{now}" 

# Fichier de log local
LOG_FILE="$HOME/borg_backup_$(date +%Y%m%d).log"

# Redirection de la sortie vers le fichier de log
exec > "$LOG_FILE" 2>&1
echo "--- Borg Backup started on $(date) ---"

# 1. CRÉATION DE L'ARCHIVE
# Sauvegarde de /home et /var/www, exclusion des caches et logs
echo "1. Création de l'archive Borg..."

/usr/bin/borg create \
    --stats \
    --compression zstd,7 \
    --exclude-cache \
    --exclude '*/cache*' \
    --exclude '*.log' \
    --exclude '/var/www/tmp' \
    "$REPO::${ARCHIVE_NAME}" \
    /home \
    /var/www
    
if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: La création de l'archive Borg a échoué."
    # Logique d'envoi d'email ici
    exit 1
fi

# 2. ROTATION ET RÉTENTION (PRUNING)
# Garder 30 archives quotidiennes (jours)
echo "2. Gestion de la rétention (30 jours)..."

/usr/bin/borg prune \
    --list \
    --keep-daily 30 \
    "$REPO"

if [ $? -ne 0 ]; then
    echo "ERREUR CRITIQUE: La rotation Borg (prune) a échoué."
    # Logique d'envoi d'email ici
    exit 1
fi
echo "--- Borg Backup finished successfully on $(date) ---"
exit 0
```

### Planification Automatique (Cron)

1.  **Permissions du script :**
    ```bash
    sudo chmod +x /usr/local/bin/backup_borg.sh
    ```
2.  **Ajout de la tâche Cron :**
    ```bash
    sudo crontab -e
    ```
    Ajouter la ligne :
    ```
    0 3 * * * /usr/local/bin/backup_borg.sh
    ```

-----

## ❓ Justification des Choix

### Pourquoi des Scripts de Sauvegarde ?

L'utilisation de scripts **Bash** permet de :

1.  **Automatiser :** Lancer des tâches complexes (copie, rotation, nettoyage, vérification) sans intervention manuelle.
2.  **Répétabilité :** Garantir que le processus de sauvegarde est toujours exécuté de la même manière, éliminant les erreurs humaines.
3.  **Traçabilité :** Intégrer la journalisation (`>> /var/log/...`) pour documenter les succès et les échecs.

### Pourquoi des Connexions SSH ?

Les connexions SSH (Secure Shell) sont utilisées pour la sauvegarde distante (Rsync/Borg) pour deux raisons :

1.  **Chiffrement :** SSH chiffre tout le trafic entre la source et la destination. Cela assure la **confidentialité** des données même si elles transitent sur un réseau non sécurisé.
2.  **Authentification :** L'utilisation de paires de clés SSH (sans mot de passe) permet d'automatiser les scripts Cron de manière sécurisée, sans exposer de mots de passe dans les scripts.

-----

## 🌐 Mise en Place du Site Web

Un site web de test a été déployé pour simuler un environnement de production.

### HTTP (Insecure)

Le site a été initialement servi via le protocole **HTTP** (HyperText Transfer Protocol) sur le **port 80** par le serveur **Apache**.

  * **Problème :** Toutes les communications (y compris les mots de passe de connexion si un CMS est utilisé) sont envoyées en **texte clair** et sont donc vulnérables aux interceptions (*sniffing*).

sudo apt update
sudo apt install apache2 php libapache2-mod-php php-mysql
cd /var/www/html
sudo nano index.html

<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Site Local de Test</title>
</head>
<body>
    <h1>Bienvenue sur mon infrastructure de Test !</h1>
    <p>Ce site est opérationnel et accessible via le réseau interne.</p>
    <p>L'heure du serveur est : <?php echo date('H:i:s'); ?></p>
</body>
</html>

sudo mv index.html index.php
sudo chown -R www-data:www-data /var/www/html/

http://<IP_DE_VOTRE_VM_SOURCE>/
http://192.168.56.102/

Pour la restauration :

Pas besoin de modifier les scrips sachant que le site a été créer dans un repository sauvegardé par un script rsync et pour la restauration :

# 1. Définir la passphrase
export BORG_PASSPHRASE="VOTRE_PHRASE_DE_PASSE_SECURE"

# 2. Restaurer le fichier index.php dans /tmp/restore
/usr/bin/borg extract \
    /mnt/nas_backup/borg_repo::NOM_DE_LA_DERNIERE_ARCHIVE \
    /tmp/restore/ \
    --paths var/www/html/index.php

# 3. Copier le fichier restauré à son emplacement d'origine
sudo cp /tmp/restore/var/www/html/index.php /var/www/html/index.php

# 4. Nettoyer
sudo rm -rf /tmp/restore


### HTTPS (Secure)

La migration vers **HTTPS** (HTTP Secure) a été réalisée pour chiffrer la communication.

1.  **Génération du Certificat :** Un certificat **auto-signé** a été généré via OpenSSL sur la VM source, car l'infrastructure est locale.
2.  **Configuration Apache :** Le module SSL d'Apache a été activé (`a2enmod ssl`), et un hôte virtuel a été créé pour le **port 443** pointant vers le certificat.
3.  **Redirection :** Une redirection permanente a été mise en place dans le Virtual Host du port 80 pour forcer les navigateurs à utiliser HTTPS.

L'accès se fait maintenant via `https://<IP_DE_VOTRE_VM_SOURCE>/`, assurant le chiffrement même dans le labo.

VM Source : 

sudo mkdir -p /etc/apache2/ssl
cd /etc/apache2/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/apache2/ssl/apache.key -out /etc/apache2/ssl/apache.crt
sudo a2enmod ssl
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/mon-site-ssl.conf
sudo nano /etc/apache2/sites-available/mon-site-ssl.conf

Bien vérifiez que ces lignes soit dans <VirtualHost _default_:443>
```
# Le chemin de votre site
DocumentRoot /var/www/html

# Les chemins de la clé et du certificat créés à l'étape 1
SSLEngine on
SSLCertificateFile /etc/apache2/ssl/apache.crt
SSLCertificateKeyFile /etc/apache2/ssl/apache.key
```
sudo a2ensite mon-site-ssl.conf
sudo nano /etc/apache2/sites-available/000-default.conf

Dans <VirtualHost *:80> tout remplacez par cette ligne : 
Redirect permanent / https://votreadresseip/

sudo a2enmod rewrite
sudo systemctl restart apache2

-----

## 🛡️ Mise en Place de pfSense

L'étape finale a été d'introduire le pare-feu **pfSense** pour centraliser la sécurité et la gestion du réseau du laboratoire.

### Rôle et Architecture

1.  **VM Dédiée :** pfSense a été installé sur une VM séparée, configurée avec **deux cartes réseau (NIC)** :
      * **WAN :** Connectée à Internet (via NAT ou Bridge de l'hôte).
      * **LAN :** Connectée à un **Réseau Interne Virtuel** (`RESEAU_LABO`).
2.  **Passerelle Centrale :** Toutes les autres VMs (Source, Destination) ont été connectées **uniquement** à ce réseau interne, forçant tout leur trafic à passer par pfSense.

### Services Clés

  * **DHCP :** pfSense a été configuré pour distribuer automatiquement les adresses IP (ex: `192.168.1.100` à `.200`) et l'adresse de la Passerelle (`192.168.1.1`) à toutes les VMs du LAN.
  * **Pare-feu :** Le pare-feu bloque par défaut le trafic non sollicité, nous obligeant à **créer explicitement des règles `Pass`** sur l'interface LAN pour autoriser le trafic HTTP/HTTPS vers le site web de la VM source, et l'accès à l'interface web d'administration de pfSense elle-même.

La mise en place de pfSense a validé la capacité à gérer le routage, le NAT, le DHCP, et les règles de sécurité, offrant un contrôle total sur le flux de données de l'infrastructure de test.

Attention : quand vous faites une config DHCP sur pfsense, il faut penser que les interfaces vm ne sont plus config côté client et serveur. Le enp0s3 doit être mis en DHCP et enp0s8 supprimé sachant qu'on a plus que le réseau interne (ça c'était pour les interfaces mais bien penser avant sur virtualbox de retirer les cartes nat des vms serveurs et backup).

### Installation et mise en place 

Tout d'abord, il faut installer l'iso pfsense en amd64 , l'iso sera en format gz et devra être extrait grâce à 7zip. Pour la config de la vm, il faut une carte nat et une carte réseau interne (toutes les autres vm ont maintenant uniquement une carte réseau interne). Pour la nouvelle vm mettre en FreeBSD 64 bit.
Lorsque vous lancez la vm, suivez ces étapes :
- install
- auto (ufs)
- entire disk
- MBR dos partition
- finish
- commit
- reboot 
- éteignez la vm
- retirez l'iso
- relancez la 
- wan : em0
- lan : em1
- y

Maitenant sur une vm du réseau interne, connectez vous à l'interface web pfsense avec pour ip indiqué sur la carte lan (192.168.1.1) :
- username : admin 
- mdp : pfsense
- pfsene setup genera info next next
- hostname : fireball (jeu de mots avec firewall :) 
- domain : hugoFireball.com
- DNS 8.8.8.8
- DNS : 8.8.4.4
- Laissez cochez et cliquez next
- timezone choissisez paris
- next
- selectedType : DHCP
- next
- lan ip address : 192.168.1.1
- mask : 24
-  next
- mdp : je ne te le montre pas (c'est secret)
Puis faire next et finish.
Bien vérifiez que les vm soit bien configuré en dhcp sur leur interfaces et testez avec un ping vers google.com.
Modifiez les scripts de sauvegardes avec les nouvelles address attribués.
Il faudra lancer le firewall maintenant avant de lancer les autres vms.
