# Chapitre 226 - Docker et orchestration de conteneurs

> "Les conteneurs sont comme des programmes exécutables pour votre infrastructure. Ils encapsulent tout ce dont une application a besoin pour fonctionner." - Solomon Hykes, co-créateur de Docker

## Introduction : La révolution des conteneurs

Docker a transformé l'industrie du logiciel en permettant de packager, distribuer et exécuter des applications de manière isolée et reproductible. Au-delà de Docker lui-même, l'orchestration de conteneurs avec Kubernetes et Docker Swarm permet de gérer des applications distribuées à l'échelle du cloud.

Dans ce chapitre, nous explorerons les conteneurs Docker, leur orchestration, et les patterns de déploiement modernes.

## Section 1 : Concepts fondamentaux des conteneurs

### 1.1 Qu'est-ce qu'un conteneur ?

**Conteneurs vs Machines virtuelles :**
```bash
echo "=== COMPARAISON CONTENEURS VS VM ==="
echo ""
echo "Machines Virtuelles :"
echo "  ✓ Isolation complète du matériel"
echo "  ✓ Hyperviseur dédié"
echo "  ✓ Ressources dédiées"
echo "  ✗ Lourd et lent au démarrage"
echo "  ✗ Consommation importante de ressources"
echo ""
echo "Conteneurs :"
echo "  ✓ Isolation au niveau OS"
echo "  ✓ Partage du kernel hôte"
echo "  ✓ Démarrage en secondes"
echo "  ✓ Faible overhead"
echo "  ✗ Partage du kernel (sécurité)"
echo "  ✗ Isolation moins stricte"
echo ""
echo "Points communs :"
echo "  • Encapsulation d'applications"
echo "  • Isolation des processus"
echo "  • Gestion des ressources"
echo "  • Portabilité"
```

**Avantages des conteneurs :**
- **Portabilité** : Fonctionne partout où Docker est installé
- **Isolation** : Applications indépendantes les unes des autres
- **Efficacité** : Partage des ressources système
- **Reproductibilité** : Environnements identiques en dev/prod
- **Évolutivité** : Démarrage/arrêt rapide des instances

### 1.2 Installation et configuration de Docker

```bash
# Installation de Docker sur Ubuntu/Debian
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# Ajouter la clé GPG officielle de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajouter le dépôt Docker
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installer Docker Engine
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io

# Démarrer Docker
sudo systemctl start docker
sudo systemctl enable docker

# Ajouter l'utilisateur au groupe docker (optionnel)
sudo usermod -aG docker $USER

# Installation sur CentOS/RHEL
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker

# Installation sur macOS
brew install --cask docker

# Installation sur Windows
# Télécharger et installer Docker Desktop depuis https://www.docker.com/products/docker-desktop

# Vérification de l'installation
docker --version
docker run hello-world

# Configuration du daemon Docker
sudo mkdir -p /etc/docker
cat > /etc/docker/daemon.json << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2",
  "insecure-registries": ["myregistry.local:5000"],
  "registry-mirrors": ["https://mirror.gcr.io"]
}
EOF

sudo systemctl restart docker
```

## Section 2 : Images et conteneurs Docker

### 2.1 Travail avec les images

```bash
# Recherche d'images
docker search nginx
docker search --filter "is-official=true" ubuntu

# Téléchargement d'images
docker pull nginx:alpine
docker pull ubuntu:20.04
docker pull python:3.9-slim

# Lister les images locales
docker images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Informations détaillées sur une image
docker inspect nginx:alpine

# Historique d'une image
docker history nginx:alpine

# Supprimer des images
docker rmi nginx:alpine
docker image prune -a  # Supprimer les images non utilisées

# Tagger des images
docker tag nginx:alpine myregistry.com/nginx:v1.0

# Pousser une image vers un registre
docker push myregistry.com/nginx:v1.0

# Sauvegarder une image
docker save nginx:alpine > nginx.tar

# Charger une image sauvegardée
docker load < nginx.tar
```

### 2.2 Gestion des conteneurs

