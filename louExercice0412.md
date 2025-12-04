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