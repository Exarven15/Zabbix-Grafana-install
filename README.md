# Installation d’un environnement Zabbix + Grafana sur une seule VM Debian 12
Dans ce projet, j’installe et configure un environnement complet de supervision basé sur Zabbix et Grafana.
Le but est de mettre en place un lab fonctionnel sur une seule VM Debian 12 afin de réaliser des tests et se familiariser avec ces outils.

---
## 1. Configuration de la VM
Pour ce lab, j’utilise une machine virtuelle Debian 12 avec les caractéristiques suivantes :
- 2 à 4 vCPU
- 6 à 8 Go de RAM
- 80 Go de stockage SSD
- Debian 12 Bookworm (64 bits)

---
## 2. Préparation du système
Je commence par mettre le système à jour et installer les outils nécessaires.
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget curl vim gnupg lsb-release software-properties-common apt-transport-https -y
sudo timedatectl set-timezone Europe/Paris
```

---
## 3. Installation de Zabbix, MariaDB et du frontend web
### 3.1 Ajout du dépôt Zabbix
```bash
wget https://repo.zabbix.com/zabbix/7.0/debian/pool/main/z/zabbix-release/zabbix-release_7.0-2+debian12_all.deb
sudo dpkg -i zabbix-release_7.0-2+debian12_all.deb
sudo apt update
```

---
### 3.2 Installation des paquets Zabbix, MariaDB et Apache
```bash
sudo apt install -y zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent mariadb-server
```

---
### 3.3 Sécurisation de MariaDB
sudo mysql_secure_installation


J’accepte les options recommandées.

3.4 Création de la base de données Zabbix
sudo mysql -uroot -p


Dans l’interface MySQL :

CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'ZabbixPass123!';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;

3.5 Importation du schéma Zabbix
zcat /usr/share/zabbix-sql-scripts/mysql/server.sql.gz | mysql -uzabbix -pZabbixPass123! zabbix

3.6 Configuration du serveur Zabbix
sudo nano /etc/zabbix/zabbix_server.conf


Je modifie la ligne suivante :

DBPassword=ZabbixPass123!

3.7 Activation des services
sudo systemctl enable zabbix-server zabbix-agent apache2 mariadb
sudo systemctl restart zabbix-server zabbix-agent apache2 mariadb

4. Configuration du frontend Zabbix

J’accède à l’interface web :

http://<IP_VM>/zabbix


Dans l’assistant :

Type de base : MySQL

Hôte : localhost

Nom : zabbix

Utilisateur : zabbix

Mot de passe : ZabbixPass123!

Nom du serveur : LabZabbix

Identifiants de connexion :

Admin / zabbix

5. Installation de Grafana
5.1 Ajout du dépôt officiel Grafana
sudo wget -q -O /usr/share/keyrings/grafana.key https://packages.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update

5.2 Installation et démarrage du service
sudo apt install grafana -y
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

5.3 Accès à l’interface Grafana
http://<IP_VM>:3000


Identifiants par défaut :

admin / admin


Je définis ensuite mon nouveau mot de passe (ex : admin1).

6. Intégration de Zabbix dans Grafana
6.1 Installation du plugin Zabbix
sudo grafana-cli plugins install alexanderzobnin-zabbix-app
sudo systemctl restart grafana-server

6.2 Ajout de la datasource Zabbix

Dans Grafana :

Configuration

Data Sources

Add data source

Choix : Zabbix

Paramètres :

URL API Zabbix :

http://localhost/zabbix/api_jsonrpc.php


Identifiants API :

Login : Admin
Password : zabbix


Je valide avec "Save & Test".

7. Test du monitoring

Je teste un incident en arrêtant l’agent Zabbix :

sudo systemctl stop zabbix-agent


Zabbix affiche l’alerte dans Monitoring → Problems.

Je relance ensuite le service :

sudo systemctl start zabbix-agent

8. Mise en place d’un script de sauvegarde

Je crée un script automatisé :

sudo nano /usr/local/bin/backup_zabbix.sh


Contenu :

#!/bin/bash
DATE=$(date +%F)
BACKUP_DIR="/var/backups/zabbix"
mkdir -p $BACKUP_DIR
mysqldump -uzabbix -pZabbixPass123! zabbix > $BACKUP_DIR/zabbix_$DATE.sql
tar czf $BACKUP_DIR/zabbix_conf_$DATE.tar.gz /etc/zabbix /usr/share/zabbix
find $BACKUP_DIR -type f -mtime +7 -delete


Activation :

sudo chmod +x /usr/local/bin/backup_zabbix.sh
sudo crontab -e


Ajout dans cron :

0 3 * * * /usr/local/bin/backup_zabbix.sh

9. Vérification des mises à jour

Je crée un script permettant de vérifier les mises à jour Zabbix et Grafana.

sudo nano /usr/local/bin/check_updates.sh


Contenu :

#!/bin/bash
echo "=== Zabbix ==="
zabbix_server --version
apt list --upgradable 2>/dev/null | grep zabbix

echo "=== Grafana ==="
grafana-server -v
apt list --upgradable 2>/dev/null | grep grafana


Activation :

sudo chmod +x /usr/local/bin/check_updates.sh


Exécution :

/usr/local/bin/check_updates.sh

10. Conclusion

Cette installation me permet de disposer d’un environnement complet de supervision :

Zabbix installé et fonctionnel

Base MariaDB configurée

Interface web opérationnelle

Grafana installé et intégré à Zabbix

Plugin Zabbix pour Grafana

Script de sauvegarde automatisé

Script de vérification des mises à jour

Tests de supervision validés

Cet environnement me sert de base pour mes futurs travaux de monitoring et d’analyse de performance.