```bash
# Exécuter un conteneur simple
docker run hello-world

# Exécuter un conteneur interactif
docker run -it ubuntu:20.04 /bin/bash

# Exécuter un conteneur en arrière-plan
docker run -d --name web nginx:alpine

# Lister les conteneurs
docker ps                    # Conteneurs en cours
docker ps -a                 # Tous les conteneurs
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Informations sur un conteneur
docker inspect web

# Logs d'un conteneur
docker logs web
docker logs -f web  # Suivre les logs en temps réel

# Exécuter une commande dans un conteneur
docker exec -it web /bin/sh

# Copier des fichiers vers/depuis un conteneur
docker cp index.html web:/usr/share/nginx/html/
docker cp web:/var/log/nginx/access.log .

# Arrêter un conteneur
docker stop web

# Redémarrer un conteneur
docker restart web

# Supprimer un conteneur
docker rm web
docker rm $(docker ps -aq)  # Supprimer tous les conteneurs arrêtés

# Nettoyer les conteneurs arrêtés
docker container prune
```

### 2.3 Réseaux et volumes

```bash
# Lister les réseaux
docker network ls

# Créer un réseau personnalisé
docker network create --driver bridge myapp_network

# Inspecter un réseau
docker network inspect bridge

# Connecter un conteneur à un réseau
docker network connect myapp_network web

# Déconnecter un conteneur d'un réseau
docker network disconnect bridge web

# Créer un volume
docker volume create myapp_data

# Lister les volumes
docker volume ls

# Inspecter un volume
docker volume inspect myapp_data

# Utiliser un volume dans un conteneur
docker run -d --name db \
  -v myapp_data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=password \
  mysql:8.0

# Monter un répertoire hôte
docker run -d --name web \
  -v /host/path:/container/path \
  nginx:alpine

# Volumes nommés vs bind mounts
echo "=== TYPES DE MONTAGES ==="
echo "Bind Mount: -v /host/path:/container/path"
echo "  ✓ Accès direct aux fichiers hôte"
echo "  ✓ Permissions identiques à l'hôte"
echo "  ✗ Dépendant de la structure hôte"
echo ""
echo "Volume Nommé: -v volume_name:/container/path"
echo "  ✓ Géré par Docker"
echo "  ✓ Portable entre hôtes"
echo "  ✓ Permissions gérées par Docker"
echo "  ✗ Moins performant pour gros volumes"
```

## Section 3 : Création d'images Docker

### 3.1 Dockerfile de base

```dockerfile
# Dockerfile pour une application Node.js
FROM node:16-alpine

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installer les dépendances
RUN npm ci --only=production

# Copier le code source
COPY . .

# Créer un utilisateur non-root
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Changer les permissions
RUN chown -R nextjs:nodejs /app
USER nextjs

# Exposer le port
EXPOSE 3000

# Commande de démarrage
CMD ["npm", "start"]
```

```dockerfile
# Dockerfile multi-stage pour une application Go
# Stage 1: Build
FROM golang:1.19-alpine AS builder

WORKDIR /app

# Copier les fichiers go.mod et go.sum
COPY go.mod go.sum ./
RUN go mod download

# Copier le code source
COPY . .

# Compiler l'application
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Stage 2: Runtime
FROM alpine:latest

# Installer les certificats CA
RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copier l'exécutable depuis le stage builder
COPY --from=builder /app/main .

# Exposer le port
EXPOSE 8080

# Commande de démarrage
CMD ["./main"]
```

```dockerfile
# Dockerfile pour une application Python avec optimisations
FROM python:3.9-slim

# Variables d'environnement pour Python
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Créer un utilisateur non-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Installer les dépendances système
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copier les requirements et installer les dépendances Python
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code source
COPY --chown=appuser:appuser . .

# Changer pour l'utilisateur non-root
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

CMD ["python", "app.py"]
```

### 3.2 Construction et optimisation d'images

