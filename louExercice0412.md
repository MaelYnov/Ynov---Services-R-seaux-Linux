### Exercice 1 : Mise en place d'une stratégie rsync


Créez un script de sauvegarde avec rsync qui :

Sauvegarde /home et /var/www

Exclut les fichiers .log et les dossiers cache

Garde 7 jours de rétention
Envoie un email en cas d'erreur



Automatisez avec cron (exécution à 3h du matin)


Testez la restauration d'un fichier supprimé


=> 

### EXERCICE 1 ###
Installe d’abord les outils nécessaires :
```
sudo apt update
sudo apt install rsync mailutils -y
```

Vérifie que l’email local fonctionne :
```
echo "Test email" | mail -s "Test" root
```


✅ 2. Structure des sauvegardes

On va stocker les sauvegardes dans :
```
/backup/rsync/
```

Avec une sauvegarde par jour :
```
/backup/rsync/2025-12-04/
/backup/rsync/2025-12-05/
```

Crée le dossier :
```
sudo mkdir -p /backup/rsync
sudo chmod 700 /backup
```


✅ 3. Script de sauvegarde rsync

Crée le script :
```
sudo nano /usr/local/bin/backup_rsync.sh
```

Colle ceci :
```
#!/bin/bash

DATE=$(date +"%Y-%m-%d")
BACKUP_DIR="/backup/rsync/$DATE"
LOG_FILE="/var/log/backup_rsync.log"
EMAIL="root"

mkdir -p "$BACKUP_DIR"

rsync -av --delete \
--exclude="*.log" \
--exclude="cache/" \
/home /var/www \
"$BACKUP_DIR" >> "$LOG_FILE" 2>&1

# Vérification d'erreur
if [ $? -ne 0 ]; then
    echo "Erreur lors de la sauvegarde du $DATE" | mail -s "ERREUR SAUVEGARDE RSYNC" "$EMAIL"
fi

# Suppression des sauvegardes de plus de 7 jours
find /backup/rsync/* -maxdepth 0 -type d -mtime +7 -exec rm -rf {} \;
```

✅ 4. Rendre le script exécutable
```
sudo chmod +x /usr/local/bin/backup_rsync.sh
```

✅ 5. Test manuel du script

Lance-le une fois à la main :
```
sudo /usr/local/bin/backup_rsync.sh
```

Vérifie :
```
ls /backup/rsync/
cat /var/log/backup_rsync.log
```

✅ 6. Automatisation avec CRON (3h du matin)

Édite le cron root :
```
sudo crontab -e
```

Ajoute cette ligne :
```
0 3 * * * /usr/local/bin/backup_rsync.sh
```

✅ Cela signifie :

0 → minute

3 → 3h

* * * → tous les jours


✅ 7. Test de la rétention 7 jours

Simule une vieille sauvegarde :
```
sudo mkdir /backup/rsync/OLD
sudo touch -d "10 days ago" /backup/rsync/OLD
sudo /usr/local/bin/backup_rsync.sh
```

Vérifie que OLD a disparu :
```
ls /backup/rsync/
```


✅ 8. Test de restauration d’un fichier supprimé
1️⃣ Crée un fichier test :
```
sudo touch /home/test_restore.txt
sudo /usr/local/bin/backup_rsync.sh
```
2️⃣ Supprime-le :
```
rm /home/test_restore.txt
```
3️⃣ Restaure-le depuis la sauvegarde :
```
cp /backup/rsync/$(date +"%Y-%m-%d")/home/test_restore.txt /home/
```
4️⃣ Vérifie :
```
ls /home/test_restore.txt
```

✅ Restauration validée !

✅ Récapitulatif de ce que tu as validé pour ton prof

✔ Sauvegarde de /home et /var/www
✔ Exclusion des .log et des dossiers cache
✔ Rétention automatique sur 7 jours
✔ Envoi d’email en cas d’erreur
✔ Automatisation via cron à 3h
✔ Test réel de restauration


### EXERCICE 2 ###

✅ Installation MariaDB
✅ Création d’une base de test
✅ Script de sauvegarde :

Dump complet quotidien

Compression gzip

Rotation 30 jours
✅ Simulation de corruption
✅ Restauration complète
✅ (Option bonus : cron automatique)

✅ 1. Installation de MariaDB
```
sudo apt update
sudo apt install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Sécurise rapidement :
```
sudo mysql_secure_installation
```

Tu peux répondre :

Mot de passe root → oui

Supprimer users anonymes → oui

Interdire root distant → oui

Supprimer DB test → oui

Reload privileges → oui

✅ 2. Création d’une base de test

Connexion :
```
sudo mysql -u root -p
```

Puis dans MariaDB :
```
CREATE DATABASE test_backup;
USE test_backup;

