# Poročilo: Docker postavitev aplikacije

## Namen

Cilj postavitve je omogočiti enostavno, ponovljivo in ločeno razvojno okolje za celotno aplikacijo, ki vključuje:
- **Frontend** (React),
- **Backend** (Node.js/Express),
- **MongoDB** podatkovno bazo,
- **Caddy** za usmerjanje HTTP zahtev (reverse proxy).

Z uporabo Dockerja zagotovimo:
- enotno okolje za vse razvijalce,
- enostaven zagon aplikacije z enim ukazom (`docker-compose up`),
- boljšo organizacijo in modularnost kode.

izdelati smo morali docker file za back in fornt end prav tako smo morali izdelati caddy file in pa docker-compose.yml ki vse poveže skupaj 

## 1. Docker file za backend
za back-end smo naredili docker file imenovan *DockerFile.backend*
```yaml
# definicija node verzije
FROM node:20-slim as dev

WORKDIR /app

# kopira back in pa front
COPY backend/package*.json ./

# Install dependencies
RUN npm install

# Copy the actual app source
COPY backend/ .

# Starting the app
CMD ["npm", "start"]
```
---
## 2. Docker file za frontend
za back end smo naredili docker file imenovan *DockerFile.frontend*

```yaml
#development
FROM node:22.14.0 as dev

#define working dir
WORKDIR /app

#copy package.json and package-lock.json to container
COPY frontend/package*.json ./
RUN npm install

#copy all filess
COPY frontend/ .

#start app in dev
CMD ["npm", "start"]
```
---

## 3. CaddyFile
```yaml
:80 {
  handle_path /api/* {
  reverse_proxy backend:5000
  }

  handle {
  reverse_proxy frontend:3000
  }
}
```

## 4. Docker-compose.yml
### 1. **MongoDB baza (`db`)**
```yaml
   mongo:
     image: mongo:latest
     container_name: kaputDB
     restart: unless-stopped
     ports:
       - "27017:27017"
     volumes:
       - kaput-data:/data/db

```
### 2. **Konfiguracija backend (` backend-dev`)**
```yaml
   backend-dev:
     container_name: backend
     build:
       context: .
       dockerfile: backend/DockerFile.backend
       target: dev
     volumes:
       - ./backend:/app
       - /app/node_modules
     ports:
       - "5000:5000"
     env_file:
       - ./backend/.env
     restart: unless-stopped
     depends_on:
       - mongo
```
### 3. **Konfiguracija backend (`frontend-dev`)**
```yaml
  frontend-dev:
    container_name: frontend
    build:
      context: .
      dockerfile: frontend/DockerFile.frontend
      target: dev
    volumes:
      - ./frontend:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    env_file:
      - ./frontend/.env
    restart: unless-stopped

    depends_on:
      - backend-dev
```
### 3. **Konfiguracija caddz (`caddy`)**
```yaml
   caddy:
     image: caddy:2-alpine
     container_name: caddy
     ports:
       - "80:80"
       - "443:443"
     volumes:
       - caddy_data:/data
       - caddy_config:/config
       - ./Caddyfile:/etc/caddy/Caddyfile
     depends_on:
       - frontend-dev
       - backend-dev
     restart: unless-stopped

   volumes:
     kaput-data:
     caddy_data:
     caddy_config:
```

## navodila in zagon 
- Namesti Docker in Docker Compose.

- V korenski mapi projekta zaženi ukaz:
```yaml
docker-compose up --build
```
Aplikacija bo dostopna na:

Frontend (prek Caddy): http://localhost

Backend API: http://localhost/api

MongoDB: mongodb://localhost:27017
