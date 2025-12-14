# Chapitre 232 - Docker Compose et multi-conteneurs

## Table des matières
- [Introduction](#introduction)
- [Docker Compose de base](#docker-compose-de-base)
- [Orchestration multi-conteneurs](#orchestration-multi-conteneurs)
- [Réseaux et volumes](#réseaux-et-volumes)
- [Gestion avancée](#gestion-avancée)
- [Conclusion](#conclusion)

## Introduction

Docker Compose simplifie la gestion d'applications multi-conteneurs en définissant et orchestrant plusieurs conteneurs avec un simple fichier YAML. C'est l'outil idéal pour développer et déployer des applications complexes.

Imaginez Docker Compose comme un chef d'orchestre : il coordonne tous les conteneurs (les musiciens) pour créer une symphonie harmonieuse (votre application complète).

## Docker Compose de base

### Fichier docker-compose.yml

**Configuration de base** :
```yaml
# docker-compose.yml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=development
    volumes:
      - .:/app
    depends_on:
      - db
      - redis

  db:
    image: postgres:14
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

**Scripts de gestion** :
```bash
#!/bin/bash
# Gestion Docker Compose

# Démarrer tous les services
start_services() {
    docker-compose up -d
    echo "Services démarrés"
}

# Arrêter tous les services
stop_services() {
    docker-compose down
    echo "Services arrêtés"
}

# Voir les logs
view_logs() {
    local service="${1:-}"
    
    if [ -z "$service" ]; then
        docker-compose logs -f
    else
        docker-compose logs -f "$service"
    fi
}

# Reconstruire les images
rebuild() {
    docker-compose build --no-cache
    docker-compose up -d
}

# Redémarrer un service
restart_service() {
    local service="$1"
    docker-compose restart "$service"
}
```

## Orchestration multi-conteneurs

### Services interdépendants

**Orchestration avancée** :
```yaml
# docker-compose.advanced.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    command: python app.py
    environment:
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
      - REDIS_URL=redis://redis:6379
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:14
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - app
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
```

## Réseaux et volumes

### Gestion des réseaux

**Scripts réseau** :
```bash
#!/bin/bash
# Gestion des réseaux Docker Compose

# Créer un réseau personnalisé
create_network() {
    docker network create app-network
}

# Inspecter le réseau
inspect_network() {
    docker network inspect "$(docker-compose ps -q | head -1 | xargs docker inspect --format='{{range .NetworkSettings.Networks}}{{.NetworkID}}{{end}}')"
}

# Lister les réseaux
list_networks() {
    docker network ls
}
```

### Gestion des volumes

**Scripts volumes** :
```bash
#!/bin/bash
# Gestion des volumes Docker Compose

# Sauvegarder un volume
backup_volume() {
    local volume="$1"
    local backup_file="${2:-backup_$(date +%Y%m%d_%H%M%S).tar}"
    
    docker run --rm \
        -v "$volume":/data \
        -v "$(pwd)":/backup \
        alpine tar czf "/backup/$backup_file" /data
    
    echo "Volume sauvegardé: $backup_file"
}

# Restaurer un volume
restore_volume() {
    local volume="$1"
    local backup_file="$2"
    
    docker run --rm \
        -v "$volume":/data \
        -v "$(pwd)":/backup \
        alpine tar xzf "/backup/$backup_file" -C /data
    
    echo "Volume restauré depuis $backup_file"
}
```

## Gestion avancée

### Scaling et production

**Scaling de services** :
```bash
#!/bin/bash
# Scaling avec Docker Compose

# Scale un service
scale_service() {
    local service="$1"
    local replicas="$2"
    
    docker-compose up -d --scale "$service=$replicas"
}

# Exemple: Scale le service web à 3 instances
# scale_service web 3
```

**Configuration production** :
```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  app:
    build: .
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - NODE_ENV=production
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

## Conclusion

Docker Compose simplifie considérablement la gestion d'applications multi-conteneurs. En définissant vos services dans un fichier YAML, vous orchestrez facilement des applications complexes.

Dans le chapitre suivant, nous explorerons la gestion multi-serveur et le déploiement distribué.