CREATE TABLE utilisateurs (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nom VARCHAR(50),
    email VARCHAR(100)
);

INSERT INTO utilisateurs (nom, email) VALUES
('Alice', 'alice@test.com'),
('Bob', 'bob@test.com');

EXIT;
```

Vérification :
```
sudo mysql -u root -p test_backup -e "SELECT * FROM utilisateurs;"
```

✅ Base fonctionnelle !
```
lou@client3:~/Desktop$ sudo mysql -u root -p test_backup -e "SELECT * FROM utilisateurs;"
Enter password: 
+----+-------+----------------+
| id | nom   | email          |
+----+-------+----------------+
|  1 | Alice | alice@test.com |
|  2 | Bob   | bob@test.com   |
+----+-------+----------------+
```

✅ 3. Dossier de sauvegarde
sudo mkdir -p /backup/mariadb
sudo chmod 700 /backup/mariadb

✅ 4. Script de sauvegarde MariaDB

Crée le script :
```
sudo nano /usr/local/bin/backup_mariadb.sh
```

Colle ceci :
```
#!/bin/bash

DATE=$(date +"%Y-%m-%d")
BACKUP_DIR="/backup/mariadb"
DB_USER="root"
DB_PASS="04042006"
LOG_FILE="/var/log/backup_mariadb.log"

FILENAME="mariadb_backup_$DATE.sql.gz"

# Dump + compression
mysqldump -u "$DB_USER" -p"$DB_PASS" --all-databases | gzip > "$BACKUP_DIR/$FILENAME" 2>> "$LOG_FILE"

# Vérification d'erreur
if [ $? -ne 0 ]; then
    echo "Erreur sauvegarde MariaDB du $DATE" | mail -s "ERREUR BACKUP MARIADB" root
fi

# Rotation > 30 jours
find "$BACKUP_DIR" -type f -name "*.gz" -mtime +30 -delete
```

✅ 5. Rendre le script exécutable
```
sudo chmod +x /usr/local/bin/backup_mariadb.sh
```
✅ 6. Test manuel de la sauvegarde
sudo /usr/local/bin/backup_mariadb.sh


Vérifie :
```
ls -lh /backup/mariadb
```
->
```
lou@client3:~/Desktop$ sudo ls -lh /backup/mariadb
total 512K
-rw-r--r-- 1 root root 510K Dec  4 12:04 mariadb_backup_2025-12-04.sql.gz
```

Tu dois voir un fichier du type :
```
mariadb_backup_2025-12-04.sql.gz
```

Test de validité du dump :
```
gunzip -c /backup/mariadb/mariadb_backup_$(date +"%Y-%m-%d").sql.gz | head
```

✅ Si tu vois des lignes SQL → c’est bon !

```
lou@client3:~/Desktop$ sudo gunzip -c /backup/mariadb/mariadb_backup_$(date +"%Y-%m-%d").sql.gz | head
/*M!999999\- enable the sandbox mode */ 
-- MariaDB dump 10.19  Distrib 10.11.13-MariaDB, for debian-linux-gnu (x86_64)
--
-- Host: localhost    Database: 
-- ------------------------------------------------------
-- Server version	10.11.13-MariaDB-0ubuntu0.24.04.1

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
```
✅ 7. Automatisation avec CRON

Édite le cron root :
```
sudo crontab -e
```
Ajoute :
```
0 2 * * * /usr/local/bin/backup_mariadb.sh
```

➡️ Sauvegarde tous les jours à 2h du matin.

✅ 8. Simulation d’une corruption

⚠️ ON CASSE LA BASE exprès.

Entre dans MariaDB :
```
sudo mysql -u root -p

DROP DATABASE test_backup;
EXIT;
```

Vérifie :
```
sudo mysql -u root -p -e "SHOW DATABASES;"
```

✅ test_backup a disparu → corruption simulée !

```
lou@client3:~/Desktop$ sudo mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 45
Server version: 10.11.13-MariaDB-0ubuntu0.24.04.1 Ubuntu 24.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> DROP DATABASE test_backup;
Query OK, 1 row affected (0.018 sec)

MariaDB [(none)]> EXIT;
Bye
lou@client3:~/Desktop$ sudo mysql -u root -p -e "SHOW DATABASES;"
Enter password: 
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
```
✅ 9. Restauration complète depuis la sauvegarde

Décompression + restauration :
```
gunzip -c /backup/mariadb/mariadb_backup_$(date +"%Y-%m-%d").sql.gz | sudo mysql -u root -p
```
✅ 10. Vérification finale
sudo mysql -u root -p test_backup -e "SELECT * FROM utilisateurs;"

```
lou@client3:~/Desktop$ sudo mysql -u root -p test_backup -e "SELECT * FROM utilisateurs;"
Enter password: 
+----+-------+----------------+
| id | nom   | email          |
+----+-------+----------------+
|  1 | Alice | alice@test.com |
|  2 | Bob   | bob@test.com   |
+----+-------+----------------+