```bash
# Construction d'une image de base
docker build -t myapp:v1.0 .

# Construction avec un contexte spécifique
docker build -t myapp:v1.0 -f Dockerfile.prod .

# Construction avec des arguments
docker build --build-arg VERSION=1.2.0 -t myapp:v1.2.0 .

# Utiliser un cache de build
docker build --cache-from myapp:v1.0 -t myapp:v1.1 .

# Optimisations de build
echo "=== TECHNIQUES D'OPTIMISATION ==="
echo ""
echo "1. Utiliser des images de base légères"
echo "   FROM alpine:latest"
echo ""
echo "2. Multi-stage builds"
echo "   FROM golang:alpine AS builder"
echo "   FROM alpine:latest"
echo "   COPY --from=builder /app/main ."
echo ""
echo "3. .dockerignore"
echo "   node_modules/"
echo "   .git/"
echo "   *.log"
echo ""
echo "4. Ordonner les instructions"
echo "   COPY package.json ."
echo "   RUN npm install"
echo "   COPY . ."
echo ""
echo "5. Utiliser des utilisateurs non-root"
echo "   RUN useradd appuser"
echo "   USER appuser"
```

```dockerfile
# Exemple de .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.coverage
.cache
.DS_Store
*.log
dist
build
```

```bash
# Analyser la taille des images
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Analyser les layers d'une image
docker history myapp:v1.0

# Comparer des images
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  wagoodman/dive myapp:v1.0

# Optimisations avancées
echo "=== OUTILS D'OPTIMISATION ==="
echo ""
echo "dive: Analyse les layers"
echo "docker-slim: Réduit la taille des images"
echo "buildkit: Builder plus rapide et efficace"
echo "buildx: Build multi-plateforme"
```

### 3.3 Registres et distribution

```bash
# Connexion à un registre
docker login myregistry.com

# Pousser une image
docker push myregistry.com/myapp:v1.0

# Tirer une image
docker pull myregistry.com/myapp:v1.0

# Lister les tags d'une image
curl -s "https://registry.hub.docker.com/v2/library/nginx/tags/list" | jq '.tags[]' | head -10

# Configurer un registre local
# Installation de registry
docker run -d -p 5000:5000 --name registry registry:2

# Pousser vers le registre local
docker tag myapp:v1.0 localhost:5000/myapp:v1.0
docker push localhost:5000/myapp:v1.0

# Configurer la sécurité pour un registre local
cat > /etc/docker/daemon.json << 'EOF'
{
  "insecure-registries": ["myregistry.local:5000"]
}
EOF

sudo systemctl restart docker

# Utiliser des secrets pour l'authentification
echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

# Webhooks et notifications
# Intégration avec Docker Hub webhooks
curl -X POST \
  -H "Content-Type: application/json" \
  -d '{"push_data":{"tag":"v1.0"},"repository":{"name":"myapp"}}' \
  $WEBHOOK_URL
```

## Section 4 : Docker Compose

### 4.1 Fichiers de composition

```yaml
# docker-compose.yml - Application multi-conteneurs
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://app:password@db:5432/myapp
    depends_on:
      - db
    volumes:
      - ./web/uploads:/app/uploads
    networks:
      - app_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=password
    volumes:
      - db_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app_network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d myapp"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:6-alpine
    volumes:
      - redis_data:/data
    networks:
      - app_network
    restart: unless-stopped
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - web_static:/var/www/html
    depends_on:
      - web
    networks:
      - app_network
    restart: unless-stopped

volumes:
  db_data:
    driver: local
  redis_data:
    driver: local
  web_static:
    driver: local

networks:
  app_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

```yaml
# docker-compose.override.yml - Configuration de développement
version: '3.8'

services:
  web:
    build:
      context: ./web
      dockerfile: Dockerfile.dev
    volumes:
      - ./web:/app
      - /app/node_modules
    environment:
      - NODE_ENV=development
      - DEBUG=*
    command: npm run dev

  db:
    ports:
      - "5432:5432"

  redis:
    ports:
      - "6379:6379"
```

### 4.2 Gestion des environnements

```bash
# Démarrer les services
docker-compose up -d

# Démarrer un service spécifique
docker-compose up -d web

# Suivre les logs
docker-compose logs -f
docker-compose logs -f web

# Exécuter des commandes
docker-compose exec web /bin/bash
docker-compose exec db psql -U app -d myapp

# Arrêter les services
docker-compose down

# Arrêter et supprimer les volumes
docker-compose down -v

# Reconstruire les images
docker-compose build
docker-compose build --no-cache

