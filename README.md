# TP4 : Docker et Docker Compose, déploiement d'une application MERN

**Étudiant :** bahri yassine
**Classe :** 3ème année LIG, ISG de Sousse
**Matière :** Introduction au DevOps, 2026-2027

## 1. Objectif

Containeriser une application MERN (MongoDB, Express, React, Node.js) avec Docker, puis l'orchestrer avec Docker Compose.

## 2. Structure du projet

```
MyAppReact
├── Client/                 # Application React (Vite)
│   ├── Dockerfile
│   └── .dockerignore
├── Server/                 # API Express
│   ├── Dockerfile
│   └── .dockerignore
├── docker-compose.yml
└── README.md
```

Le dépôt a été cloné avec :

```bash
git clone https://github.com/wiem-kb/MyAppReact.git
```

## 3. Dockerfiles

### Serveur (`Server/Dockerfile`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 9000
CMD ["npm", "start"]
```

| Instruction | Rôle |
|---|---|
| `FROM node:lts-alpine` | Image de base légère avec Node.js |
| `WORKDIR /app` | Répertoire de travail dans le conteneur |
| `COPY package*.json ./` | Copie des fichiers de dépendances (profite du cache Docker) |
| `RUN npm install` | Installation des dépendances |
| `COPY . .` | Copie du reste du code |
| `EXPOSE 9000` | Port de l'API |
| `CMD ["npm", "start"]` | Démarrage du serveur |

La variable `MONGO_URI` est fournie au conteneur à l'exécution (voir sections 5 et 6).

### Client (`Client/Dockerfile`)

```dockerfile
FROM node:lts-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
RUN npm install -g serve
EXPOSE 3000
CMD ["serve", "-s", "dist", "-l", "3000"]
```

Le client est construit avec `npm run build`, puis servi par le serveur HTTP léger `serve`. Le projet utilise Vite, donc le dossier de sortie est `dist`.

### Fichiers `.dockerignore`

Dans `Client` et `Server` :

```
node_modules
dist
```

Ils évitent de copier `node_modules` dans l'image et accélèrent la construction.

## 4. Construction des images

```bash
docker build -t mern-server ./Server
docker build -t mern-client ./Client
```

## 5. Exécution manuelle (étapes 3 à 5 du TP)

```bash
docker network create mern-network
docker run -d --name mongodb --network mern-network mongo
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mydb mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

Le réseau `mern-network` permet aux conteneurs de communiquer entre eux par leur nom (par exemple `mongodb`). Vérification avec `docker ps` :

![docker ps](screenshots/docker-ps.png)

Avant de passer à Docker Compose, les conteneurs et le réseau manuels sont supprimés :

```bash
docker rm -f client server mongodb
docker network rm mern-network
```

## 6. Docker Compose

Fichier `docker-compose.yml` à la racine du projet :

```yaml
services:
  mongodb:
    image: mongo
    container_name: mongodb
    networks:
      - mern-network

  server:
    build: ./Server
    container_name: server
    ports:
      - "9000:9000"
    environment:
      - MONGO_URI=mongodb://mongodb:27017/mydb
    depends_on:
      - mongodb
    networks:
      - mern-network

  client:
    build: ./Client
    container_name: client
    ports:
      - "3000:3000"
    depends_on:
      - server
    networks:
      - mern-network

networks:
  mern-network:
    driver: bridge
```

Lancement de l'application :

```bash
docker compose up --build
```

Vérification de l'état des services avec `docker compose ps` :

![docker compose ps](screenshots/compose-ps.png)

Arrêt des services :

```bash
docker compose down
```

## 7. Test de l'application

Ouvrir `http://localhost:3000`. Le client React appelle l'API Express et affiche le message reçu, ce qui prouve que la chaîne navigateur → client → serveur fonctionne.

![Application](screenshots/app.png)

## 8. Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|---|---|---|
| Erreur 404 sur `localhost:3000` | Le client est une application Vite : le build est généré dans `dist`, mais `serve` pointait vers `build` | Remplacer par `CMD ["serve", "-s", "dist", "-l", "3000"]` |
| `.dockerignore` ignoré | Le fichier était nommé `as.dockerignore` | Renommage en `.dockerignore` |
| Le serveur ne trouvait pas MongoDB | `MONGO_URI` n'était pas fournie au conteneur | Ajout de `-e MONGO_URI=...` (commande `docker run`) et de `environment` (Compose) |
| Conflits de noms et de ports | Conteneurs manuels encore présents | `docker rm -f` puis `docker network rm` avant Compose |

## 9. Conclusion

Ce TP m'a permis de containeriser un client React et un serveur Express avec des Dockerfiles, de les relier à MongoDB par un réseau Docker, puis de tout orchestrer avec un seul fichier `docker-compose.yml` lancé par `docker compose up --build`.