```
✅ Si tu retrouves Alice et Bob → RESTOURATION VALIDÉE 🎉

✅ Ce que tu peux écrire dans ton compte-rendu

✔ Installation et sécurisation de MariaDB
✔ Création d’une base de test fonctionnelle
✔ Script automatique de sauvegarde avec :

Dump complet

Compression gzip

Rotation 30 jours
✔ Simulation d’une perte de données
✔ Restauration complète et vérifiée

### Exercice 3 ###

✅ 1. Installation de BorgBackup

Sur Debian/Ubuntu :
```
sudo apt update
sudo apt install borgbackup -y
```

Vérifie l’installation :
```
borg --version
```
✅ 2. Initialisation d’un dépôt chiffré

On va créer un dépôt dans /backup/borg_repo. Borg permet le chiffrement intégré.
```
sudo mkdir -p /backup/borg_repo
sudo borg init --encryption=repokey /backup/borg_repo
```

--encryption=repokey → chiffrement avec une clé stockée dans le dépôt (plus simple que passphrase seule)

Borg va te demander un mot de passe pour le chiffrement : note-le bien, tu en auras besoin pour la restauration.

mdp : aze

Vérifie le dépôt :
```
borg list /backup/borg_repo
```

Pour l’instant il sera vide.

✅ 3. Créer des sauvegardes quotidiennes pendant 3 jours

On va créer un dossier de test à sauvegarder, par exemple /home/lou/test_borg :
```
mkdir -p ~/test_borg
echo "Version 1" > ~/test_borg/fichier.txt
```
Sauvegarde jour 1 :
```
borg create --stats /backup/borg_repo::jour1 ~/test_borg
```
->
```
ou@client3:~/Desktop$ sudo borg create --stats /backup/borg_repo::jour1 ~/test_borg
Enter passphrase for key /backup/borg_repo: 
------------------------------------------------------------------------------
Repository: /backup/borg_repo
Archive name: jour1
Archive fingerprint: fd309f9803df168461228a455524b0a839c55df0a39f9eba13884479a2c1b470
Time (start): Thu, 2025-12-04 12:33:33
Time (end):   Thu, 2025-12-04 12:33:33
Duration: 0.02 seconds
Number of files: 1
Utilization of max. archive size: 0%
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
This archive:                  527 B                578 B                578 B
All archives:                   10 B                 53 B                817 B

                       Unique chunks         Total chunks
Chunk index:                       3                    3
------------------------------------------------------------------------------
```
Ajoute une modification pour jour 2 :
```
echo "Version 2" > ~/test_borg/fichier.txt
borg create --stats /backup/borg_repo::jour2 ~/test_borg
```

Ajoute une modification pour jour 3 :
```
echo "Version 3" > ~/test_borg/fichier.txt
borg create --stats /backup/borg_repo::jour3 ~/test_borg
```

--stats → affiche les statistiques, dont la taille sauvegardée et la déduplication

✅ 4. Observer la déduplication

Liste les archives dans le dépôt :
```
borg list /backup/borg_repo
```
->
```
lou@client3:~/Desktop$ sudo borg list /backup/borg_repo
Enter passphrase for key /backup/borg_repo: 
jour1                                Thu, 2025-12-04 12:33:33 [fd309f9803df168461228a455524b0a839c55df0a39f9eba13884479a2c1b470]
jour2                                Thu, 2025-12-04 12:36:07 [f8cd17e644bc6ba70c172e40f1015aa8c124348c519fff4ec7d87444d964a4c0]
jour3                                Thu, 2025-12-04 12:36:44 [cbc3a31db93ef1af0e0f13073813710e34f4dfc5c8b9bde6ef6ad944d50c4f3c]
```
Pour voir les tailles et l’espace économisé :
```
borg info /backup/borg_repo
```
-> 
```
lou@client3:~/Desktop$ sudo borg info /backup/borg_repo
Enter passphrase for key /backup/borg_repo: 
Repository ID: 1c1cb28b228c08c9a278ac9d16dd450cea5529009ed41a6b1e8dc72dd46613e8
Location: /backup/borg_repo
Encrypted: Yes (repokey)
Cache: /root/.cache/borg/1c1cb28b228c08c9a278ac9d16dd450cea5529009ed41a6b1e8dc72dd46613e8
Security dir: /root/.config/borg/security/1c1cb28b228c08c9a278ac9d16dd450cea5529009ed41a6b1e8dc72dd46613e8
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
All archives:                   30 B                159 B              2.48 kB

                       Unique chunks         Total chunks
