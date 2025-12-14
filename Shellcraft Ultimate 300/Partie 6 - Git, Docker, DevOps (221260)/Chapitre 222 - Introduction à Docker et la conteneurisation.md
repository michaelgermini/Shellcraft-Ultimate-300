# Chapitre 222 - Introduction à Docker et la conteneurisation

> "Les conteneurs Docker n'ont pas résolu le problème du déploiement - ils l'ont fait disparaître. Pour la première fois, 'ça marche sur ma machine' est devenu une déclaration de succès plutôt qu'une excuse." - Évangéliste Docker

## Introduction : La révolution des conteneurs

Docker représente une rupture fondamentale dans la façon dont nous concevons, développons et déployons les applications. En permettant d'empaqueter applications et dépendances dans des conteneurs légers et portables, Docker a transformé l'infrastructure logicielle d'un art complexe en une science reproductible.

Dans ce chapitre, nous explorerons les concepts fondamentaux de la conteneurisation, l'architecture de Docker, et comment intégrer les conteneurs dans les workflows DevOps modernes.

## Section 1 : Concepts fondamentaux de la conteneurisation

### 1.1 De la virtualisation aux conteneurs

**Virtualisation traditionnelle :**
- Machine virtuelle complète avec OS invité
- Hyperviseur gérant les ressources matérielles
- Isolation complète mais overhead important
- Démarrage en minutes

**Conteneurisation :**
- Processus isolés partageant le kernel hôte
- Isolation au niveau système de fichiers et réseau
- Overhead minimal, démarrage en secondes
- Portabilité et densité élevées

```bash
# Comparaison des approches
echo "=== VIRTUALISATION TRADITIONNELLE ==="
echo "VM Size: ~GB (OS + Application)"
echo "Boot Time: Minutes"
echo "Resource Usage: High overhead"
echo "Isolation: Complete (hardware level)"
echo ""

echo "=== CONTENEURISATION ==="
echo "Container Size: ~MB (Application only)"
echo "Boot Time: Seconds"
echo "Resource Usage: Minimal overhead"
echo "Isolation: Kernel level (namespaces, cgroups)"
```

### 1.2 Architecture de Docker

**Composants principaux :**

1. **Docker Engine (Moteur)**
   - Daemon dockerd
   - API REST
   - CLI docker

2. **Images**
   - Templates immuables
   - Système de fichiers en couches
   - Métadonnées et configuration

3. **Conteneurs**
   - Instances d'images en cours d'exécution
   - Isolation via namespaces et cgroups
   - Éphémères par défaut

4. **Registries**
   - Stockage et distribution d'images
   - Docker Hub, registries privés
   - Contrôle d'accès et versioning

```bash
# Architecture Docker en action
echo "=== CYCLE DE VIE DOCKER ==="
echo "1. PULL: Téléchargement d'image depuis registry"
echo "   docker pull nginx:latest"
echo ""
echo "2. RUN: Création et démarrage de conteneur"
echo "   docker run -d -p 80:80 nginx:latest"
echo ""
echo "3. BUILD: Construction d'image personnalisée"
echo "   docker build -t myapp:latest ."
echo ""
echo "4. PUSH: Publication d'image vers registry"
echo "   docker push myuser/myapp:latest"
```

## Section 2 : Premiers pas avec Docker

### 2.1 Installation et configuration

```bash
# Installation sur Ubuntu/Debian
sudo apt-get update
sudo apt-get install apt-transport-https ca-certificates curl gnupg lsb-release

# Ajout de la clé GPG officielle Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Ajout du repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installation
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

# Démarrage du service
sudo systemctl start docker
sudo systemctl enable docker

# Ajout utilisateur au groupe docker (évite sudo)
sudo usermod -aG docker $USER
# Note: Relancer la session ou utiliser 'newgrp docker'

# Vérification de l'installation
docker --version
docker run hello-world
```

```bash
# Installation sur CentOS/RHEL
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Installation sur macOS (via Homebrew)
brew install --cask docker

# Installation sur Windows
# Via Chocolatey
choco install docker-desktop

# Ou téléchargement direct depuis docker.com
```

### 2.2 Commandes Docker essentielles

