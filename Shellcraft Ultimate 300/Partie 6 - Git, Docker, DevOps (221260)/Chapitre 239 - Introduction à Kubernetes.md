# Chapitre 239 - Introduction à Kubernetes

## Table des matières
- [Introduction](#introduction)
- [Concepts Kubernetes](#concepts-kubernetes)
- [Déploiement de base](#déploiement-de-base)
- [Services et networking](#services-et-networking)
- [Scaling et gestion](#scaling-et-gestion)
- [Conclusion](#conclusion)

## Introduction

Kubernetes (K8s) est la plateforme d'orchestration de conteneurs la plus populaire. Elle automatise le déploiement, le scaling, et la gestion des applications conteneurisées.

Imaginez Kubernetes comme un chef d'orchestre pour conteneurs : il gère automatiquement où déployer vos conteneurs, comment les faire communiquer, et comment les faire évoluer selon la charge.

## Concepts Kubernetes

### Architecture Kubernetes

**Composants principaux** :
```bash
#!/bin/bash
# Concepts Kubernetes de base

# Pod : plus petite unité déployable
cat > pod-example.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  containers:
  - name: myapp
    image: myapp:latest
    ports:
    - containerPort: 80
EOF

# Deployment : gestion des Pods
cat > deployment-example.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 80
EOF

# Service : exposition des Pods
cat > service-example.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
EOF
```

## Déploiement de base

### Scripts de déploiement

**Déploiement Kubernetes** :
```bash
#!/bin/bash
# Déploiement Kubernetes de base

deploy_to_k8s() {
    local namespace="${1:-default}"
    local image_tag="${2:-latest}"
    
    echo "=== Déploiement Kubernetes ==="
    
    # Créer le namespace si nécessaire
    kubectl create namespace "$namespace" --dry-run=client -o yaml | kubectl apply -f -
    
    # Déployer
    kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${PROJECT_NAME}
  namespace: ${namespace}
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ${PROJECT_NAME}
  template:
    metadata:
      labels:
        app: ${PROJECT_NAME}
    spec:
      containers:
      - name: ${PROJECT_NAME}
        image: ${DOCKER_REGISTRY}/${PROJECT_NAME}:${image_tag}
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
---
apiVersion: v1
kind: Service
metadata:
  name: ${PROJECT_NAME}-service
  namespace: ${namespace}
spec:
  selector:
    app: ${PROJECT_NAME}
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: LoadBalancer
EOF
    
    echo "✓ Déploiement Kubernetes terminé"
}

# Vérifier le statut
check_k8s_status() {
    local namespace="${1:-default}"
    
    echo "=== Statut Kubernetes ==="
    kubectl get pods -n "$namespace"
    kubectl get services -n "$namespace"
    kubectl get deployments -n "$namespace"
}
```

## Services et networking

### Gestion des services

**Scripts de service** :
```bash
#!/bin/bash
# Gestion des services Kubernetes

# Exposer un service
expose_service() {
    local service_name="$1"
    local port="${2:-80}"
    local namespace="${3:-default}"
    
    kubectl expose deployment "$service_name" \
        --type=LoadBalancer \
        --port="$port" \
        --namespace="$namespace"
}

# Port forwarding
port_forward() {
    local service_name="$1"
    local local_port="${2:-8080}"
    local remote_port="${3:-80}"
    local namespace="${4:-default}"
    
    kubectl port-forward \
        "service/$service_name" \
        "$local_port:$remote_port" \
        -n "$namespace"
}
```

## Scaling et gestion

### Auto-scaling

**Scripts de scaling** :
```bash
#!/bin/bash
# Scaling Kubernetes

# Scale manuel
scale_deployment() {
    local deployment="$1"
    local replicas="$2"
    local namespace="${3:-default}"
    
    kubectl scale deployment "$deployment" \
        --replicas="$replicas" \
        -n "$namespace"
}

# Auto-scaling HPA
setup_hpa() {
    local deployment="$1"
    local min_replicas="${2:-2}"
    local max_replicas="${3:-10}"
    local cpu_percent="${4:-80}"
    local namespace="${5:-default}"
    
    kubectl autoscale deployment "$deployment" \
        --min="$min_replicas" \
        --max="$max_replicas" \
        --cpu-percent="$cpu_percent" \
        -n "$namespace"
}
```

## Conclusion

Kubernetes offre une orchestration puissante pour les applications conteneurisées. Avec ses concepts de Pods, Deployments, et Services, vous pouvez gérer des applications complexes à grande échelle.

Dans le chapitre suivant, nous explorerons les projets pratiques Kubernetes et les cas d'usage réels.