Chunk index:                       9                    9
```

Tu verras que les données inchangées ne sont pas dupliquées, Borg utilise la déduplication par blocs.

✅ 5. Restaurer une version spécifique d’un fichier

Supposons qu’on veuille restaurer la version du jour 2 :
```
borg extract /backup/borg_repo::jour2 ~/test_borg/fichier.txt
```

Vérifie :
```
cat ~/test_borg/fichier.txt
# Devrait afficher "Version 2"
```

Si tu veux restaurer tous les fichiers de l’archive :
```
borg extract /backup/borg_repo::jour2
```

->
```
lou@client3:~/Desktop$ sudo borg extract /backup/borg_repo::jour2 ~/test_borg/fichier.txt
Enter passphrase for key /backup/borg_repo: 
lou@client3:~/Desktop$ cat ~/test_borg/fichier.txt
Version 3
```

✅ 6. Bonus : Automatiser les sauvegardes quotidiennes avec cron

Édite le cron de ton utilisateur :
```
crontab -e
```

Ajoute par exemple pour 4h du matin :
```
0 4 * * * borg create --stats /backup/borg_repo::$(date +\%Y-\%m-\%d) ~/test_b
```

### Exercice 4 ###

🟦 1) Inventaire des systèmes et des données

Mon infrastructure est composée de plusieurs machines virtuelles sous Ubuntu.

| Machine     | Rôle                  | Données sauvegardées | Dossiers        |
| ----------- | --------------------- | -------------------- | --------------- |
| VM-Web      | Serveur web Apache    | Site web             | `/var/www/html` |
| VM-DB       | Serveur base MySQL    | Bases de données     | MySQL           |
| VM-Backup   | Serveur de sauvegarde | Sauvegardes          | `/backup`       |
| Poste admin | Administration        | Scripts              | `/home`         |

Données critiques :
- Fichiers du site web
- Bases de données MySQL
- Comptes utilisateurs Linux
- Fichiers de configuration système

🎯 Objectifs :
- RPO : 24 heures
- RTO : 2 heures

🟧 2) Procédures de sauvegarde
✅ Création du dossier de sauvegarde
```
sudo mkdir -p /backup
```
✅ Sauvegarde des fichiers système et du site web
```
sudo rsync -av --delete /home /var/www /etc /backup/
```

Cette commande permet de sauvegarder :
- les utilisateurs
- le site web
- la configuration système

✅ Sauvegarde de la base de données MySQL
```
sudo mysqldump -u root -p --all-databases | gzip > /backup/mysql.sql.gz
```
✅ Automatisation tous les jours avec CRON
```
crontab -e
```

Puis ajout :
```
0 2 * * * rsync -av --delete /home /var/www /etc /backup/
0 3 * * * mysqldump -u root -pPASSWORD --all-databases | gzip > /backup/mysql.sql.gz
```

🟧 3) Procédures de restauration
🔴 Restauration du site web
```
sudo rsync -av /backup/var/www/ /var/www/

sudo systemctl restart apache2
```
🔴 Restauration des utilisateurs
```
sudo rsync -av /backup/home/ /home/
```
🔴 Restauration complète de la base MySQL
```
gunzip < /backup/mysql.sql.gz | sudo mysql -u root -p
```
🔴 Restauration complète après attaque ou crash
```
sudo systemctl stop apache2 mysql
sudo rsync -av --delete /backup/ /
sudo reboot
```


🟨 4) Tests à effectuer
✅ Test 1 : Suppression du site puis restauration
```
sudo rm /var/www/html/index.html
```

Puis :
```
sudo rsync -av /backup/var/www/html/index.html /var/www/html/
```

✅ Le site fonctionne → test validé

✅ Test 2 : Suppression d’une base MySQL
```
sudo mysql -u root -p
DROP DATABASE test;
EXIT
```

Puis restauration :
```
gunzip < /backup/mysql.sql.gz | mysql -u root -p
```

✅ La base est restaurée

✅ Test 3 : Vérification de la présence des sauvegardes
```
ls /backup
```
🟨 5) Contacts et responsabilités
| Rôle                   | Nom  | Mission                    |
| ---------------------- | ---- | -------------------------- |
| Administrateur système | Moi  | Sauvegarde et restauration |
| Responsable sécurité   | Prof | Analyse incident           |
| Responsable IT         | Prof | Validation finale          |


✅ Conclusion

- Ce Plan de Reprise d’Activité me permet de :
- Protéger mes données
- Restaurer un site après une panne
- Restaurer une base MySQL après une corruption
- Tester régulièrement les sauvegardes

J’utilise :
- rsync pour les fichiers
- mysqldump pour MySQL
- cron pour l’automatisation

Une sauvegarde est considérée comme valide uniquement après un test de restauration réussi.