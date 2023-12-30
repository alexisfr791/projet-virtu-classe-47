# Guide de Déploiement de l'Application

Ce guide explique comment déployer l'application en utilisant des commandes Docker directes. Assurez-vous d'avoir Docker installé sur votre machine avant de commencer.

## Étapes de Déploiement avec les commandes Docker (version Manuelle)

### 1. Cloner le Projet

```bash
git clone https://github.com/pascalito007/esiea-ressources.git
cd esiea-ressources
```
### 2. Construction du réseau 
```bash
docker network create --driver=bridge cats-or-dogs-network --subnet=172.23.0.0/24 || true
```
### 3. Construction de la database (docker Volume)
```bash
docker volume create db-data
```
### 4. Construction des images Docker 
```bash
docker build -t alexisfr-virtu-vote:cmd ./vote/
docker build -t alexisfr-virtu-seed-data:cmd ./seed-data/
docker build -t alexisfr-virtu-result:cmd ./result/
docker build -t alexisfr-virtu-worker:cmd ./worker/
```
### 5. Lancement des conteneurs 

Lancer le db 
```bash
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.3 \
        --name db \
        -v db-data:/var/lib/postgresql/data \
        -v "$(pwd)/healthchecks:/healthchecks" \
        --health-cmd="/healthchecks/postgres.sh" \
        --health-interval="5s" \
        -e POSTGRES_PASSWORD=postgres \
        -p 5432:5432 \
        -d \
        postgres:15-alpine
```
Lancer le redis 
```bash 
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.4 \
        --name redis \
        -v "$(pwd)/healthchecks:/healthchecks" \
        --health-cmd="/healthchecks/redis.sh" \
        --health-interval="5s" \
        -p 6379:6379 \
        -d \
        redis
```
Lancer le vote 
```bash
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.2 \
        --name vote \
        -v "$(pwd)/vote:/usr/local/app" \
        -p 5002:80 \
        -d \
        alexisfr-virtu-vote:cmd
```
Lancer le result 
```bash
docker rm result || true
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.5 \
        --name result \
        --health-cmd="curl -f http://localhost/ || exit 1" \
        --health-interval="30s" \
        -p 5001:80 \
        -p 127.0.0.1:9229:9229 \
        -d \
        alexisfr-virtu-result:cmd
```
Lancer le worker
```bash
docker rm worker || true
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.6 \
        --name worker \
        -d \
        alexisfr-virtu-worker:cmd
```
Lancer le seed-data
```bash
docker rm seed-data || true
sleep 2
docker run \
        --network=cats-or-dogs-network \
        --ip 172.23.0.7 \
        --name seed-data \
        -d \
        alexisfr-virtu-seed-data:cmd
```
Une fois les conteneurs lancer il ne reste plus qu'à aller sur le http://172.16.50.129:5001 pour la page résultat et le http://172.16.50.129:5002 pour la page vote

### 6. Arrêter les conteneurs 
```bash
docker stop db redis result vote worker
```