```bash
# Informations système
docker version          # Version de Docker
docker info            # Informations détaillées
docker system df       # Utilisation du disque
docker system prune    # Nettoyage des ressources inutilisées

# Gestion des images
docker images          # Lister les images locales
docker pull nginx      # Télécharger une image
docker rmi nginx       # Supprimer une image
docker tag nginx mynginx  # Taguer une image
docker save nginx > nginx.tar  # Exporter une image
docker load < nginx.tar        # Importer une image

# Gestion des conteneurs
docker ps              # Conteneurs en cours
docker ps -a           # Tous les conteneurs
docker run nginx       # Démarrer un conteneur
docker stop <id>       # Arrêter un conteneur
docker rm <id>         # Supprimer un conteneur
docker logs <id>       # Logs d'un conteneur
docker exec -it <id> bash  # Shell interactif
```

### 2.3 Premier conteneur interactif

```bash
# Conteneur simple
docker run hello-world

# Conteneur avec shell interactif
docker run -it ubuntu:latest /bin/bash

# Conteneur en arrière-plan (daemon)
docker run -d nginx:latest

# Exposition de ports
docker run -d -p 8080:80 nginx:latest

# Montage de volumes
docker run -v /host/path:/container/path ubuntu:latest

# Variables d'environnement
docker run -e MY_VAR=value nginx:latest

# Nom personnalisé
docker run --name my-container nginx:latest

# Redémarrage automatique
docker run --restart unless-stopped nginx:latest
```

## Section 3 : Construction d'images Docker

### 3.1 Le Dockerfile

```dockerfile
# Exemple de Dockerfile complet
FROM ubuntu:20.04

# Labels pour métadonnées
LABEL maintainer="your-email@example.com"
LABEL version="1.0"
LABEL description="Exemple d'application conteneurisée"

# Variables d'environnement
ENV APP_HOME=/app
ENV PORT=3000

# Installation des dépendances système
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# Création de l'utilisateur non-root
RUN useradd -m -s /bin/bash appuser

# Création des répertoires
RUN mkdir -p $APP_HOME && chown appuser:appuser $APP_HOME

# Changement d'utilisateur
USER appuser

# Répertoire de travail
WORKDIR $APP_HOME

# Copie des fichiers de l'application
COPY --chown=appuser:appuser package*.json ./
COPY --chown=appuser:appuser src/ ./src/

# Installation des dépendances npm
RUN npm ci --only=production

# Exposition du port
EXPOSE $PORT

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:$PORT/health || exit 1

# Volume pour données persistantes
VOLUME ["/app/data"]

# Commande de démarrage
CMD ["npm", "start"]
```

### 3.2 Construction et optimisation d'images

```bash
# Construction basique
docker build -t myapp:latest .

# Construction avec build context spécifique
docker build -t myapp:v1.0 -f Dockerfile.prod .

# Utilisation du cache de build
docker build --no-cache -t myapp:latest .

# Build multi-stage pour optimiser la taille
cat > Dockerfile.multistage << 'EOF'
# Stage 1: Build
FROM node:16-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# Stage 2: Runtime
FROM nginx:alpine

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# Construction du build multi-stage
docker build -f Dockerfile.multistage -t myapp:optimized .

# Comparaison des tailles
docker images myapp
# myapp:latest     ~500MB (avec node et sources)
# myapp:optimized  ~20MB  (nginx seulement)
```

### 3.3 Gestion avancée des layers

```bash
# Inspection des layers d'une image
docker history nginx:latest

# Optimisation de l'ordre des instructions
cat > Dockerfile.optimized << 'EOF'
FROM ubuntu:20.04

# Installation des dépendances (regrouper pour un layer)
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copie des fichiers statiques (avant les dépendances changeantes)
COPY config/static.conf /etc/app/static.conf

# Installation des dépendances d'application
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copie du code source (change souvent, layer séparé)
COPY . /app

CMD ["/app/start.sh"]
EOF

# Utilisation de .dockerignore pour réduire le contexte
cat > .dockerignore << 'EOF'
.git
.gitignore
README.md
docs/
tests/
*.log
.env
.DS_Store
Thumbs.db
EOF

# Build avec contexte optimisé
docker build -t myapp:lean .
```

## Section 4 : Réseau et volumes Docker

### 4.1 Configuration réseau

```bash
# Réseaux Docker par défaut
docker network ls

# Création de réseaux personnalisés
docker network create --driver bridge my-network
docker network create --driver overlay my-overlay-network

# Connexion de conteneurs à un réseau
docker run -d --name web --network my-network nginx
docker run -d --name api --network my-network myapi:latest

# Inspection du réseau
docker network inspect my-network

# Communication inter-conteneurs
docker exec web ping api  # Depuis conteneur web vers api

# Exposition de ports avec mapping
docker run -d -p 8080:80 --name web nginx
# Port 80 du conteneur exposé sur port 8080 de l'hôte

# Exposition dynamique (port assigné automatiquement)
docker run -d -P --name web nginx
docker port web  # Voir le mapping

# Mode host (pas d'isolation réseau)
docker run --network host nginx
```

