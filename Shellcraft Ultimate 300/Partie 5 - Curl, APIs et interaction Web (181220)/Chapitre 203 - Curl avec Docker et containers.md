# Chapitre 203 - Curl avec Docker et containers

## Table des matières
- [Introduction](#introduction)
- [cURL dans Docker](#curl-dans-docker)
- [Scripts containerisés](#scripts-containerisés)
- [Orchestration](#orchestration)
- [Conclusion](#conclusion)

## Introduction

L'utilisation de cURL avec Docker permet de créer des environnements isolés et reproductibles pour vos scripts d'interaction API.

## cURL dans Docker

**Image Docker avec cURL** :
```dockerfile
# Dockerfile
FROM alpine:latest

RUN apk add --no-cache curl jq

WORKDIR /app

COPY scripts/ .

CMD ["sh", "api_script.sh"]
```

**Utilisation** :
```bash
#!/bin/bash
# Utilisation de cURL dans Docker

run_in_docker() {
    docker run --rm \
        -v "$(pwd):/app" \
        -e API_TOKEN="$API_TOKEN" \
        curl-api:latest \
        ./api_script.sh
}
```

## Scripts containerisés

**Scripts Docker Compose** :
```yaml
# docker-compose.yml
version: '3.8'

services:
  api-client:
    build: .
    environment:
      - API_TOKEN=${API_TOKEN}
      - API_BASE_URL=${API_BASE_URL}
    volumes:
      - ./scripts:/app/scripts
    command: ./scripts/monitor.sh
```

## Orchestration

**Orchestration avec Kubernetes** :
```yaml
# k8s/api-client.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: api-monitor
spec:
  schedule: "*/5 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: api-client
            image: curl-api:latest
            env:
            - name: API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: api-secrets
                  key: token
```

## Conclusion

Docker et les containers permettent d'isoler et d'orchestrer vos scripts cURL de manière professionnelle.

