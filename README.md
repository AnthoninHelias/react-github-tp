
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
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 3000
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
version: '3'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
```

---

## 5. Ouverture des Ports et Vérification

1. Ouvrez le port `3000` dans le *Security Group* de l’instance EC2.
2. Vérifiez que le conteneur tourne :
```bash
docker ps
```
3. Accédez à votre application sur :  
http://<ip-public-ec2>:3000

---

## 🟢 Fin Prête au Déploiement Continu !
