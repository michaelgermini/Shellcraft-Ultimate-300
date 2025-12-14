# Chapitre 238 - Déploiement cloud

## Table des matières
- [Introduction](#introduction)
- [Déploiement AWS](#déploiement-aws)
- [Déploiement Azure](#déploiement-azure)
- [Déploiement GCP](#déploiement-gcp)
- [Multi-cloud](#multi-cloud)
- [Conclusion](#conclusion)

## Introduction

Le déploiement cloud permet de déployer vos applications sur des infrastructures cloud scalables et fiables. Chaque fournisseur cloud offre ses propres outils et services.

Imaginez le cloud comme un terrain de jeu infini : vous pouvez créer, détruire, et redimensionner vos infrastructures à volonté, payant seulement ce que vous utilisez.

## Déploiement AWS

### Scripts AWS

**Déploiement sur AWS ECS** :
```bash
#!/bin/bash
# Déploiement AWS ECS

set -euo pipefail

AWS_REGION="${AWS_REGION:-us-east-1}"
CLUSTER_NAME="${CLUSTER_NAME:-myapp-cluster}"
SERVICE_NAME="${SERVICE_NAME:-myapp-service}"

deploy_to_aws() {
    local image_tag="$1"
    
    echo "=== Déploiement AWS ECS ==="
    
    # Build et push vers ECR
    build_and_push_ecr "$image_tag"
    
    # Mettre à jour le service
    update_ecs_service "$image_tag"
    
    # Attendre le déploiement
    wait_for_deployment
    
    echo "✓ Déploiement AWS terminé"
}

build_and_push_ecr() {
    local tag="$1"
    local ecr_repo="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${PROJECT_NAME}"
    
    # Login ECR
    aws ecr get-login-password --region "$AWS_REGION" | \
        docker login --username AWS --password-stdin "$ecr_repo"
    
    # Build
    docker build -t "${ecr_repo}:${tag}" .
    
    # Push
    docker push "${ecr_repo}:${tag}"
    
    echo "✓ Image poussée vers ECR: ${ecr_repo}:${tag}"
}

update_ecs_service() {
    local tag="$1"
    
    # Mettre à jour le service avec la nouvelle image
    aws ecs update-service \
        --cluster "$CLUSTER_NAME" \
        --service "$SERVICE_NAME" \
        --force-new-deployment \
        --region "$AWS_REGION"
}

wait_for_deployment() {
    echo "Attente du déploiement..."
    aws ecs wait services-stable \
        --cluster "$CLUSTER_NAME" \
        --services "$SERVICE_NAME" \
        --region "$AWS_REGION"
    
    echo "✓ Service stable"
}
```

## Déploiement Azure

### Scripts Azure

**Déploiement sur Azure Container Instances** :
```bash
#!/bin/bash
# Déploiement Azure Container Instances

set -euo pipefail

AZURE_RESOURCE_GROUP="${AZURE_RESOURCE_GROUP:-myapp-rg}"
AZURE_REGION="${AZURE_REGION:-eastus}"

deploy_to_azure() {
    local image_tag="$1"
    
    echo "=== Déploiement Azure ==="
    
    # Build et push vers ACR
    build_and_push_acr "$image_tag"
    
    # Déployer sur ACI
    deploy_aci "$image_tag"
    
    echo "✓ Déploiement Azure terminé"
}

build_and_push_acr() {
    local tag="$1"
    local acr_name="${AZURE_ACR_NAME}"
    local acr_login="${acr_name}.azurecr.io"
    
    # Login ACR
    az acr login --name "$acr_name"
    
    # Build
    az acr build \
        --registry "$acr_name" \
        --image "${PROJECT_NAME}:${tag}" \
        .
    
    echo "✓ Image poussée vers ACR"
}

deploy_aci() {
    local tag="$1"
    
    az container create \
        --resource-group "$AZURE_RESOURCE_GROUP" \
        --name "$PROJECT_NAME" \
        --image "${AZURE_ACR_NAME}.azurecr.io/${PROJECT_NAME}:${tag}" \
        --registry-login-server "${AZURE_ACR_NAME}.azurecr.io" \
        --registry-username "$AZURE_ACR_USERNAME" \
        --registry-password "$AZURE_ACR_PASSWORD" \
        --dns-name-label "$PROJECT_NAME" \
        --ports 80
    
    echo "✓ Container déployé sur ACI"
}
```

## Déploiement GCP

### Scripts GCP

**Déploiement sur Google Cloud Run** :
```bash
#!/bin/bash
# Déploiement Google Cloud Run

set -euo pipefail

GCP_PROJECT="${GCP_PROJECT:-myapp-project}"
GCP_REGION="${GCP_REGION:-us-central1}"

deploy_to_gcp() {
    local image_tag="$1"
    
    echo "=== Déploiement GCP ==="
    
    # Build et push vers GCR
    build_and_push_gcr "$image_tag"
    
    # Déployer sur Cloud Run
    deploy_cloud_run "$image_tag"
    
    echo "✓ Déploiement GCP terminé"
}

build_and_push_gcr() {
    local tag="$1"
    local gcr_repo="gcr.io/${GCP_PROJECT}/${PROJECT_NAME}"
    
    # Build avec Cloud Build
    gcloud builds submit \
        --tag "${gcr_repo}:${tag}" \
        --project "$GCP_PROJECT"
    
    echo "✓ Image poussée vers GCR"
}

deploy_cloud_run() {
    local tag="$1"
    
    gcloud run deploy "$PROJECT_NAME" \
        --image "gcr.io/${GCP_PROJECT}/${PROJECT_NAME}:${tag}" \
        --platform managed \
        --region "$GCP_REGION" \
        --project "$GCP_PROJECT" \
        --allow-unauthenticated
    
    echo "✓ Service déployé sur Cloud Run"
}
```

## Multi-cloud

### Script multi-cloud

**Déploiement multi-cloud** :
```bash
#!/bin/bash
# Déploiement multi-cloud

deploy_multi_cloud() {
    local image_tag="$1"
    local clouds="${2:-aws,azure,gcp}"
    
    IFS=',' read -ra CLOUD_ARRAY <<< "$clouds"
    
    for cloud in "${CLOUD_ARRAY[@]}"; do
        echo "Déploiement sur $cloud..."
        
        case "$cloud" in
            aws)
                deploy_to_aws "$image_tag" &
                ;;
            azure)
                deploy_to_azure "$image_tag" &
                ;;
            gcp)
                deploy_to_gcp "$image_tag" &
                ;;
        esac
    done
    
    wait
    echo "✓ Tous les déploiements terminés"
}
```

## Conclusion

Le déploiement cloud offre la scalabilité et la fiabilité nécessaires pour les applications modernes. Chaque fournisseur cloud a ses avantages : choisissez selon vos besoins ou utilisez une stratégie multi-cloud.

Dans le chapitre suivant, nous explorerons Kubernetes et l'orchestration de conteneurs à grande échelle.

