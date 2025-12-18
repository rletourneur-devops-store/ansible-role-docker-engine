# Docker Engine - Guide Jour 1

Guide de démarrage rapide pour valider et utiliser Docker après déploiement via le rôle `devops_store.docker_engine`.

---

## 1. Vérifications Post-Déploiement

### 1.1 Vérifier l'installation Docker

```bash
# Version Docker installée
docker --version

# Informations complètes du daemon
docker info
```

**Résultat attendu** : Docker Engine installé avec les informations sur le driver de stockage, la configuration réseau et le nombre de conteneurs.

### 1.2 Vérifier le service Docker

```bash
# État du service
sudo systemctl status docker

# Vérifier que Docker démarre au boot
sudo systemctl is-enabled docker
```

### 1.3 Vérifier la configuration daemon.json

```bash
# Afficher la configuration
cat /etc/docker/daemon.json | jq .

# Vérifications clés à confirmer :
# - "live-restore": true
# - "log-driver": "json-file" avec rotation
# - "data-root" si configuré
```

### 1.4 Vérifier les utilisateurs Docker

```bash
# Liste des utilisateurs du groupe docker
getent group docker

# Tester sans sudo (en tant qu'utilisateur configuré)
docker ps
```

---

## 2. Commandes de Base

### 2.1 Gestion des Conteneurs

```bash
# Lister tous les conteneurs (actifs et arrêtés)
docker ps -a

# Démarrer un conteneur de test
docker run --rm hello-world

# Logs d'un conteneur
docker logs <container_id>

# Suivre les logs en temps réel
docker logs -f <container_id>

# Exécuter une commande dans un conteneur
docker exec -it <container_id> /bin/bash
```

### 2.2 Gestion des Images

```bash
# Lister les images locales
docker images

# Télécharger une image
docker pull nginx:latest

# Supprimer une image
docker rmi <image_id>
```

### 2.3 Gestion des Volumes

```bash
# Lister les volumes
docker volume ls

# Inspecter un volume
docker volume inspect <volume_name>
```

### 2.4 Gestion des Réseaux

```bash
# Lister les réseaux
docker network ls

# Inspecter un réseau
docker network inspect bridge
```

---

## 3. Vérification du Proxy (si configuré)

```bash
# Vérifier la configuration systemd
cat /etc/systemd/system/docker.service.d/http-proxy.conf

# Tester l'accès à Docker Hub via proxy
docker pull alpine:latest
```

---

## 4. Vérification des Métriques Prometheus (si activées)

```bash
# Tester l'endpoint métriques
curl http://localhost:9323/metrics

# Métriques clés à vérifier :
# - engine_daemon_container_states_containers
# - engine_daemon_image_actions_seconds
```

---

## 5. Vérification du Cron de Maintenance

```bash
# Vérifier la tâche cron de nettoyage
sudo crontab -l | grep docker

# Résultat attendu : docker system prune à 3h00
```

---

## 6. Checklist de Santé Rapide

| Vérification | Commande | État Attendu |
|--------------|----------|--------------|
| Service Docker actif | `systemctl is-active docker` | `active` |
| Conteneur test | `docker run --rm hello-world` | `Hello from Docker!` |
| Daemon config | `docker info --format '{{.LiveRestoreEnabled}}'` | `true` |
| Data root | `docker info --format '{{.DockerRootDir}}'` | Chemin configuré |
| Utilisateurs | `groups $USER \| grep docker` | `docker` présent |

---

## 7. Accès et Logs

### Logs Docker daemon

```bash
# Logs du service Docker
sudo journalctl -u docker -f

# Logs d'un conteneur spécifique
docker logs --tail 100 <container_id>
```

### Emplacement des données

| Élément | Chemin par défaut |
|---------|-------------------|
| Daemon config | `/etc/docker/daemon.json` |
| Data root | `/var/lib/docker` (ou `docker_data_root`) |
| Logs conteneurs | `<data-root>/containers/<id>/` |
| Images | `<data-root>/overlay2/` |

---

## 8. Premiers Pas - Test Complet

```bash
# 1. Télécharger une image
docker pull nginx:alpine

# 2. Démarrer un conteneur
docker run -d --name test-nginx -p 8080:80 nginx:alpine

# 3. Vérifier le conteneur
docker ps

# 4. Tester l'accès
curl http://localhost:8080

# 5. Arrêter et supprimer
docker stop test-nginx
docker rm test-nginx
```

**Résultat attendu** : Page HTML Nginx par défaut affichée.