# Mettre à jour les services
docker-compose pull
docker-compose up -d

# Scaling des services
docker-compose up -d --scale web=3

# Variables d'environnement
cat > .env << 'EOF'
COMPOSE_PROJECT_NAME=myapp
POSTGRES_PASSWORD=secretpassword
REDIS_PASSWORD=redispass
EOF

# Utilisation de profiles
# docker-compose.yml avec profiles
services:
  db:
    # ...
  
  redis:
    # ...
    profiles:
      - cache
  
  monitoring:
    image: prometheus
    profiles:
      - monitoring

# Démarrer seulement les services de cache
docker-compose --profile cache up -d
```

## Section 5 : Orchestration avec Docker Swarm

### 5.1 Initialisation d'un Swarm

```bash
# Initialiser un Swarm
docker swarm init

# Obtenir le token de jointure pour les workers
docker swarm join-token worker

# Obtenir le token de jointure pour les managers
docker swarm join-token manager

# Joindre un nœud au Swarm
docker swarm join --token SWMTKN-... 192.168.1.10:2377

# Lister les nœuds
docker node ls

# Inspecter un nœud
docker node inspect node-01

# Promouvoir un nœud en manager
docker node promote node-02

# Rétrograder un manager
docker node demote node-01

# Supprimer un nœud
docker node rm node-03

# État du Swarm
docker info
```

### 5.2 Services et stacks

```yaml
# docker-compose.swarm.yml - Déploiement en Swarm
version: '3.8'

services:
  web:
    image: myapp/web:v1.0
    ports:
      - "80:3000"
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        max_failure_ratio: 0.3
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      placement:
        constraints:
          - node.role == worker
          - node.labels.region == us-east
    networks:
      - app_network
    secrets:
      - db_password

  db:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=app
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.storage == fast
    volumes:
      - db_data:/var/lib/postgresql/data
    networks:
      - app_network
    secrets:
      - db_password

  redis:
    image: redis:6-alpine
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.memory == high
    volumes:
      - redis_data:/data
    networks:
      - app_network
    command: redis-server --appendonly yes --requirepass $REDIS_PASSWORD

volumes:
  db_data:
    driver: local
  redis_data:
    driver: local

networks:
  app_network:
    driver: overlay
    attachable: true

secrets:
  db_password:
    external: true
```

```bash
# Déployer une stack
docker stack deploy -c docker-compose.swarm.yml myapp

# Lister les stacks
docker stack ls

# Lister les services d'une stack
docker stack services myapp

# Lister les tâches d'un service
docker service ps myapp_web

# Inspecter un service
docker service inspect myapp_web

# Logs d'un service
docker service logs myapp_web

# Mettre à jour un service
docker service update --image myapp/web:v1.1 myapp_web

# Scaling d'un service
docker service scale myapp_web=5

# Rollback d'un service
docker service update --rollback myapp_web

# Supprimer une stack
docker stack rm myapp

# Gestion des secrets
echo "mypassword" | docker secret create db_password -

# Lister les secrets
docker secret ls

# Inspecter un secret
docker secret inspect db_password
```

### 5.3 Configuration et secrets

```bash
# Créer une configuration
echo "max_connections = 100" | docker config create postgres_config -

# Créer un secret
echo "supersecretpassword" | docker secret create db_password -

# Utiliser dans un service
docker service create \
  --name db \
  --config postgres_config \
  --secret db_password \
  postgres:13-alpine

# Lister les configurations
docker config ls

# Lister les secrets
docker secret ls

# Mettre à jour un secret
echo "newpassword" | docker secret create db_password_v2 -
docker service update --secret-rm db_password --secret-add db_password_v2 db
docker secret rm db_password
```

## Section 6 : Kubernetes - Orchestration avancée

### 6.1 Concepts fondamentaux

```bash
echo "=== ARCHITECTURE KUBERNETES ==="
echo ""
echo "Control Plane (Maître):"
echo "  • API Server: Interface REST"
echo "  • etcd: Stockage clé-valeur"
echo "  • Controller Manager: Contrôleurs"
echo "  • Scheduler: Planification des pods"
echo ""
echo "Nodes (Ouvriers):"
echo "  • kubelet: Agent sur chaque nœud"
echo "  • kube-proxy: Gestion réseau"
echo "  • Container Runtime: Docker, containerd"
echo ""
echo "Objets principaux:"
echo "  • Pod: Unité de déploiement minimale"
echo "  • Deployment: Gestion des déploiements"
echo "  • Service: Exposition réseau"
echo "  • ConfigMap/Secret: Configuration"
echo "  • Ingress: Routage HTTP"
```

### 6.2 Installation et configuration

```bash
# Installation de kubectl
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Windows
choco install kubernetes-cli

