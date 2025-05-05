
# Déploiement Automatique d'une Application Node.js via GitHub Actions sur un VPS AWS avec Docker

## 1. Génération et Gestion des Clés SSH

### Générer une clé SSH locale :
```bash
ssh-keygen -t rsa
```

Ajoutez la clé publique à votre compte GitHub (dans *Settings > SSH and GPG keys*).

---

## 2. Création et Configuration du VPS AWS

1. Créez une instance EC2 Debian.
2. Connectez-vous à l’instance :
```bash
ssh -i <votre_cle.pem> admin@<ip-public-ec2>
```
3. Ajoutez votre clé publique dans `/home/<user>/.ssh/authorized_keys`.
4. Créez un nouvel utilisateur dédié au déploiement :
```bash
sudo adduser deployer
sudo usermod -aG sudo deployer
```
5. Configurez son accès SSH (copiez votre clé publique dans `/home/deployer/.ssh/authorized_keys`).
6. Installez Docker et Docker Compose :
```bash
sudo apt-get update
sudo apt install docker-compose docker.io
```
7. Ajoutez l’utilisateur au groupe Docker :
```bash
sudo usermod -aG docker deployer
```
8. Vérifiez l’installation :
```bash
docker run hello-world
```

---

## 3. Préparation des Secrets GitHub

1. Connectez-vous au VPS avec l’utilisateur `deployer`.
2. Générez une clé SSH :
```bash
ssh-keygen -t rsa
```
3. Ajoutez la **clé publique** à GitHub.
4. Encodez la **clé privée** :
```bash
cat ~/.ssh/id_rsa | base64 -w 0
```
5. Dans les *Secrets* du dépôt GitHub, ajoutez :

| Nom               | Valeur                       |
|------------------|------------------------------|
| `SSH_PRIVATE_KEY`| clé privée encodée en base64 |
| `SSH_USER`       | `deployer`                   |
| `SSH_HOST`       | IP publique EC2              |
| `WORK_DIR`       | `/home/deployer/app`         |
| `MAIN_BRANCH`    | `main`                       |

---

## 4. Configuration de Docker pour l’Application

### `Dockerfile`

```Dockerfile
# Utiliser une image de base (exemple avec Node.js)
FROM node:18-alpine
# Définir le répertoire de travail
WORKDIR /app
# Copier les fichiers package.json et package-lock.json pour installer les dépendances en premier
COPY package*.json ./
# Installer les dépendances
RUN npm install
# Copier le reste des fichiers du projet
COPY . .
# Exposer le port utilisé par l'application
EXPOSE 3000
# Démarrer l'application
CMD ["npm", "start"]
```

Le Dockerfile définit l’environnement d’exécution de l’application Node.js dans un conteneur Docker léger basé sur Alpine Linux. Voici les étapes principales :

Base Image : Utilise l'image ``` node:18-alpine ``` , une version allégée de Node.js adaptée pour la production.

Répertoire de travail : Définit ``` /app ``` comme répertoire de travail à l’intérieur du conteneur.

Installation des dépendances :

Copie les fichiers ``` package.json ``` et ``` package-lock.json ```.

Exécute ``` npm install  ```pour installer les dépendances de production.

Ajout du code source : Copie tous les fichiers de l’application dans le conteneur.

Exposition du port : Le port ``` 3000 ``` est exposé pour accéder à l’application.

Commande de démarrage : Lance l’application avec ``` npm start ```.

Ce Dockerfile permet de créer une image légère et optimisée, idéale pour un déploiement automatisé sur un VPS via GitHub Actions.


### `docker-compose.yml`

```yaml
services:
  app:
    build: .
    container_name: aws_test
    ports:
      - "3000:3000"
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
    command: npm start

  db:
    image: postgres
    volumes:
      - ./database:/var/lib/postgresql/data
    restart: always
    environment:
      POSTGRES_DB: qcmdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5435:5432"

  adminer:
    image: adminer
    restart: always
    ports:
      - "9000:8080"
```

Ce fichier docker-compose.yml définit un environnement de développement complet avec trois services principaux : l'application Node.js, une base de données PostgreSQL et l'outil de gestion Adminer.

🔹 Service app
``` build: ```  : Construit l’image Docker à partir du Dockerfile local.

 ``` container_name: aws_test ``` : Nomme explicitement le conteneur (utile pour les commandes Docker).

ports:

Redirige le port ``` 3000 ``` du conteneur vers le port ``` 3000 ``` de l’hôte, pour accéder à l’application.

volumes:

``` .:/app ``` : Monte le code source local dans le conteneur, facilitant le rechargement en développement.

``` /app/node_modules ``` : Préserve les dépendances installées dans le conteneur sans être écrasées par le volume du projet.

``` environment: ```

Définit la variable ``` NODE_ENV=development ``` pour activer les outils et logs utiles pendant le développement.

command:

Lance l’application avec npm start.

🔹 Service db
``` image: postgres```  : Utilise l’image officielle PostgreSQL.

``` volumes: ``` 

```./database:/var/lib/postgresql/data ```: Persiste les données de la base dans un dossier local.

```restart: always ```: Relance automatiquement le conteneur en cas de panne ou redémarrage du système.

```environment:```

```POSTGRES_DB=qcmdb``` : Nom de la base de données.

```POSTGRES_USER=postgres```, ```POSTGRES_PASSWORD=postgres``` : Identifiants par défaut.

ports:

Redirige le port ```5435``` de l’hôte vers le ```5432``` du conteneur PostgreSQL.

🔹 Service adminer
```image: adminer``` : Lance l’interface web Adminer, un outil léger pour administrer PostgreSQL.

``` restart: always ``` : Assure que l’interface est toujours disponible.

ports:

Redirige le port ``` 8080 ``` du conteneur vers le port ``` 9000 ``` de l’hôte, accessible via :
``` http://<IP>:9000 ```

Le fichier docker-compose.yml permet de gérer facilement le déploiement de l’application via Docker. Voici une explication des éléments utilisés :

``` version: '3' ``` : Spécifie la version du format de fichier Docker Compose (ici, v3, compatible avec la majorité des versions de Docker).

``` services: ``` : Définit les services à exécuter dans les conteneurs.
Ici, un seul service nommé web.

🔹 Service web
``` build: . ```: Indique que l’image Docker doit être construite à partir du Dockerfile situé à la racine du projet.

``` ports: ```

Redirige le port ``` 3000 ``` du conteneur vers le port ``` 3000 ``` de la machine hôte (le VPS), permettant d’accéder à l’application via ``` http://<IP>:3000 ```.

environment:

Définit une variable d’environnement ``` NODE_ENV ``` avec la valeur production, indiquant à Node.js de fonctionner en mode production (optimisé).


## 5. Ouverture des Ports et Vérification

1. Ouvrez le port `3000` dans le *Security Group* de l’instance EC2.
2. Vérifiez que le conteneur tourne :
```bash
docker ps
```
3. Accédez à votre application sur :  
``` http://<ip-public-ec2>:3000 ```

---

## 🟢 Fin Prête au Déploiement Continu !
