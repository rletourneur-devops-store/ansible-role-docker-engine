# Docker Engine - Runbook Opérations Jour 2+

Runbook détaillé pour les opérations avancées, la maintenance et les scénarios de récupération Docker.

---

## Table des Matières

1. [Troubleshooting Courant](#1-troubleshooting-courant)
2. [Gestion de l'Espace Disque](#2-gestion-de-lespace-disque)
3. [Problèmes Réseau](#3-problèmes-réseau)
4. [Mise à Jour Docker](#4-mise-à-jour-docker)
5. [Récupération de Conteneurs](#5-récupération-de-conteneurs)
6. [Scénarios Catastrophe](#6-scénarios-catastrophe)
7. [Configuration Avancée](#7-configuration-avancée)
8. [Monitoring et Alertes](#8-monitoring-et-alertes)

---

## 1. Troubleshooting Courant

### 1.1 Docker daemon ne démarre pas

**Symptômes** : `systemctl status docker` montre `failed`

**Diagnostic** :
```bash
# Voir les logs détaillés
sudo journalctl -xeu docker --no-pager | tail -100

# Vérifier la syntaxe du daemon.json
sudo cat /etc/docker/daemon.json | jq .
```

**Causes fréquentes et solutions** :

| Cause | Solution |
|-------|----------|
| `daemon.json` invalide | Corriger la syntaxe JSON, vérifier les virgules |
| Port 2375/2376 occupé | `sudo lsof -i :2375` puis arrêter le processus |
| Permissions `data-root` | `sudo chown -R root:root /path/to/data-root` |
| Espace disque insuffisant | Voir [Section 2](#2-gestion-de-lespace-disque) |

**Résolution** :
```bash
# Après correction, redémarrer
sudo systemctl restart docker
sudo systemctl status docker
```

### 1.2 Conteneur ne démarre pas

**Diagnostic** :
```bash
# Inspecter l'état du conteneur
docker inspect <container_id> --format '{{.State.Status}}'

# Voir les logs de démarrage
docker logs <container_id>

# Voir les événements
docker events --since 10m --filter container=<container_id>
```

**Causes fréquentes** :

| Message d'erreur | Solution |
|-----------------|----------|
| `OCI runtime create failed` | Vérifier les mounts et permissions |
| `port is already allocated` | Libérer le port : `docker stop $(docker ps -q --filter publish=<port>)` |
| `no space left on device` | Nettoyer l'espace disque (voir Section 2) |

### 1.3 Problème de permissions utilisateur

**Symptôme** : `permission denied while trying to connect to the Docker daemon socket`

**Solution** :
```bash
# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER

# Appliquer sans déconnexion
newgrp docker

# Vérifier
docker ps
```

---

## 2. Gestion de l'Espace Disque

### 2.1 Diagnostiquer l'utilisation disque

```bash
# Vue d'ensemble de l'utilisation Docker
docker system df

# Vue détaillée
docker system df -v

# Taille du data-root
sudo du -sh /var/lib/docker
# ou si configuré autrement
sudo du -sh $(docker info --format '{{.DockerRootDir}}')
```

### 2.2 Nettoyage manuel

```bash
# Nettoyage complet (images, conteneurs arrêtés, volumes orphelins, cache)
docker system prune -a --volumes

# Nettoyage sélectif - conteneurs arrêtés uniquement
docker container prune

# Nettoyage sélectif - images non utilisées
docker image prune -a

# Nettoyage sélectif - volumes orphelins
docker volume prune

# Nettoyage avec filtre temps (plus de 24h)
docker system prune -a --filter "until=24h"
```

### 2.3 Nettoyage d'urgence (disque plein)

**⚠️ ATTENTION : Procédure à risque**

```bash
# 1. Identifier les conteneurs les plus gourmands
docker ps --size --format "table {{.ID}}\t{{.Names}}\t{{.Size}}"

# 2. Identifier les images les plus volumineuses
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -h

# 3. Supprimer les logs volumineux (temporaire)
sudo sh -c 'truncate -s 0 /var/lib/docker/containers/*/*-json.log'

# 4. Supprimer les images <none>
docker rmi $(docker images -f "dangling=true" -q) 2>/dev/null

# 5. Si toujours critique, supprimer les conteneurs arrêtés
docker rm $(docker ps -a -q --filter status=exited)
```

### 2.4 Déplacer le data-root vers un nouveau disque

**Scénario** : Disque /var plein, nouveau disque disponible sur /mnt/data

```bash
# 1. Arrêter Docker
sudo systemctl stop docker

# 2. Copier les données existantes
sudo rsync -aP /var/lib/docker/ /mnt/data/docker/

# 3. Configurer le nouveau chemin
sudo cat << EOF > /etc/docker/daemon.json
{
  "data-root": "/mnt/data/docker",
  "live-restore": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
EOF

# 4. Démarrer Docker
sudo systemctl start docker

# 5. Vérifier
docker info --format '{{.DockerRootDir}}'

# 6. (Optionnel) Supprimer l'ancien répertoire après validation
# sudo rm -rf /var/lib/docker
```

---

## 3. Problèmes Réseau

### 3.1 Conteneur n'a pas accès à Internet

**Diagnostic** :
```bash
# Tester depuis un conteneur
docker run --rm alpine ping -c 3 8.8.8.8
docker run --rm alpine nslookup google.com

# Vérifier la configuration réseau Docker
docker network inspect bridge

# Vérifier iptables
sudo iptables -L -n | grep -i docker
```

**Solutions** :

```bash
# Redémarrer le daemon Docker
sudo systemctl restart docker

# Recréer le réseau bridge
docker network rm bridge
sudo systemctl restart docker

# Vérifier le forwarding IP
cat /proc/sys/net/ipv4/ip_forward
# Si 0, activer :
sudo sysctl -w net.ipv4.ip_forward=1
```

### 3.2 Conflits d'adresses IP

**Symptôme** : Conflit avec le réseau interne de l'entreprise

**Solution** : Configurer des plages d'adresses personnalisées

```bash
# Dans /etc/docker/daemon.json
{
  "bip": "172.26.0.1/16",
  "default-address-pools": [
    {"base": "172.27.0.0/16", "size": 24}
  ]
}
```

```bash
# Appliquer
sudo systemctl restart docker
```

### 3.3 Port non accessible depuis l'extérieur

**Diagnostic** :
```bash
# Vérifier le mapping de ports
docker port <container_id>

# Vérifier l'écoute
sudo ss -tlnp | grep <port>

# Vérifier iptables
sudo iptables -L -n -t nat | grep <port>
```

**Solution** : Vérifier que le conteneur écoute sur 0.0.0.0, pas 127.0.0.1

---

## 4. Mise à Jour Docker

### 4.1 Mise à jour standard

```bash
# 1. Sauvegarder la configuration
sudo cp /etc/docker/daemon.json /etc/docker/daemon.json.bak

# 2. Mettre à jour les paquets
sudo apt update
sudo apt upgrade docker-ce docker-ce-cli containerd.io

# 3. Vérifier la version
docker --version

# 4. Vérifier que les conteneurs live-restore sont toujours actifs
docker ps
```

### 4.2 Rollback après mise à jour problématique

```bash
# 1. Arrêter Docker
sudo systemctl stop docker

# 2. Lister les versions disponibles
apt-cache madison docker-ce

# 3. Installer une version spécifique
sudo apt install docker-ce=<VERSION> docker-ce-cli=<VERSION>

# 4. Bloquer les mises à jour auto
sudo apt-mark hold docker-ce docker-ce-cli

# 5. Redémarrer
sudo systemctl start docker
```

---

## 5. Récupération de Conteneurs

### 5.1 Conteneur crashé en boucle

```bash
# Voir l'historique des restarts
docker inspect <container_id> --format '{{.RestartCount}}'

# Voir le dernier exit code
docker inspect <container_id> --format '{{.State.ExitCode}}'

# Copier les fichiers de config depuis le conteneur arrêté
docker cp <container_id>:/path/to/config ./config_backup

# Démarrer en mode interactif pour debug
docker run -it --entrypoint /bin/sh <image>
```

### 5.2 Récupérer les données d'un conteneur supprimé

**Si le volume était nommé** :
```bash
# Lister tous les volumes
docker volume ls

# Inspecter le volume
docker volume inspect <volume_name>

# Monter le volume dans un nouveau conteneur
docker run -it -v <volume_name>:/data alpine ls -la /data
```

**Si le volume était anonyme** :
```bash
# Chercher dans le data-root
sudo find /var/lib/docker/volumes -name "*.db" 2>/dev/null
```

### 5.3 Exporter/Importer un conteneur

```bash
# Exporter un conteneur (état courant)
docker export <container_id> > container_backup.tar

# Importer comme nouvelle image
docker import container_backup.tar myapp:restored

# Créer une image depuis un conteneur (avec historique)
docker commit <container_id> myapp:snapshot
```

---

## 6. Scénarios Catastrophe

### 6.1 🔴 Corruption du système de fichiers Docker

**Symptômes** :
- `docker ps` renvoie des erreurs
- Conteneurs en état `Dead`
- Erreurs `layer does not exist`

**Procédure de récupération** :

```bash
# 1. SAUVEGARDER d'abord ce qui peut l'être
sudo mkdir -p /backup/docker-emergency
sudo cp -r /var/lib/docker/volumes /backup/docker-emergency/

# 2. Arrêter Docker
sudo systemctl stop docker

# 3. Nettoyer le cache buildkit
sudo rm -rf /var/lib/docker/buildkit

# 4. Réparer avec check
# (ATTENTION : peut supprimer des données corrompues)
sudo dockerd --config-file=/etc/docker/daemon.json &
docker system prune --all --volumes
sudo pkill dockerd

# 5. Si échec, réinitialisation complète
# ⚠️ PERTE DE TOUTES LES DONNÉES ⚠️
# sudo rm -rf /var/lib/docker
# sudo systemctl start docker
```

### 6.2 🔴 Serveur reboote avec conteneurs critiques

**Avec live-restore activé** (comportement par défaut du rôle) :

```bash
# Les conteneurs survivent au redémarrage du daemon
# Vérifier l'état après reboot
docker ps

# Si conteneurs non visibles mais processus actifs
sudo ps aux | grep containerd-shim

# Reconnecter Docker aux conteneurs orphelins
sudo systemctl restart containerd
sudo systemctl restart docker
```

**Sans live-restore** :

```bash
# Lister les conteneurs avec restart policy
docker ps -a --filter "status=exited" --format "{{.ID}} {{.Names}}"

# Démarrer manuellement
docker start <container_id>

# Ou créer avec restart policy pour le futur
docker update --restart unless-stopped <container_id>
```

### 6.3 🔴 Perte totale du serveur - Reconstruction

**Pré-requis** : Avoir sauvegardé régulièrement les volumes et configurations

```bash
# 1. Réinstaller Docker via Ansible
ansible-playbook -i inventaire playbook-docker.yml

# 2. Restaurer les volumes depuis la sauvegarde
sudo tar -xzf volumes_backup.tar.gz -C /var/lib/docker/

# 3. Récupérer les images depuis le registry
docker pull registry.example.com/app:latest

# 4. Recréer les conteneurs (si docker-compose)
docker-compose up -d

# 5. Vérifier
docker ps
```

### 6.4 🔴 Registry privé inaccessible

**Scénario** : Le registry Harbor est down et vous devez déployer

**Solutions temporaires** :

```bash
# 1. Utiliser le cache local (si l'image existe déjà)
docker images | grep myapp
docker run myapp:cached-version

# 2. Exporter/importer entre serveurs
# Sur serveur avec l'image :
docker save myapp:latest | gzip > myapp.tar.gz
scp myapp.tar.gz user@target:/tmp/

# Sur serveur cible :
gunzip -c /tmp/myapp.tar.gz | docker load

# 3. Configurer un registry miroir temporaire
docker run -d -p 5000:5000 registry:2
docker tag myapp:latest localhost:5000/myapp:latest
docker push localhost:5000/myapp:latest
```

---

## 7. Configuration Avancée

### 7.1 Limiter les ressources par défaut

```bash
# Dans /etc/docker/daemon.json
{
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  }
}
```

### 7.2 Activer le mode debug

```bash
# Dans /etc/docker/daemon.json
{
  "debug": true
}

# Appliquer
sudo systemctl restart docker

# Voir les logs debug
sudo journalctl -u docker -f
```

### 7.3 Configuration registre insecure

```bash
# Dans /etc/docker/daemon.json
{
  "insecure-registries": [
    "registry.internal.lan:5000",
    "10.0.0.50:5000"
  ]
}

# Appliquer
sudo systemctl restart docker
```

---

## 8. Monitoring et Alertes

### 8.1 Métriques Prometheus

```bash
# Vérifier l'endpoint métriques
curl -s http://localhost:9323/metrics | head -20

# Métriques importantes à monitorer :
# - engine_daemon_container_states_containers{state="running"}
# - engine_daemon_health_checks_failed_total
# - builder_builds_failed_total
```

### 8.2 Script de health check

```bash
#!/bin/bash
# /usr/local/bin/docker-healthcheck.sh

# Vérifier le service
if ! systemctl is-active --quiet docker; then
    echo "CRITICAL: Docker service is not running"
    exit 2
fi

# Vérifier l'espace disque
USAGE=$(docker system df --format '{{.Size}}' | head -1)
echo "Docker disk usage: $USAGE"

# Vérifier les conteneurs en erreur
UNHEALTHY=$(docker ps --filter health=unhealthy --format '{{.Names}}' | wc -l)
if [ "$UNHEALTHY" -gt 0 ]; then
    echo "WARNING: $UNHEALTHY unhealthy containers"
    docker ps --filter health=unhealthy --format '{{.Names}}'
    exit 1
fi

echo "OK: Docker is healthy"
exit 0
```

### 8.3 Alertes recommandées

| Métrique | Seuil Warning | Seuil Critical |
|----------|---------------|----------------|
| Espace disque Docker | 70% | 85% |
| Conteneurs unhealthy | > 0 | > 2 |
| Conteneurs restarting | > 3/5min | > 10/5min |
| Service Docker down | - | > 30s |

---

## Annexe : Commandes de Référence Rapide

```bash
# Statut global
docker system info
docker system df

# Nettoyage
docker system prune -af --volumes

# Debug
docker events --since 1h
journalctl -u docker --since "1 hour ago"

# Inspection
docker inspect <id> | jq .
docker stats --no-stream

# Réseau
docker network ls
docker network inspect bridge

# Volumes
docker volume ls
docker volume inspect <name>
```