# Vérification
kubectl version --client

# Installation de Minikube (pour le développement)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Démarrer Minikube
minikube start --driver=docker

# Installation de Kind (Kubernetes in Docker)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.14.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/

# Créer un cluster Kind
cat > kind-config.yaml << 'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
EOF

kind create cluster --config kind-config.yaml

# Configuration kubectl
kubectl cluster-info
kubectl get nodes
kubectl config current-context
```

### 6.3 Déploiements et services

```yaml
# deployment.yaml - Déploiement d'une application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp/web:v1.0
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: database-url
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 250m
            memory: 256Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        volumeMounts:
        - name: uploads
          mountPath: /app/uploads
      volumes:
      - name: uploads
        persistentVolumeClaim:
          claimName: uploads-pvc
```

```yaml
# service.yaml - Service pour exposer l'application
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
  type: ClusterIP

---
# Service LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
  type: LoadBalancer

---
# Ingress pour le routage HTTP
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
  tls:
  - hosts:
    - myapp.example.com
    secretName: tls-secret
```

```yaml
# configmap.yaml - Configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.json: |
    {
      "database": {
        "host": "postgres-service",
        "port": 5432,
        "name": "myapp"
      },
      "redis": {
        "host": "redis-service",
        "port": 6379
      },
      "log_level": "info"
    }

---
# secret.yaml - Secrets
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Valeurs encodées en base64
  database-url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAZGJfaG9zdDo1NDMyL215YXBw
  db-password: c3VwZXJzZWNyZXRwYXNzd29yZA==

---
# pvc.yaml - Persistent Volume Claim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```bash
# Appliquer les manifests
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# Lister les ressources
kubectl get pods
kubectl get services
kubectl get deployments
kubectl get configmaps
kubectl get secrets

# Détailler une ressource
kubectl describe pod web-app-12345-abcde

# Logs d'un pod
kubectl logs web-app-12345-abcde
kubectl logs -f web-app-12345-abcde  # Suivre les logs

# Exécuter une commande dans un pod
kubectl exec -it web-app-12345-abcde -- /bin/bash

# Port forwarding
kubectl port-forward svc/web-app-service 8080:80

# Scaling d'un deployment
kubectl scale deployment web-app --replicas=5

# Rolling update
kubectl set image deployment/web-app web=myapp/web:v1.1

# Rollback
kubectl rollout undo deployment/web-app

# Status du rollout
kubectl rollout status deployment/web-app

# Supprimer des ressources
kubectl delete -f deployment.yaml
kubectl delete pod,pvc,configmap,secret --all
```

### 6.4 Gestion des ressources et monitoring

```yaml
# hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80

---
# vpa.yaml - Vertical Pod Autoscaler (nécessite installation)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  updatePolicy:
    updateMode: "Auto"

---
# networkpolicy.yaml - Politiques réseau
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-policy
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api-gateway
    ports:
    - protocol: TCP
      port: 3000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

```bash
# Monitoring avec kubectl
kubectl top pods
kubectl top nodes

# Événements
kubectl get events --sort-by=.metadata.creationTimestamp

# Debugging des pods
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous  # Logs du conteneur précédent

# Debug avec ephemeral containers
kubectl debug <pod-name> -it --image=busybox -- /bin/sh

# Installation de metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Installation de Prometheus et Grafana (avec Helm)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/prometheus
helm install grafana stable/grafana
```

## Section 7 : Patterns avancés et DevOps

### 7.1 GitOps avec ArgoCD

