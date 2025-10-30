# Documentation Apache Guacamole

[[_TOC_]]

Contexte : Mettre en place un serveur Bastion pour sécuriser les accès RDP/SSH des intervenants externes, sans divulguer les identifiants administrateurs. Le Bastion sera isolé dans une DMZ avec journalisation et captures vidéos des sessions.

## 1. Préparation et installation

### 1.1 Installation ISO
- Vérifier l’intégrité de l’image ISO avant installation.  
- Lancer l’installation standard. (ISO Debian13.1 hoisi)

### 1.2 Paramétrages réseau
 
- IP : 10.100.30.6/23
- Gateway : 10.100.31.254
- Serveur DNS :  Windows Server rôle DNS
- Nom FQDN : SVDL01-RDG-01.cgo.local

### 1.3 Configuration machine
- Joindre le poste au domaine (Domaine AD).
- Définir les utilisateurs :
- `root`
- `infra`
- etc.

### 1.4 Gestion du disque
- Mise en place du partitionnement avec **LVM**.  
- `/home`, `/var`, `/tmp` sur des partitions séparées.

### 1.5 Extension de partition
Exemple d’extension :
```bash
lvextend -L +2G /dev/vg_name/lv_name
resize2fs /dev/vg_name/lv_name
```
### 1.6 Renommer un volume group (VG)

Renommer l’ancien nom en nouveau nom :

```bash
vgrename ancien_nom nouveau_nom
```

Mettre à jour les fichiers de configuration :

```bash
/etc/fstab
/boot/grub/grub.cfg
```

### 1.7 Configuration des agents et du pare-feu
- Déployer les agents machine (Veeam, Supervision, etc.).
- Ajouter les règles nécessaires au pare-feu.
- Vérifier la communication avec **Internet / Serveur DNS**.

---

## 2. Installation et configuration d’Apache Guacamole


