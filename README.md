# Case Study Lombok

> Fork de [Elec_business_spring](https://github.com/Nahima697/Elec_business_spring)
> utilisé pour valider la prise en charge de Lombok par HexaGlue.

---

# Electricity Business – Déploiement via Docker Compose

## 1. Présentation

Electricity Business est une application pour réserver des bornes de recharge électrique.
Elle est composée de deux parties :

- **Backend (`elecbusiness_spring`)** : Spring Boot 3.x, Java 21, PostgreSQL, Redis, JWT, OTP, Testcontainers, H2  
- **Frontend (`elecbusiness_front`)** : Angular, Ionic, Tailwind CSS  

Tout est dockerisé pour simplifier le déploiement local et serveur.

---

## 2. Prérequis

- Docker  
- Docker Compose  
- Accès terminal / SSH avec droits sudo  

---

## 3. Cloner les repositories

```bash
git clone https://github.com/Nahima697/elecbusiness_spring.git
git clone https://github.com/Nahima697/elecbusiness_front.git


```
---
## 4.Créer un fichier .env à la racine avec le contenu suivant :

```bash
# PostgreSQL
POSTGRES_DB=eb_db
POSTGRES_USER=eb_user
POSTGRES_PASSWORD=eb_password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Mail
MAIL_HOST=mail
MAIL_PORT=1025

# JWT
JWT_SECRET=<votre_secret_jwt>

```
---
## 5. Schéma d’architecture des conteneurs

           +---------------------+
           |      Frontend       |
           |   Angular/Ionic     |
           |      Port 8100      |
           +----------+----------+
                      |
                      v
           +---------------------+
           |       Backend       |
           |     Spring Boot     |
           |      Port 8080      |
           |   JWT + OTP Auth    |
           +----------+----------+
                      |
          -------------------------
          |                       |
          v                       v
    +------------+           +------------+
    | PostgreSQL |           |    Redis   |
    |  Port 5432 |           |  Port 6379 |
    +------------+           +------------+
                      ^
                      |
                 +-----------+
                 |  MailHog  |
                 |   SMTP    |
                 |  Port 1025|
                 +-----------+

---
## 6. Lancer l’application

Depuis le dossier contenant le `docker-compose.yaml` :

```bash
docker-compose up --build -d

```
Vérifier les conteneurs :

```bash
docker-compose ps

```
---
## 7. Accès aux services

- Frontend : [http://<serveur>:8100](http://<serveur>:8100)  
- Backend : [http://<serveur>:8080](http://<serveur>:8080)  
- MailHog : [http://<serveur>:8025](http://<serveur>:8025)  
- PostgreSQL : via port 5432 (si nécessaire)  
- Redis : via port 6379 (si nécessaire)  

---

## 8. Arrêter l’application

```bash
docker-compose down
```
---

## 9. DockerFile
### 9.1 Spring Boot

# Étape 1 : build avec Maven 
```bash

FROM maven:3.9.1-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests
```

# Étape 2 : image finale pour exécution
```bash
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
```

### 9.2 Frontend (Angular/Ionic)
# Étape 1 : build avec Node
```bash
FROM node:22 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build --prod
```

# Étape 2 : servir avec Nginx
```bash
FROM nginx:alpine
COPY --from=build /app/dist/elecbusiness_front /usr/share/nginx/html
EXPOSE 8100
CMD ["nginx", "-g", "daemon off;"]
```
---
## 10. Déploiement sur serveur
### 10.1 Préparation du serveur

# Mettre à jour le serveur
```bash
sudo apt update && sudo apt upgrade -y
```

# Installer Git, Docker et Docker Compose
```bash
sudo apt install -y git docker.io docker-compose
```

# Activer Docker
```bash
sudo systemctl enable docker
sudo usermod -aG docker $USER
```

## 10.2 Déploiement initial
```bash
# Cloner les projets
cd /opt
git clone https://github.com/Nahima697/elecbusiness_spring.git
git clone https://github.com/Nahima697/elecbusiness_front.git

# Placer le fichier .env correctement
cd elecbusiness_spring # ou frontend si besoin

# Builder et lancer les conteneurs
docker-compose build
docker-compose up -d

# Vérifier les conteneurs
docker-compose ps
```

## 10.3 Mise à jour et redéploiement
```bash
# Pull des derniers changements
cd /opt/elecbusiness_spring
git pull
./mvnw clean verify
cd ../elecbusiness_front
git pull
npm install
npm run test
npm run build
cd ..

# Rebuild et relancer
docker-compose build
docker-compose up -d
```
## 11. Stratégie de tests et déploiement
### 11.1 Séparation

Tests → phase de build dans Dockerfile

Déploiement → démarre uniquement du code/images déjà testés

docker-compose up ne lance pas les tests