```bash
# Installation d'ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Accès à l'interface web
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Récupération du mot de passe admin
kubectl get pods -n argocd
kubectl logs -n argocd argocd-server-xyz

# Connexion CLI
argocd login localhost:8080

# Créer une application
argocd app create myapp \
  --repo https://github.com/myorg/myapp.git \
  --path k8s \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default

# Synchroniser l'application
argocd app sync myapp

# Status de l'application
argocd app get myapp
```

```yaml
# argocd-app.yaml - Application ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
    - PrunePropagationPolicy=foreground
```

### 7.2 Intégration CI/CD

```yaml
# .github/workflows/deploy.yml - Pipeline GitHub Actions
name: Deploy to Kubernetes

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Setup Docker Buildx
      uses: docker/setup-buildx-action@v2
      
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v3
      with:
        context: .
        push: true
        tags: myapp/web:${{ github.sha }}, myapp/web:latest
    
    - name: Update deployment image
      run: |
        sed -i 's|image: myapp/web:.*|image: myapp/web:${{ github.sha }}|g' k8s/deployment.yaml
    
    - name: Commit updated deployment
      run: |
        git config --local user.email "action@github.com"
        git config --local user.name "GitHub Action"
        git add k8s/deployment.yaml
        git commit -m "Update deployment image to ${{ github.sha }}"
        git push
    
    - name: Setup kubectl
      uses: azure/k8s-set-context@v2
      with:
        method: kubeconfig
        kubeconfig: ${{ secrets.KUBE_CONFIG }}
    
    - name: Deploy to Kubernetes
      run: |
        kubectl apply -f k8s/
        kubectl rollout status deployment/web-app
    
    - name: Run tests
      run: |
        kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- curl -f http://web-app-service/health
```

### 7.3 Sécurité des conteneurs

```bash
echo "=== SÉCURITÉ DES CONTENEURS ==="
echo ""
echo "1. Images de base sûres:"
echo "   • Utiliser des images officielles"
echo "   • Scanner les vulnérabilités (Trivy, Clair)"
echo "   • Mettre à jour régulièrement"
echo ""
echo "2. Utilisateurs non-root:"
echo "   • USER directive dans Dockerfile"
echo "   • Droits minimum nécessaires"
echo ""
echo "3. Secrets et configurations:"
echo "   • Ne pas stocker dans l'image"
echo "   • Utiliser ConfigMaps et Secrets"
echo "   • Rotation régulière"
echo ""
echo "4. Network policies:"
echo "   • Restreindre le trafic réseau"
echo "   • Principe du moindre privilège"
echo ""
echo "5. Resource limits:"
echo "   • CPU et mémoire limits"
echo "   • Prévention des attaques DoS"
echo ""
echo "6. Monitoring et logging:"
echo "   • Audit des accès"
echo "   • Détection d'anomalies"
```

```yaml
# security-context.yaml - Contextes de sécurité
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    runAsNonRoot: true
  containers:
  - name: app
    image: myapp/web:v1.0
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
        - ALL
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

## Conclusion : L'ère des conteneurs

Docker et Kubernetes ont révolutionné le déploiement d'applications, permettant des architectures cloud-native évolutives et maintenables. L'orchestration de conteneurs est devenue le standard pour les applications modernes.

Dans le prochain chapitre, nous explorerons les pratiques DevOps avancées, incluant les pipelines CI/CD complets, l'intégration Terraform/Ansible/Kubernetes, et les stratégies de déploiement modernes.

---

**Exercice pratique :** Créez une application complète avec Docker et Kubernetes qui inclut :
1. Application multi-conteneurs avec Docker Compose
2. Déploiement sur Kubernetes avec ingress et services
3. Configuration avec ConfigMaps et Secrets
4. Autoscaling et monitoring
5. Pipeline CI/CD avec tests automatisés

**Challenge avancé :** Développez une plateforme d'orchestration multi-cloud qui :
- Supporte Kubernetes, Docker Swarm et ECS
- Implémente des patterns de déploiement avancés (blue-green, canary)
- Gère la sécurité et la conformité
- Fournit des dashboards de monitoring unifiés
- S'intègre avec les outils DevOps existants

**Réflexion :** Comment les conteneurs ont-ils transformé le développement logiciel, et quelles sont les implications pour les équipes d'exploitation et les architectures applicatives ?