### 2.1 Prérequis
- Serveur sous Linux (Debian13 en l'occurrence)
- Accès administrateur (root ou via sudo)
- Serveur de base de données (MySQL/MariaDB ou PostgreSQL)
- Tomcat et Guacamole Server installés

---

### 2.2 Installation du serveur Guacamole avec docker

#### Étapes principales :
1. Installation des dépendances Docker :
```bash
sudo apt-get install apt-transport-https ca-certificates curl gnupg2 software-properties-common
```
2. Ajouter le dépôt officiel Docker
```bash
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
```bash
 echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
```
```bash
apt-get update
```

3. Installation des paquets Docker
```bash
apt-get install docker-ce docker-ce-cli containerd.io
```

>Démarrage automatique de Docker au lancement de machine
```bash
systemctl enable docker
```

### 2.3 Mise en place des conteneurs pour Apache Guacamole

1. Créer le dossier pour installer le serveur 
```bash
mkdir -p /opt/guacamole
```

2. Pull les images docker pour les conteneurs essentiels
```bash
pull guacamole/guacamole:latest (v1.6.0)
pull guacamole/guacd:latest (v1.6.0)
pull mysql:latest (v9.4.0)
```

3. Récupérer le script d'initialisation de la BDD
```bash 
docker run --rm guacamole/guacamole:latest /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
```

4. Créer le fichier `docker-compose.yml` initial pour la BDD :
```bash
services:
  guacdb:
    container_name: guacamoledb
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'mdproot'
      MYSQL_DATABASE: 'guacamole_db'
      MYSQL_USERNAME: 'mysql'
      MYSQL_PASSWORD: 'mdpmysql'
    volumes:
      - './db-data:/var/lib/mysql'
volumes:
  db-data:
```

Puis lancer le conteneur
```bash
docker compose up -d
```

5. Initialiser la BDD
```bash
docker cp initdb.sql guacamoledb:/initdb.sql
```
Puis stopper le conteneur
```bash
docker compose down
```

6. Compléter le fichier `docker-compose.yml` avec tous les services nécessaires :
```yaml
services:
  guacdb:
    container_name: guacamoledb
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'mdproot'
      MYSQL_DATABASE: 'guacamole_db'
      MYSQL_USERNAME: 'mysql'
      MYSQL_PASSWORD: 'mdpmysql'
    volumes:
      - './db-data:/var/lib/mysql'

  guacd:
    container_name: guacd
    image: guacamole/guacd:latest
    restart: always

  guacamole:
    container_name: guacamole
    image: guacamole/guacamole:latest
    restart: always
    expose:
      - "8080"
    environment:
      GUACD_HOSTNAME: "guacd"
      MYSQL_HOSTNAME: "guacdb"
      MYSQL_DATABASE: "guacamole_db"
      MYSQL_USERNAME: "mysql"
      MYSQL_PASSWORD: "mdpmysql"
      TOTP_ENABLED: "true"
    depends_on:
      - guacdb
      - guacd

volumes:
  db-data:
```

Puis relancer le conteneur
```bash
docker compose up -d
```

Et tester la page `http://ip_locale:8080/guacamole`.

## 3. Mise en place HTTPS + redirection HTTP -> HTTPS
(certificat déjà généré)

## 1. Installation et modules Apache
* **Installation apache 2 et démarrage au lancement**
    * `apt install apache2`
    * `systemctl enable apache2`

* **Activation des modules pour utiliser le reverse proxy**
    * `a2enmod proxy proxy_wstunnel proxy_http ssl rewrite`
    * `systemctl restart apache2`

## 2. Création et activation du site
* **Création du site en fichier `.conf`**
    * `nano /etc/apache2/sites-available/guacamole.conf`

* **Activation du site**
    * `a2ensite guacamole.conf`
    * `systemctl reload apache2`

* **Désactiver la page par défaut (la 80)**
    * *(Default) Pour éviter conflit avec docker et guacamole*
    * `a2dissite 000-default.conf`

* **Vérification**
    * Configuration finie, tester le site en 80 pour la redirection
    * puis en 443 pour voir s'il fonctionne

## 3. Exemple de Configuration (Reverse Proxy)

* **Schéma :** `nom du site` -> `Contenu` -> `backend`

### Fichier guacamole.conf

# Redirection de HTTP (80) vers HTTPS (443)
```apache
<VirtualHost *:80>
    Servername {FQDN-Serveur}
    Redirect permanent / https://{page-du-site-active}/
</VirtualHost>

# Configuration du reverse proxy en HTTPS

<VirtualHost *:443>
    Servername {FQDN-Serveur}

    # Redirige le /guacamole
    SSLEngine On
    SSLCertificateFile {lien vers certificat.pem}
    SSLCertificateKeyFile {lien vers clé_privée.pem}

    # Proxy principal
    ProxyPass / {lien-vers-page-web:port}/guacamole/ flushpackets=on
    ProxyPassReverse / {lien-vers-page-web:port}/guacamole/

    # Configuration spécifique pour Guacamole (websocket)
    <Location /websocket-tunnel>
        Order allow,deny
        Allow from all
        ProxyPass ws://{lien-vers-page-web}/guacamole/websocket-tunnel
        ProxyPassReverse ws://{lien-vers-page-web}/guacamole/websocket-tunnel
    </Location>
</VirtualHost>
```

## 4. Mise en place dossier de record pour les enregistrements vidéos RDP

### 1. Modification du fichier `docker-compose.yml`

Ajout sous les sections `services:` -> `guacamole:` et `guacd:`

```yaml
services:
  guacdb:
    container_name: guacamoledb
    image: mysql:latest
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'mdproot'
      MYSQL_DATABASE: 'guacamole_db'
      MYSQL_USERNAME: 'mysql'
      MYSQL_PASSWORD: 'mdpmysql'
    volumes:
      - './db-data:/var/lib/mysql'

  guacd:
    container_name: guacd
    image: guacamole/guacd:latest
    restart: always
    volumes:
      - /opt/guacamole/recordings:/var/lib/guacamole/recordings:rw

  guacamole:
    container_name: guacamole
    image: guacamole/guacamole:latest
    restart: always
    expose:
      - "8080"
    environment:
      GUACD_HOSTNAME: "guacd"
      MYSQL_HOSTNAME: "guacdb"
      MYSQL_DATABASE: "guacamole_db"
      MYSQL_USERNAME: "mysql"
      MYSQL_PASSWORD: "mdpmysql"
      TOTP_ENABLED: "true"
      RECORDING_ENABLED: "true"
    volumes:
      - /opt/guacamole/recordings:/var/lib/guacamole/recordings:ro
    depends_on:
      - guacdb
      - guacd

volumes:
  db-data:
```
### 2. Donner les bons droits pour lire/écrire dans le `/recordings`
* Les droits se mettent sur l'hôte alors qu'ils seront associés aux utilisateurs dans les conteneurs, en l'occurrence donner les droits écritures et lectures à l'utilisateur guacd, c'est lui qui fera les records des vidéos.
* Pour le savoir, se diriger directement sur le conteneur en question, et regarder l'UID et GID besoin.

Commande pour aller en session interactive sur le conteneur choisi : 
```bash
docker exec -it <container> sh
```

Lister les UID et GID du conteneur : 

```bash
cat /etc/passwd
```

Pour le dossier `/recordings` on doit changer les owner (Group et Utilisateur), pour cela on va mettre en Owner l'utilisateur guacd et groupe guacamole, si on fait les commandes précédentes, on remarque l'UID de guacd(1000) et le GID de guacamole(1001).

* Changer alors les owners via la commande associée

```bash
chown -R 1000:1001 /opt/guacamole/recordings
```

Ensuite on associe les droits lectures / écritures : 
```bash
chmod -R 2750 /opt/guacamole/recordings
```