### 4.2 Gestion des volumes

```bash
# Volumes nommés
docker volume create my-data
docker run -v my-data:/data alpine ls /data

# Montage de répertoires hôtes
docker run -v /host/path:/container/path alpine ls /container/path

# Montage en lecture seule
docker run -v /host/path:/container/path:ro alpine touch /container/path/test

# Volumes anonymes (auto-générés)
docker run -v /data alpine ls /data

# Inspection des volumes
docker volume ls
docker volume inspect my-data

# Sauvegarde de volumes
docker run --rm -v my-data:/source -v /host/backup:/backup alpine tar czf /backup/my-data.tar.gz -C /source .

# Restauration de volumes
docker run --rm -v my-data:/dest -v /host/backup:/backup alpine tar xzf /backup/my-data.tar.gz -C /dest

# Nettoyage des volumes
docker volume prune  # Volumes anonymes non utilisés
docker volume rm $(docker volume ls -q)  # Tous les volumes
```

### 4.3 Docker Compose pour orchestration multi-conteneurs

```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - api
    environment:
      - API_URL=http://api:3000
    networks:
      - frontend
      - backend

  api:
    build: ./api
    ports:
      - "3000:3000"
    volumes:
      - api-data:/app/data
      - ./logs:/app/logs
    environment:
      - DB_HOST=database
      - REDIS_URL=redis://cache:6379
    depends_on:
      - database
      - cache
    networks:
      - backend
    restart: unless-stopped

  database:
    image: postgres:13-alpine
    environment:
      - POSTGRES_DB=myapp
      - POSTGRES_USER=myapp
      - POSTGRES_PASSWORD=secret
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - backend
    restart: unless-stopped

  cache:
    image: redis:alpine
    volumes:
      - cache-data:/data
    networks:
      - backend
    restart: unless-stopped

volumes:
  api-data:
  db-data:
  cache-data:

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
```

```bash
# Gestion avec docker-compose
docker-compose up -d        # Démarrage de tous les services
docker-compose ps           # Statut des services
docker-compose logs         # Logs de tous les services
docker-compose logs web     # Logs d'un service spécifique
docker-compose exec web bash  # Shell dans un conteneur
docker-compose stop         # Arrêt de tous les services
docker-compose down         # Arrêt et suppression
docker-compose down -v      # Avec suppression des volumes

# Construction et déploiement
docker-compose build        # Construction des images
docker-compose push         # Push vers registry
docker-compose pull         # Pull des images

# Scaling de services
docker-compose up -d --scale api=3

# Mise à jour progressive
docker-compose up -d --no-deps web
```

## Section 5 : Sécurité et bonnes pratiques Docker

### 5.1 Images sécurisées

```bash
# Utilisation d'images officielles et vérifiées
docker pull alpine:latest          # Image officielle minimale
docker pull nginx:alpine          # Version alpine plus légère

# Vérification des vulnérabilités
docker scan nginx:latest

# Construction d'images non-root
cat > Dockerfile.secure << 'EOF'
FROM node:16-alpine

# Création d'utilisateur non-privilégié
RUN addgroup -g 1001 -S appuser && \
    adduser -S -D -H -u 1001 -h /app -s /sbin/nologin -G appuser -g appuser appuser

WORKDIR /app

# Installation des dépendances en tant que root
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

# Changement de propriétaire des fichiers
RUN chown -R appuser:appuser /app

# Changement d'utilisateur
USER appuser

COPY . .

EXPOSE 3000
CMD ["npm", "start"]
EOF

# Scan de sécurité pendant le build
docker build --security-opt "label=disable" -t myapp:secure .

# Utilisation de secrets lors du build
echo "mysecretpassword" | docker build --secret id=mysecret -t myapp:with-secret .
```

### 5.2 Durcissement des conteneurs

```bash
# Capabilities réduites
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE nginx

# Mode read-only
docker run --read-only -v /tmp nginx

# Security options
docker run --security-opt no-new-privileges nginx
docker run --security-opt apparmor=docker-default nginx

# Limitation des ressources
docker run -m 512m --cpus 0.5 nginx  # Mémoire et CPU limités

# Health checks
docker run --health-cmd "curl -f http://localhost/ || exit 1" --health-interval 30s nginx

# Secrets management
echo "mypassword" | docker secret create db_password -
docker service create --secret db_password nginx

# Logging sécurisé
docker run --log-driver json-file --log-opt max-size=10m --log-opt max-file=3 nginx
```

