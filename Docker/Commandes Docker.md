# Commandes Essentielles Docker

## Table des matières
1. [Gestion des conteneurs](#1-gestion-des-conteneurs)
2. [Gestion des images](#2-gestion-des-images)
3. [Réseau & ports](#3-réseau--ports)
4. [Volumes & persistance](#4-volumes--persistance)
5. [Docker Compose](#5-docker-compose)
6. [Inspection & logs](#6-inspection--logs)
7. [Nettoyage](#7-nettoyage)

---

## 1. Gestion des conteneurs

Lister les conteneurs actifs :
```bash
docker ps
```

Lister tous les conteneurs (y compris arrêtés) :
```bash
docker ps -a
```

Démarrer un conteneur depuis une image :
```bash
docker run <image>
```
> Télécharge l'image si absente localement, puis crée et démarre un conteneur.

Démarrer en mode détaché (arrière-plan) :
```bash
docker run -d <image>
```

Démarrer avec un nom personnalisé :
```bash
docker run -d --name <nom> <image>
```

Démarrer un conteneur arrêté :
```bash
docker start <nom_ou_id>
```

Arrêter un conteneur :
```bash
docker stop <nom_ou_id>
```

Redémarrer un conteneur :
```bash
docker restart <nom_ou_id>
```

Supprimer un conteneur arrêté :
```bash
docker rm <nom_ou_id>
```

Accéder à un shell dans un conteneur actif :
```bash
docker exec -it <nom_ou_id> bash
```
> Remplacer `bash` par `sh` sur les images Alpine.

---

## 2. Gestion des images

Lister les images locales :
```bash
docker images
```

Télécharger une image depuis Docker Hub :
```bash
docker pull <image>:<tag>
```
> Exemple : `docker pull nginx:alpine` — sans tag, utilise `latest` par défaut.

Supprimer une image :
```bash
docker rmi <image>
```

Construire une image depuis un Dockerfile :
```bash
docker build -t <nom>:<tag> <chemin_dockerfile>
```
> Exemple : `docker build -t monapp:1.0 .`

---

## 3. Réseau & ports

Exposer un port hôte vers un port conteneur :
```bash
docker run -d -p <port_hote>:<port_conteneur> <image>
```
> Exemple : `-p 8080:80` → accessible sur `http://localhost:8080`

Lier uniquement sur localhost (non accessible depuis le réseau local) :
```bash
docker run -d -p 127.0.0.1:<port_hote>:<port_conteneur> <image>
```

Lister les réseaux Docker :
```bash
docker network ls
```

Créer un réseau personnalisé :
```bash
docker network create <nom_reseau>
```

Connecter un conteneur à un réseau :
```bash
docker network connect <nom_reseau> <nom_conteneur>
```

---

## 4. Volumes & persistance

Monter un dossier de l'hôte dans le conteneur :
```bash
docker run -v <chemin_hote>:<chemin_conteneur> <image>
```
> Exemple : `-v /home/alex/site:/usr/share/nginx/html`  
> Les données survivent à la suppression du conteneur.

Créer un volume Docker nommé :
```bash
docker volume create <nom_volume>
```

Utiliser un volume nommé :
```bash
docker run -v <nom_volume>:<chemin_conteneur> <image>
```

Lister les volumes :
```bash
docker volume ls
```

Supprimer un volume :
```bash
docker volume rm <nom_volume>
```

---

## 5. Docker Compose

Démarrer la stack en arrière-plan :
```bash
docker compose up -d
```

Arrêter la stack sans supprimer les conteneurs :
```bash
docker compose stop
```

Arrêter et supprimer les conteneurs :
```bash
docker compose down
```

Arrêter, supprimer conteneurs et volumes :
```bash
docker compose down -v
```
> ⚠️ Les données en volume sont supprimées définitivement.

Reconstruire les images avant démarrage :
```bash
docker compose up -d --build
```

Afficher les logs de la stack :
```bash
docker compose logs -f
```

---

## 6. Inspection & logs

Afficher les logs d'un conteneur :
```bash
docker logs <nom_ou_id>
```

Suivre les logs en temps réel :
```bash
docker logs -f <nom_ou_id>
```

Inspecter la configuration d'un conteneur :
```bash
docker inspect <nom_ou_id>
```

Afficher l'utilisation des ressources en temps réel :
```bash
docker stats
```

---

## 7. Nettoyage

Supprimer tous les conteneurs arrêtés :
```bash
docker container prune
```

Supprimer toutes les images non utilisées :
```bash
docker image prune -a
```

Supprimer tous les volumes non utilisés :
```bash
docker volume prune
```

Tout nettoyer d'un coup (conteneurs, images, réseaux, cache) :
```bash
docker system prune -a
```
> ⚠️ Supprime tout ce qui n'est pas actuellement utilisé.

Arrêter et supprimer tous les conteneurs en une commande :
```bash
docker stop $(docker ps -q) && docker rm $(docker ps -aq)
```