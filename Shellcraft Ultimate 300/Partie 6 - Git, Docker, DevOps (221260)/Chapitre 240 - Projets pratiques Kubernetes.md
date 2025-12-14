# Chapitre 240 - Projets pratiques Kubernetes

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Application web complète](#projet-1--application-web-complète)
- [Projet 2 : Base de données avec StatefulSet](#projet-2--base-de-données-avec-statefulset)
- [Projet 3 : Monitoring avec Prometheus](#projet-3--monitoring-avec-prometheus)
- [Conclusion](#conclusion)

## Introduction

Ces projets pratiques Kubernetes démontrent l'application réelle de Kubernetes dans des scénarios de production, combinant plusieurs concepts pour créer des solutions complètes.

## Projet 1 : Application web complète

**Déploiement complet** :
```yaml
# k8s/web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
        image: web-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
```

**Script de déploiement** :
```bash
#!/bin/bash
# Déploiement application web Kubernetes

deploy_web_app() {
    kubectl apply -f k8s/web-app.yaml
    kubectl rollout status deployment/web-app
    echo "✓ Application web déployée"
}
```

## Projet 2 : Base de données avec StatefulSet

**StatefulSet pour base de données** :
```yaml
# k8s/database-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
```

## Projet 3 : Monitoring avec Prometheus

**Stack de monitoring** :
```yaml
# k8s/monitoring.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: config
          mountPath: /etc/prometheus
      volumes:
      - name: config
        configMap:
          name: prometheus-config
```

## Conclusion

Ces projets pratiques démontrent l'utilisation réelle de Kubernetes pour déployer des applications complètes en production.

Dans le chapitre suivant, nous explorerons le workflow DevOps complet qui intègre tous les concepts appris.