### 5.3 Registres et distribution sécurisée

```bash
# Connexion à un registry privé
docker login myregistry.com
# Entrer username/password

# Push vers registry privé
docker tag myapp:latest myregistry.com/myapp:v1.0
docker push myregistry.com/myapp:v1.0

# Utilisation de secrets pour l'authentification
docker build --secret id=npm_token --build-arg NPM_TOKEN .
```

## Section 6 : Intégration DevOps avec Docker

### 6.1 CI/CD avec Docker

```bash
# Pipeline CI/CD simple avec Docker
#!/bin/bash

# Configuration
IMAGE_NAME="myapp"
TAG="${CI_COMMIT_REF_NAME:-latest}"
REGISTRY="registry.example.com"

echo "=== PIPELINE CI/CD DOCKER ==="

# Étape 1: Construction
echo "1. Construction de l'image..."
docker build -t "$IMAGE_NAME:$TAG" .

if [[ $? -ne 0 ]]; then
    echo "❌ Échec de la construction"
    exit 1
fi

# Étape 2: Tests
echo "2. Exécution des tests..."
docker run --rm "$IMAGE_NAME:$TAG" npm test

if [[ $? -ne 0 ]]; then
    echo "❌ Échec des tests"
    exit 1
fi

# Étape 3: Analyse de sécurité
echo "3. Analyse de sécurité..."
docker scan "$IMAGE_NAME:$TAG"

# Étape 4: Taggage et push
echo "4. Publication de l'image..."
docker tag "$IMAGE_NAME:$TAG" "$REGISTRY/$IMAGE_NAME:$TAG"
docker push "$REGISTRY/$IMAGE_NAME:$TAG"

if [[ $? -ne 0 ]]; then
    echo "❌ Échec de la publication"
    exit 1
fi

# Étape 5: Déploiement (optionnel)
if [[ "$DEPLOY_ENV" == "production" ]]; then
    echo "5. Déploiement en production..."
    docker-compose -f docker-compose.prod.yml up -d
fi

echo "✅ Pipeline terminé avec succès"
```

### 6.2 Monitoring des conteneurs

```bash
# Statistiques des conteneurs
docker stats

# Inspection détaillée
docker inspect mycontainer

# Logs centralisés
docker run -d --name logstash -p 5044:5044 logstash:latest

# Configuration des logs
docker run --log-driver gelf --log-opt gelf-address=udp://logstash:5044 nginx

# Health checks automatisés
cat > Dockerfile.health << 'EOF'
FROM nginx:alpine

COPY healthcheck.sh /healthcheck.sh
RUN chmod +x /healthcheck.sh

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD /healthcheck.sh

CMD ["nginx", "-g", "daemon off;"]
EOF

cat > healthcheck.sh << 'EOF'
#!/bin/sh

# Vérification de la santé du service
if curl -f http://localhost/health > /dev/null 2>&1; then
    exit 0
else
    exit 1
fi
EOF

# Monitoring avec Prometheus
cat > docker-compose.monitoring.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter
    ports:
      - "9100:9100"
EOF
```

## Conclusion : Docker comme fondation DevOps

Docker a transformé l'empaquetage et le déploiement d'applications d'une discipline complexe en une pratique standardisée et reproductible. En permettant de créer des environnements isolés et portables, Docker devient l'infrastructure de base des workflows DevOps modernes.

Dans le prochain chapitre, nous explorerons les pipelines CI/CD avancés, découvrant comment orchestrer automatiquement les tests, les builds, et les déploiements dans des environnements cloud-native.

---

**Exercice pratique :** Créez un environnement de développement complet avec Docker qui inclut :
1. Application multi-conteneurs (web + API + base de données)
2. Configuration de développement et production
3. Volumes pour la persistance des données
4. Réseau isolé pour la sécurité
5. Monitoring intégré avec health checks

**Challenge avancé :** Développez un système de déploiement blue/green avec Docker qui :
- Déploie une nouvelle version en parallèle
- Teste automatiquement la santé des conteneurs
- Bascule le trafic progressivement
- Rollback automatique en cas de problème
- Logging complet de toutes les opérations

**Réflexion :** Comment Docker change-t-il fondamentalement l'approche du développement logiciel, et quels sont les impacts sur la productivité des équipes et la fiabilité des déploiements ?

