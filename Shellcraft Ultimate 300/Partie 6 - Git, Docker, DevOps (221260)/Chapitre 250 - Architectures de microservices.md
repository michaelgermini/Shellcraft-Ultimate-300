# Chapitre 250 - Architectures de microservices

## Table des matières
- [Introduction](#introduction)
- [Concepts de microservices](#concepts-de-microservices)
- [Communication entre services](#communication-entre-services)
- [Gestion de la configuration](#gestion-de-la-configuration)
- [Conclusion](#conclusion)

## Introduction

Les architectures de microservices décomposent les applications en services indépendants et déployables séparément.

## Concepts de microservices

**Déploiement de microservices** :
```yaml
# microservices-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:latest
        ports:
        - containerPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 8081
```

## Communication entre services

**Service Mesh avec Istio** :
```yaml
# istio-virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: user-service
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 80
    - destination:
        host: user-service
        subset: v2
      weight: 20
```

## Gestion de la configuration

**ConfigMap et Secrets** :
```yaml
# config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://db:5432/mydb"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  api_key: <base64-encoded-key>
```

## Conclusion

Les microservices offrent flexibilité et scalabilité au prix d'une complexité accrue de gestion.

