# Configuration d'un NAS avec RAID 5, Samba, SFTP et WebDAV

## Introduction
Un **Network Attached Storage (NAS)** est un appareil de stockage autonome connecté à un réseau privé ou professionnel. Il permet de sauvegarder, partager et sécuriser des fichiers accessibles depuis plusieurs appareils.

Ce guide détaille la configuration d'un NAS avec **RAID 5**, **Samba**, **SFTP** et **WebDAV** sur une machine virtuelle sous Linux.

---
## 1. Configuration du RAID 5

### Installation de `mdadm`
Mise à jour de la machine :
```sh
sudo apt update && sudo apt upgrade
```
Installation de `mdadm` :
```sh
sudo apt install mdadm
```
Vérification de l'installation :
```sh
sudo mdadm -V
```
### Création de la matrice RAID 5
Identification des disques :
```sh
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT
```
Création du RAID 5 avec 3 disques :
```sh
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
```
### Création et montage du système de fichiers
```sh
sudo mkfs.ext4 -F /dev/md0
sudo mkdir -p /mnt/md0
sudo mount /dev/md0 /mnt/md0
df -h -x devtmpfs -x tmpfs
```
### Configuration pour le démarrage automatique
Enregistrement de la configuration RAID :
```sh
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
sudo update-initramfs -u
```
Ajout au fichier `/etc/fstab` :
```sh
echo '/dev/md0 /mnt/md0 ext4 defaults,nofail,discard 0 0' | sudo tee -a /etc/fstab
```

---
## 2. Configuration de Samba

### Installation de Samba
```sh
sudo apt update
sudo apt install samba
```
Ajout d'un utilisateur Samba :
```sh
sudo smbpasswd -a nomutilisateur
```
Création d'un dossier partagé :
```sh
mkdir /mnt/md0/nomdudossier
```
Configuration de Samba :
```sh
sudo nano /etc/samba/smb.conf
```
Ajout du partage :
```
[nomdudossier]
path = /mnt/md0/nomdudossier
read only = no
guest ok = no
valid users = nomutilisateur
```
Redémarrage de Samba :
```sh
sudo systemctl restart smbd.service
```
Test de l'accès :
```sh
smbclient -U nomutilisateur //[IP_address]/nomdudossier -c 'ls'
```

---
## 3. Sécurisation avec SFTP

### Installation de ProFTPD
```sh
sudo apt update
sudo apt install proftpd -y
sudo systemctl status proftpd
```
### Configuration du pare-feu
```sh
sudo apt install ufw -y
sudo ufw enable
sudo ufw allow 6500/tcp
```
Modification de `/etc/ssh/sshd_config` :
```
Port 6500
AllowUsers nomutilisateur
```
Redémarrage des services :
```sh
sudo systemctl restart proftpd && sudo systemctl restart ssh
```
Test de connexion :
```sh
sftp -P 6500 nomutilisateur@IP_Adress
```
Création des répertoires dédiés dans le RAID :
```sh
sudo mkdir -p /mnt/md0/sftp/nomutilisateur
sudo chown nomutilisateur:nomutilisateur /mnt/md0/sftp/nomutilisateur
```

---
## 4. Configuration de WebDAV

### Installation d'Apache2 et activation de WebDAV
```sh
sudo apt update
sudo apt install apache2
sudo a2enmod dav_fs
```
Création du répertoire WebDAV :
```sh
sudo mkdir /var/www/webdav
sudo sh -c 'echo "Bienvenue depuis WebDAV.local" > /var/www/webdav/index.html'
sudo chown www-data:www-data /var/www/webdav
```
Création de la configuration du site :
```sh
sudo nano /etc/apache2/sites-available/webdav.local.conf
```
Ajout du contenu suivant :
```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName webdav.local
    DocumentRoot /var/www/webdav
    <Directory />
        Options FollowSymLinks
        AllowOverride None
    </Directory>
    <Directory /var/www/webdav/>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride None
        Order allow,deny
        allow from all
    </Directory>
</VirtualHost>
```
Activation du site et rechargement d'Apache2 :
```sh
sudo a2ensite webdav.local
sudo service apache2 reload
```
### Activation et test de WebDAV
Ajout du répertoire WebDAV :
```sh
sudo mkdir /var/www/webdav/svn
sudo touch /var/www/webdav/svn/linuxconfig.txt
sudo chown www-data:www-data /var/www/webdav/svn
```
Modification du fichier `/etc/apache2/sites-available/webdav.local.conf` :
```
Alias /svn /var/www/webdav/svn
<Location /svn>
    DAV On
</Location>
```
Redémarrage du serveur :
```sh
sudo service apache2 restart
```
Test via : `http://[IP_Serveur]/svn`

### Configuration de l'authentification WebDAV
```sh
sudo mkdir /usr/local/apache2/
sudo htpasswd -c /usr/local/apache2/webdav.passwords nomutilisateur
```
Ajout dans la configuration Apache :
```
<Location /svn>
    DAV On
    AuthType Basic
    AuthName "webdav"
    AuthUserFile /usr/local/apache2/webdav.passwords
    Require valid-user
</Location>
```
Redémarrage :
```sh
sudo service apache2 restart
```
Test d'accès avec authentification WebDAV.

---
## Conclusion
Vous avez maintenant un NAS sécurisé avec **RAID 5**, **Samba**, **SFTP** et **WebDAV**. Ce guide couvre la configuration et la sécurisation pour garantir un accès distant fiable et sécurisé.


