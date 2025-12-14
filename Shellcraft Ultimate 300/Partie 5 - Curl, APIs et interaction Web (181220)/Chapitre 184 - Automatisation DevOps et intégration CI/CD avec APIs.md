# Chapitre 184 - Automatisation DevOps et intégration CI/CD avec APIs

> "Les APIs ne sont pas seulement des interfaces pour les applications - elles sont les tuyaux qui relient tous les outils DevOps en un orchestre symphonique d'automatisation." - Architecte DevOps

## Introduction : APIs comme infrastructure d'automatisation

Dans l'écosystème DevOps moderne, les APIs sont les interfaces invisibles qui relient les outils, automatisent les workflows, et orchestrent les déploiements. GitHub, Jenkins, Docker, Kubernetes, AWS, Azure - tous exposent des APIs riches qui permettent de construire des pipelines d'automatisation complets. Dans ce chapitre, nous explorerons comment les APIs transforment le DevOps d'une collection d'outils disparates en un système intégré et programmable.

Des pipelines CI/CD aux infrastructures as code, nous découvrirons comment les APIs deviennent les fondations de l'automatisation moderne.

## Section 1 : APIs Git et gestion du code source

### 1.1 Intégration GitHub API

```bash
#!/bin/bash
# Automatisation GitHub via API

# Configuration
GITHUB_TOKEN="${GITHUB_TOKEN:-your_token_here}"
GITHUB_API="https://api.github.com"
REPO_OWNER="myorg"
REPO_NAME="myapp"
LOG_FILE="github_automation.log"

# Fonction de logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Fonction d'appel API GitHub
github_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${GITHUB_API}${endpoint}"
    
    curl -s \
        -X "$method" \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Accept: application/vnd.github.v3+json" \
        ${data:+-d "$data"} \
        "$url"
}

# Fonction de gestion des PRs
manage_pull_requests() {
    log "=== GESTION DES PULL REQUESTS ==="
    
    # Récupération des PRs ouvertes
    local prs=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/pulls?state=open")
    
    echo "$prs" | jq -r '.[] | "\(.number): \(.title) by \(.user.login) [\(.base.ref) <- \(.head.ref)]"'
    
    # Analyse des PRs
    local pr_count=$(echo "$prs" | jq '. | length')
    log "PRs ouvertes: $pr_count"
    
    # Vérification des checks
    for pr_number in $(echo "$prs" | jq -r '.[].number'); do
        local checks=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/commits/$(echo "$prs" | jq -r ".[] | select(.number == $pr_number) | .head.sha")/check-runs")
        
        local failed_checks=$(echo "$checks" | jq '[.check_runs[] | select(.conclusion == "failure")] | length')
        local total_checks=$(echo "$checks" | jq '.check_runs | length')
        
        if [[ $failed_checks -gt 0 ]]; then
            log "PR #$pr_number: $failed_checks/$total_checks checks échoués"
            
            # Commentaire automatique sur la PR
            local comment="⚠️ **Checks échoués**: $failed_checks/$total_checks checks n'ont pas passé.
            
Vérifiez les détails des checks avant de merger."
            
            github_api_call "POST" "/repos/$REPO_OWNER/$REPO_NAME/issues/$pr_number/comments" "{\"body\": \"$comment\"}"
        fi
    done
}

# Fonction de création de release
create_github_release() {
    local tag_name="$1"
    local release_name="$2"
    local description="$3"
    local draft="${4:-false}"
    local prerelease="${5:-false}"
    
    log "Création de release: $tag_name"
    
    local release_data=$(cat <<EOF
{
    "tag_name": "$tag_name",
    "name": "$release_name",
    "body": "$description",
    "draft": $draft,
    "prerelease": $prerelease
}
EOF
)
    
    local response=$(github_api_call "POST" "/repos/$REPO_OWNER/$REPO_NAME/releases" "$release_data")
    
    if echo "$response" | jq -e '.id' > /dev/null 2>&1; then
        local release_id=$(echo "$response" | jq -r '.id')
        local release_url=$(echo "$response" | jq -r '.html_url')
        log "✓ Release créée: $release_url (ID: $release_id)"
        echo "$release_id"
    else
        log "✗ Échec de création de release: $(echo "$response" | jq -r '.message')"
        return 1
    fi
}

# Fonction d'upload d'assets de release
upload_release_asset() {
    local release_id="$1"
    local asset_path="$2"
    local asset_name="${3:-$(basename "$asset_path")}"
    
    log "Upload d'asset: $asset_name"
    
    local upload_url=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/releases/$release_id" | jq -r '.upload_url' | sed 's/{.*}//')
    
    local response=$(curl -s \
        -X POST \
        -H "Authorization: token $GITHUB_TOKEN" \
        -H "Content-Type: $(file -b --mime-type "$asset_path")" \
        --data-binary @"$asset_path" \
        "$upload_url?name=$asset_name")
    
    if echo "$response" | jq -e '.id' > /dev/null 2>&1; then
        log "✓ Asset uploadé: $asset_name"
    else
        log "✗ Échec d'upload: $(echo "$response" | jq -r '.message')"
    fi
}

# Fonction de gestion des issues
manage_issues() {
    log "=== GESTION DES ISSUES ==="
    
    # Issues ouvertes
    local issues=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/issues?state=open")
    
    echo "$issues" | jq -r '.[] | "\(.number): \(.title) [\(.labels[].name] // []) by \(.user.login)"'
    
    # Statistiques
    local bug_count=$(echo "$issues" | jq '[.[] | select(.labels[].name == "bug")] | length')
    local enhancement_count=$(echo "$issues" | jq '[.[] | select(.labels[].name == "enhancement")] | length')
    
    log "Issues - Bugs: $bug_count, Enhancements: $enhancement_count"
    
    # Création d'issues automatiques
    create_automated_issue() {
        local title="$1"
        local body="$2"
        local labels="$3"
        
        local issue_data=$(cat <<EOF
{
    "title": "$title",
    "body": "$body",
    "labels": [$labels]
}
EOF
)
        
        local response=$(github_api_call "POST" "/repos/$REPO_OWNER/$REPO_NAME/issues" "$issue_data")
        
        if echo "$response" | jq -e '.number' > /dev/null 2>&1; then
            log "✓ Issue créée: $(echo "$response" | jq -r '.html_url')"
        fi
    }
    
    # Exemple: Issue pour dependabot alerts
    local alerts=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/dependabot/alerts?state=open")
    local critical_alerts=$(echo "$alerts" | jq '[.[] | select(.security_advisory.severity == "critical")] | length')
    
    if [[ $critical_alerts -gt 0 ]]; then
        create_automated_issue \
            "🚨 Alertes de sécurité critiques Dependabot" \
            "Il y a $critical_alerts alertes de sécurité critiques nécessitant une attention immédiate." \
            '"security","critical"'
    fi
}

# Pipeline d'automatisation GitHub
github_automation_pipeline() {
    log "=== PIPELINE D'AUTOMATISATION GITHUB ==="
    
    # 1. Gestion des PRs
    manage_pull_requests
    
    # 2. Gestion des issues
    manage_issues
    
    # 3. Création de release automatique (si tag présent)
    if [[ -n "${CI_COMMIT_TAG:-}" ]]; then
        local release_description="Release automatique créée le $(date)"
        local release_id=$(create_github_release "$CI_COMMIT_TAG" "Release $CI_COMMIT_TAG" "$release_description")
        
        if [[ -n "$release_id" ]]; then
            # Upload d'artifacts
            if [[ -f "build/app.zip" ]]; then
                upload_release_asset "$release_id" "build/app.zip"
            fi
            
            if [[ -f "docs/api-docs.html" ]]; then
                upload_release_asset "$release_id" "docs/api-docs.html" "api-documentation.html"
            fi
        fi
    fi
    
    log "=== PIPELINE TERMINÉ ==="
}

# Fonction de surveillance des webhooks
monitor_webhooks() {
    log "=== SURVEILLANCE DES WEBHOOKS ==="
    
    # Récupération des webhooks configurés
    local webhooks=$(github_api_call "GET" "/repos/$REPO_OWNER/$REPO_NAME/hooks")
    
    echo "$webhooks" | jq -r '.[] | "\(.id): \(.name) -> \(.config.url) [\(.active)]"'
    
    # Test des webhooks
    for hook_id in $(echo "$webhooks" | jq -r '.[].id'); do
        local test_result=$(github_api_call "POST" "/repos/$REPO_OWNER/$REPO_NAME/hooks/$hook_id/tests")
        
        if [[ $(echo "$test_result" | jq '. | length') -gt 0 ]]; then
            log "✓ Webhook $hook_id: test réussi"
        else
            log "✗ Webhook $hook_id: test échoué"
        fi
    done
}

# Exécution des automatisations
github_automation_pipeline
monitor_webhooks
```

### 1.2 Intégration GitLab API

```bash
#!/bin/bash
# Automatisation GitLab via API

# Configuration
GITLAB_TOKEN="${GITLAB_TOKEN:-your_token_here}"
GITLAB_API="https://gitlab.com/api/v4"
PROJECT_ID="12345678"  # ID du projet GitLab
LOG_FILE="gitlab_automation.log"

# Fonction d'appel API GitLab
gitlab_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${GITLAB_API}${endpoint}"
    
    curl -s \
        -X "$method" \
        -H "PRIVATE-TOKEN: $GITLAB_TOKEN" \
        -H "Content-Type: application/json" \
        ${data:+-d "$data"} \
        "$url"
}

# Fonction de gestion des merge requests
manage_merge_requests() {
    log "=== GESTION DES MERGE REQUESTS ==="
    
    # MRs ouvertes
    local mrs=$(gitlab_api_call "GET" "/projects/$PROJECT_ID/merge_requests?state=opened")
    
    echo "$mrs" | jq -r '.[] | "\(.iid): \(.title) by \(.author.name) [\(.source_branch) -> \(.target_branch)]"'
    
    # Analyse des pipelines
    for mr_iid in $(echo "$mrs" | jq -r '.[].iid'); do
        local pipelines=$(gitlab_api_call "GET" "/projects/$PROJECT_ID/merge_requests/$mr_iid/pipelines")
        
        local failed_pipelines=$(echo "$pipelines" | jq '[.[] | select(.status == "failed")] | length')
        local total_pipelines=$(echo "$pipelines" | length)
        
        if [[ $failed_pipelines -gt 0 ]]; then
            log "MR !$mr_iid: $failed_pipelines/$total_pipelines pipelines échoués"
            
            # Commentaire automatique
            local comment="⚠️ **Pipelines échoués**: $failed_pipelines/$total_pipelines pipelines n'ont pas passé.
            
Vérifiez les logs des pipelines avant de merger."
            
            gitlab_api_call "POST" "/projects/$PROJECT_ID/merge_requests/$mr_iid/notes" "{\"body\": \"$comment\"}"
        fi
    done
}

# Fonction de gestion des pipelines CI/CD
manage_ci_pipelines() {
    log "=== GESTION DES PIPELINES CI/CD ==="
    
    # Pipelines récentes
    local pipelines=$(gitlab_api_call "GET" "/projects/$PROJECT_ID/pipelines?per_page=20")
    
    echo "$pipelines" | jq -r '.[] | "\(.id): \(.ref) - \(.status) (\(.created_at))"'
    
    # Analyse des échecs
    local failed_pipelines=$(echo "$pipelines" | jq '[.[] | select(.status == "failed")][0:5]')
    
    if [[ $(echo "$failed_pipelines" | jq '. | length') -gt 0 ]]; then
        log "Pipelines échoués récents:"
        echo "$failed_pipelines" | jq -r '.[] | "  - Pipeline \(.id): \(.ref) - \(.web_url)"'
        
        # Création d'issues pour les échecs critiques
        echo "$failed_pipelines" | jq -r '.[] | select(.ref == "main" or .ref == "master") | @base64' | while read -r pipeline_data; do
            local pipeline=$(echo "$pipeline_data" | base64 -d)
            local pipeline_id=$(echo "$pipeline" | jq -r '.id')
            local ref=$(echo "$pipeline" | jq -r '.ref')
            
            local issue_data=$(cat <<EOF
{
    "title": "🚨 Pipeline échoué sur branche $ref",
    "description": "Le pipeline $pipeline_id a échoué sur la branche $ref.\n\n[Voir les détails]($(echo "$pipeline" | jq -r '.web_url'))",
    "labels": ["bug", "ci-failure", "urgent"]
}
EOF
)
            
            gitlab_api_call "POST" "/projects/$PROJECT_ID/issues" "$issue_data"
        done
    fi
}

# Fonction de déploiement automatique
automated_deployment() {
    local environment="$1"
    local ref="${2:-main}"
    
    log "=== DÉPLOIEMENT AUTOMATIQUE: $environment ==="
    
    # Vérification du pipeline réussi le plus récent
    local latest_pipeline=$(gitlab_api_call "GET" "/projects/$PROJECT_ID/pipelines?ref=$ref&status=success" | jq '.[0]')
    
    if [[ -z "$latest_pipeline" || "$latest_pipeline" == "null" ]]; then
        log "✗ Aucun pipeline réussi trouvé pour $ref"
        return 1
    fi
    
    local pipeline_id=$(echo "$latest_pipeline" | jq -r '.id')
    local commit_sha=$(echo "$latest_pipeline" | jq -r '.sha')
    
    log "Pipeline réussi trouvé: $pipeline_id (commit: $commit_sha)"
    
    # Création d'un déploiement
    local deploy_data=$(cat <<EOF
{
    "environment": "$environment",
    "sha": "$commit_sha",
    "ref": "$ref",
    "tag": false
}
EOF
)
    
    local deployment=$(gitlab_api_call "POST" "/projects/$PROJECT_ID/deployments" "$deploy_data")
    
    if echo "$deployment" | jq -e '.id' > /dev/null 2>&1; then
        local deploy_id=$(echo "$deployment" | jq -r '.id')
        log "✓ Déploiement créé: $deploy_id"
        
        # Marquage du déploiement comme réussi (simulation)
        sleep 5  # Simulation du temps de déploiement
        
        gitlab_api_call "PUT" "/projects/$PROJECT_ID/deployments/$deploy_id" '{"status": "success"}'
        log "✓ Déploiement marqué comme réussi"
        
    else
        log "✗ Échec de création du déploiement: $(echo "$deployment" | jq -r '.message')"
    fi
}

# Fonction de gestion des environnements
manage_environments() {
    log "=== GESTION DES ENVIRONNEMENTS ==="
    
    # Liste des environnements
    local environments=$(gitlab_api_call "GET" "/projects/$PROJECT_ID/environments")
    
    echo "$environments" | jq -r '.[] | "\(.id): \(.name) - \(.state)"'
    
    # Nettoyage des environnements obsolètes
    local obsolete_envs=$(echo "$environments" | jq '[.[] | select(.state == "stopped" and (.updated_at | fromdate < (now - 604800)))]')
    
    if [[ $(echo "$obsolete_envs" | jq '. | length') -gt 0 ]]; then
        log "Environnements obsolètes à nettoyer: $(echo "$obsolete_envs" | jq '. | length')"
        
        echo "$obsolete_envs" | jq -r '.[].id' | while read -r env_id; do
            gitlab_api_call "DELETE" "/projects/$PROJECT_ID/environments/$env_id"
            log "✓ Environnement $env_id supprimé"
        done
    fi
}

# Pipeline d'automatisation GitLab
gitlab_automation_pipeline() {
    log "=== PIPELINE D'AUTOMATISATION GITLAB ==="
    
    # 1. Gestion des merge requests
    manage_merge_requests
    
    # 2. Gestion des pipelines CI/CD
    manage_ci_pipelines
    
    # 3. Gestion des environnements
    manage_environments
    
    # 4. Déploiement automatique (si conditions remplies)
    if [[ "${AUTO_DEPLOY:-false}" == "true" ]]; then
        automated_deployment "staging" "develop"
        automated_deployment "production" "main"
    fi
    
    log "=== PIPELINE TERMINÉ ==="
}

# Exécution
gitlab_automation_pipeline
```

## Section 2 : APIs Docker et orchestration de conteneurs

### 2.1 API Docker Engine

```bash
#!/bin/bash
# Automatisation Docker via API

# Configuration
DOCKER_SOCKET="${DOCKER_SOCKET:-/var/run/docker.sock}"
DOCKER_API_BASE="http://localhost"
if [[ -n "$DOCKER_HOST" ]]; then
    DOCKER_API_BASE="$DOCKER_HOST"
fi

# Fonction d'appel API Docker
docker_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    curl -s \
        --unix-socket "$DOCKER_SOCKET" \
        -X "$method" \
        -H "Content-Type: application/json" \
        ${data:+-d "$data"} \
        "${DOCKER_API_BASE}/v1.41${endpoint}"
}

# Fonction de gestion des conteneurs
manage_containers() {
    log "=== GESTION DES CONTENEURS ==="
    
    # Liste des conteneurs
    local containers=$(docker_api_call "GET" "/containers/json?all=true")
    
    echo "$containers" | jq -r '.[] | "\(.Id[0:12]): \(.Names[0]) - \(.State) (\(.Status))"'
    
    # Statistiques des conteneurs
    local running=$(echo "$containers" | jq '[.[] | select(.State == "running")] | length')
    local stopped=$(echo "$containers" | jq '[.[] | select(.State != "running")] | length')
    
    log "Conteneurs - Running: $running, Stopped: $stopped"
    
    # Nettoyage automatique
    local dangling_containers=$(echo "$containers" | jq '[.[] | select(.State != "running" and (.Created | fromdate < (now - 86400)))]')
    
    if [[ $(echo "$dangling_containers" | jq '. | length') -gt 0 ]]; then
        log "Conteneurs arrêtés depuis plus de 24h: $(echo "$dangling_containers" | jq '. | length')"
        
        echo "$dangling_containers" | jq -r '.[].Id' | while read -r container_id; do
            docker_api_call "DELETE" "/containers/$container_id"
            log "✓ Conteneur supprimé: ${container_id:0:12}"
        done
    fi
}

# Fonction de gestion des images
manage_images() {
    log "=== GESTION DES IMAGES ==="
    
    # Liste des images
    local images=$(docker_api_call "GET" "/images/json")
    
    echo "$images" | jq -r '.[] | "\(.Id[7:19]): \(.RepoTags[0] // "none") - \(.Size | . / 1024 / 1024 | round)MB"'
    
    # Images dangling (sans tag)
    local dangling_images=$(docker_api_call "GET" "/images/json?filters={\"dangling\":[\"true\"]}")
    
    if [[ $(echo "$dangling_images" | jq '. | length') -gt 0 ]]; then
        log "Images dangling: $(echo "$dangling_images" | jq '. | length')"
        
        echo "$dangling_images" | jq -r '.[].Id' | while read -r image_id; do
            docker_api_call "DELETE" "/images/$image_id"
            log "✓ Image dangling supprimée: ${image_id:7:12}"
        done
    fi
    
    # Build d'image via API
    build_image() {
        local dockerfile_path="$1"
        local tag="$2"
        local context_path="${3:-$(dirname "$dockerfile_path")}"
        
        log "Construction d'image: $tag"
        
        # Création d'une archive tar du contexte
        local tar_file="/tmp/docker_context_$(date +%s).tar"
        tar -cf "$tar_file" -C "$context_path" .
        
        # Build via API
        local build_response=$(curl -s \
            --unix-socket "$DOCKER_SOCKET" \
            -X POST \
            -H "Content-Type: application/tar" \
            --data-binary @"$tar_file" \
            "${DOCKER_API_BASE}/v1.41/build?t=$tag&dockerfile=$(basename "$dockerfile_path")")
        
        # Nettoyage
        rm "$tar_file"
        
        # Vérification du succès
        if echo "$build_response" | grep -q '"stream":"Successfully tagged"'; then
            log "✓ Image construite: $tag"
        else
            log "✗ Échec de construction: $(echo "$build_response" | jq -r '.error' 2>/dev/null || echo 'Erreur inconnue')"
        fi
    }
}

# Fonction de déploiement de stack
deploy_stack() {
    local compose_file="$1"
    local stack_name="$2"
    
    log "Déploiement de stack: $stack_name"
    
    # Lecture du fichier compose
    local compose_content=$(cat "$compose_file")
    
    # Conversion en format API Docker (simplifié)
    # Note: L'API Docker Compose nécessite docker-compose ou docker stack deploy
    
    # Simulation avec docker stack deploy
    if command -v docker >/dev/null 2>&1; then
        if docker stack deploy -c "$compose_file" "$stack_name"; then
            log "✓ Stack déployé: $stack_name"
            
            # Vérification des services
            sleep 5
            local services=$(docker stack services "$stack_name")
            log "Services actifs: $(echo "$services" | wc -l)"
            
        else
            log "✗ Échec de déploiement de stack"
        fi
    fi
}

# Fonction de monitoring des conteneurs
monitor_containers() {
    log "=== MONITORING DES CONTENEURS ==="
    
    # Statistiques des conteneurs
    local containers=$(docker_api_call "GET" "/containers/json")
    
    for container_id in $(echo "$containers" | jq -r '.[].Id'); do
        local stats=$(docker_api_call "GET" "/containers/$container_id/stats?stream=false")
        
        local name=$(echo "$stats" | jq -r '.name[1:]')
        local cpu_percent=$(echo "$stats" | jq -r '(.cpu_stats.cpu_usage.total_usage / .cpu_stats.system_cpu_usage * 100) | round')
        local memory_usage=$(echo "$stats" | jq -r '(.memory_stats.usage / 1024 / 1024) | round')
        local memory_limit=$(echo "$stats" | jq -r '(.memory_stats.limit / 1024 / 1024) | round')
        
        printf "Conteneur: %s | CPU: %d%% | Mémoire: %dMB/%dMB\n" \
            "$name" "$cpu_percent" "$memory_usage" "$memory_limit"
        
        # Alertes
        if [[ $cpu_percent -gt 80 ]]; then
            log "⚠️ CPU élevé pour $name: $cpu_percent%"
        fi
        
        if [[ $memory_usage -gt $((memory_limit * 80 / 100)) ]]; then
            log "⚠️ Mémoire élevée pour $name: ${memory_usage}MB/${memory_limit}MB"
        fi
    done
}

# Pipeline d'automatisation Docker
docker_automation_pipeline() {
    log "=== PIPELINE D'AUTOMATISATION DOCKER ==="
    
    # 1. Gestion des conteneurs
    manage_containers
    
    # 2. Gestion des images
    manage_images
    
    # 3. Monitoring
    monitor_containers
    
    # 4. Déploiement de stack (si fichier compose présent)
    if [[ -f "docker-compose.yml" ]]; then
        deploy_stack "docker-compose.yml" "myapp"
    fi
    
    log "=== PIPELINE TERMINÉ ==="
}

# Fonction de récupération des logs
get_container_logs() {
    local container_name="$1"
    local lines="${2:-100}"
    
    log "Récupération des logs: $container_name (dernières $lines lignes)"
    
    local logs=$(docker_api_call "GET" "/containers/$container_name/logs?stdout=true&stderr=true&tail=$lines")
    
    # Les logs sont retournés encodés, décodage nécessaire
    echo "$logs" | base64 -d 2>/dev/null || echo "$logs"
}

# Exécution des automatisations Docker
docker_automation_pipeline

# Exemple de récupération de logs
if [[ -n "${CONTAINER_TO_CHECK:-}" ]]; then
    get_container_logs "$CONTAINER_TO_CHECK" 50
fi
```

### 2.2 API Kubernetes

```bash
#!/bin/bash
# Automatisation Kubernetes via API

# Configuration
KUBECONFIG="${KUBECONFIG:-$HOME/.kube/config}"
KUBERNETES_API_SERVER="${KUBERNETES_API_SERVER:-https://kubernetes.docker.internal:6443}"
NAMESPACE="${NAMESPACE:-default}"

# Fonction d'appel API Kubernetes
k8s_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    curl -s \
        --cacert <(kubectl config view --raw -o json | jq -r '.clusters[0].cluster."certificate-authority-data"' | base64 -d) \
        --cert <(kubectl config view --raw -o json | jq -r '.users[0].user."client-certificate-data"' | base64 -d) \
        --key <(kubectl config view --raw -o json | jq -r '.users[0].user."client-key-data"' | base64 -d) \
        -X "$method" \
        -H "Content-Type: application/json" \
        ${data:+-d "$data"} \
        "${KUBERNETES_API_SERVER}/api/v1${endpoint}"
}

# Fonction de gestion des pods
manage_pods() {
    log "=== GESTION DES PODS ==="
    
    # Liste des pods
    local pods=$(k8s_api_call "GET" "/namespaces/$NAMESPACE/pods")
    
    echo "$pods" | jq -r '.items[] | "\(.metadata.name): \(.status.phase) (\(.status.podIP // "N/A"}))"'
    
    # Analyse des statuts
    local running=$(echo "$pods" | jq '[.items[] | select(.status.phase == "Running")] | length')
    local pending=$(echo "$pods" | jq '[.items[] | select(.status.phase == "Pending")] | length')
    local failed=$(echo "$pods" | jq '[.items[] | select(.status.phase == "Failed")] | length')
    
    log "Pods - Running: $running, Pending: $pending, Failed: $failed"
    
    # Redémarrage des pods échoués
    if [[ $failed -gt 0 ]]; then
        log "Redémarrage des pods échoués..."
        
        echo "$pods" | jq -r '.items[] | select(.status.phase == "Failed") | .metadata.name' | while read -r pod_name; do
            log "Suppression du pod échoué: $pod_name"
            k8s_api_call "DELETE" "/namespaces/$NAMESPACE/pods/$pod_name"
        done
    fi
}

# Fonction de déploiement d'application
deploy_application() {
    local app_name="$1"
    local image="$2"
    local replicas="${3:-1}"
    
    log "Déploiement d'application: $app_name"
    
    # Création du déploiement
    local deployment_data=$(cat <<EOF
{
    "apiVersion": "apps/v1",
    "kind": "Deployment",
    "metadata": {
        "name": "$app_name",
        "namespace": "$NAMESPACE"
    },
    "spec": {
        "replicas": $replicas,
        "selector": {
            "matchLabels": {
                "app": "$app_name"
            }
        },
        "template": {
            "metadata": {
                "labels": {
                    "app": "$app_name"
                }
            },
            "spec": {
                "containers": [{
                    "name": "$app_name",
                    "image": "$image",
                    "ports": [{
                        "containerPort": 80
                    }]
                }]
            }
        }
    }
}
EOF
)
    
    local response=$(curl -s \
        --cacert <(kubectl config view --raw -o json | jq -r '.clusters[0].cluster."certificate-authority-data"' | base64 -d) \
        --cert <(kubectl config view --raw -o json | jq -r '.users[0].user."client-certificate-data"' | base64 -d) \
        --key <(kubectl config view --raw -o json | jq -r '.users[0].user."client-key-data"' | base64 -d) \
        -X POST \
        -H "Content-Type: application/json" \
        -d "$deployment_data" \
        "${KUBERNETES_API_SERVER}/apis/apps/v1/namespaces/$NAMESPACE/deployments")
    
    if echo "$response" | jq -e '.metadata.name' > /dev/null 2>&1; then
        log "✓ Déploiement créé: $app_name"
        
        # Création du service
        local service_data=$(cat <<EOF
{
    "apiVersion": "v1",
    "kind": "Service",
    "metadata": {
        "name": "${app_name}-service",
        "namespace": "$NAMESPACE"
    },
    "spec": {
        "selector": {
            "app": "$app_name"
        },
        "ports": [{
            "port": 80,
            "targetPort": 80
        }],
        "type": "ClusterIP"
    }
}
EOF
)
        
        k8s_api_call "POST" "/namespaces/$NAMESPACE/services" "$service_data"
        log "✓ Service créé: ${app_name}-service"
        
    else
        log "✗ Échec de déploiement: $(echo "$response" | jq -r '.message')"
    fi
}

# Fonction de monitoring des ressources
monitor_resources() {
    log "=== MONITORING DES RESSOURCES K8S ==="
    
    # Utilisation des nœuds
    local nodes=$(k8s_api_call "GET" "/nodes")
    
    echo "$nodes" | jq -r '.items[] | "\(.metadata.name): \(.status.conditions[] | select(.type == "Ready") | .status)"'
    
    # Utilisation des pods par namespace
    local pod_usage=$(k8s_api_call "GET" "/pods")
    
    echo "$pod_usage" | jq 'group_by(.metadata.namespace) | map({
        namespace: .[0].metadata.namespace,
        total_pods: length,
        running_pods: map(select(.status.phase == "Running")) | length
    }) | sort_by(.total_pods) | reverse | .[0:5]' | jq -r '.[] | "\(.namespace): \(.running_pods)/\(.total_pods) pods"'
    
    # Métriques des conteneurs (nécessite metrics-server)
    if kubectl top pods >/dev/null 2>&1; then
        log "Métriques des pods:"
        kubectl top pods --all-namespaces | head -10
    fi
}

# Fonction de sauvegarde des configurations
backup_configurations() {
    local backup_dir="${1:-k8s_backup_$(date +%Y%m%d_%H%M%S)}"
    
    log "Sauvegarde des configurations dans: $backup_dir"
    mkdir -p "$backup_dir"
    
    # Sauvegarde des déploiements
    local deployments=$(k8s_api_call "GET" "/apis/apps/v1/namespaces/$NAMESPACE/deployments")
    
    echo "$deployments" | jq -r '.items[] | @base64' | while read -r deployment_data; do
        local deployment=$(echo "$deployment_data" | base64 -d)
        local name=$(echo "$deployment" | jq -r '.metadata.name')
        
        echo "$deployment" | jq '.' > "$backup_dir/deployment_${name}.json"
    done
    
    # Sauvegarde des services
    local services=$(k8s_api_call "GET" "/namespaces/$NAMESPACE/services")
    
    echo "$services" | jq -r '.items[] | @base64' | while read -r service_data; do
        local service=$(echo "$service_data" | base64 -d)
        local name=$(echo "$service" | jq -r '.metadata.name')
        
        echo "$service" | jq '.' > "$backup_dir/service_${name}.json"
    done
    
    log "✓ Configurations sauvegardées: $(ls "$backup_dir"/*.json | wc -l) fichiers"
    
    # Création d'archive
    tar -czf "${backup_dir}.tar.gz" "$backup_dir"
    rm -rf "$backup_dir"
    
    log "✓ Archive créée: ${backup_dir}.tar.gz"
}

# Pipeline d'automatisation Kubernetes
k8s_automation_pipeline() {
    log "=== PIPELINE D'AUTOMATISATION KUBERNETES ==="
    
    # 1. Gestion des pods
    manage_pods
    
    # 2. Monitoring des ressources
    monitor_resources
    
    # 3. Déploiement d'application (si demandé)
    if [[ -n "${APP_NAME:-}" && -n "${APP_IMAGE:-}" ]]; then
        deploy_application "$APP_NAME" "$APP_IMAGE" "${APP_REPLICAS:-1}"
    fi
    
    # 4. Sauvegarde des configurations
    if [[ "${BACKUP_CONFIG:-false}" == "true" ]]; then
        backup_configurations
    fi
    
    log "=== PIPELINE TERMINÉ ==="
}

# Exécution du pipeline
k8s_automation_pipeline
```

## Section 3 : Intégration CI/CD avec APIs

### 3.1 Pipeline Jenkins via API

```bash
#!/bin/bash
# Automatisation Jenkins via API

# Configuration
JENKINS_URL="${JENKINS_URL:-http://localhost:8080}"
JENKINS_USER="${JENKINS_USER:-admin}"
JENKINS_TOKEN="${JENKINS_TOKEN:-your_token_here}"

# Fonction d'appel API Jenkins
jenkins_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local crumb
    if [[ "$method" != "GET" ]]; then
        # Récupération du crumb pour CSRF
        crumb=$(curl -s \
            -u "$JENKINS_USER:$JENKINS_TOKEN" \
            "$JENKINS_URL/crumbIssuer/api/json" | jq -r '.crumb')
    fi
    
    curl -s \
        -u "$JENKINS_USER:$JENKINS_TOKEN" \
        ${crumb:+-H "Jenkins-Crumb: $crumb"} \
        -X "$method" \
        ${data:+-d "$data"} \
        "$JENKINS_URL$endpoint"
}

# Fonction de gestion des jobs
manage_jobs() {
    log "=== GESTION DES JOBS JENKINS ==="
    
    # Liste des jobs
    local jobs=$(jenkins_api_call "GET" "/api/json?tree=jobs[name,color,lastBuild[number,result]]")
    
    echo "$jobs" | jq -r '.jobs[] | "\(.name): \(.color) - Build \(.lastBuild.number // "N/A"): \(.lastBuild.result // "N/A")"'
    
    # Analyse des statuts
    local success_jobs=$(echo "$jobs" | jq '[.jobs[] | select(.color | contains("blue"))] | length')
    local failed_jobs=$(echo "$jobs" | jq '[.jobs[] | select(.color | contains("red"))] | length')
    
    log "Jobs - Succès: $success_jobs, Échec: $failed_jobs"
    
    # Redémarrage des jobs échoués
    echo "$jobs" | jq -r '.jobs[] | select(.color | contains("red")) | .name' | while read -r job_name; do
        log "Redémarrage du job échoué: $job_name"
        jenkins_api_call "POST" "/job/$job_name/build"
    done
}

# Fonction de création de job
create_jenkins_job() {
    local job_name="$1"
    local job_config="$2"
    
    log "Création du job: $job_name"
    
    local response=$(jenkins_api_call "POST" "/createItem?name=$job_name" "$job_config")
    
    if [[ -z "$response" ]]; then
        log "✓ Job créé: $job_name"
    else
        log "✗ Échec de création: $response"
    fi
}

# Fonction de déclenchement de build
trigger_build() {
    local job_name="$1"
    local parameters="$2"
    
    log "Déclenchement de build: $job_name"
    
    if [[ -n "$parameters" ]]; then
        # Build paramétré
        jenkins_api_call "POST" "/job/$job_name/buildWithParameters" "$parameters"
    else
        # Build simple
        jenkins_api_call "POST" "/job/$job_name/build"
    fi
    
    # Attente du début du build
    sleep 5
    
    # Récupération du numéro du dernier build
    local build_info=$(jenkins_api_call "GET" "/job/$job_name/lastBuild/api/json?tree=number,building,result")
    local build_number=$(echo "$build_info" | jq -r '.number')
    
    log "Build lancé: #$build_number"
    
    # Surveillance du build
    while true; do
        local status=$(jenkins_api_call "GET" "/job/$job_name/$build_number/api/json?tree=building,result")
        local is_building=$(echo "$status" | jq -r '.building')
        local result=$(echo "$status" | jq -r '.result')
        
        if [[ "$is_building" == "false" ]]; then
            if [[ "$result" == "SUCCESS" ]]; then
                log "✓ Build #$build_number terminé avec succès"
            else
                log "✗ Build #$build_number échoué: $result"
            fi
            break
        fi
        
        sleep 10
    done
}

# Fonction de récupération des artifacts
download_artifacts() {
    local job_name="$1"
    local build_number="$2"
    local artifact_pattern="$3"
    local output_dir="${4:-artifacts}"
    
    mkdir -p "$output_dir"
    
    log "Téléchargement des artifacts: $job_name #$build_number"
    
    # Liste des artifacts
    local artifacts=$(jenkins_api_call "GET" "/job/$job_name/$build_number/api/json?tree=artifacts[fileName,relativePath]")
    
    echo "$artifacts" | jq -r '.artifacts[] | select(.fileName | test("'$artifact_pattern'")) | .relativePath' | while read -r artifact_path; do
        local filename=$(basename "$artifact_path")
        log "Téléchargement: $filename"
        
        curl -s \
            -u "$JENKINS_USER:$JENKINS_TOKEN" \
            -o "$output_dir/$filename" \
            "$JENKINS_URL/job/$job_name/$build_number/artifact/$artifact_path"
        
        if [[ -f "$output_dir/$filename" ]]; then
            log "✓ Artifact téléchargé: $filename"
        fi
    done
}

# Pipeline d'automatisation Jenkins
jenkins_automation_pipeline() {
    log "=== PIPELINE D'AUTOMATISATION JENKINS ==="
    
    # 1. Gestion des jobs
    manage_jobs
    
    # 2. Création de job (si demandé)
    if [[ -n "${NEW_JOB_NAME:-}" && -f "${NEW_JOB_CONFIG:-}" ]]; then
        create_jenkins_job "$NEW_JOB_NAME" "$(cat "$NEW_JOB_CONFIG")"
    fi
    
    # 3. Déclenchement de builds
    if [[ -n "${BUILD_JOB:-}" ]]; then
        trigger_build "$BUILD_JOB" "${BUILD_PARAMETERS:-}"
        
        # Téléchargement des artifacts si build réussi
        if [[ -n "${ARTIFACT_PATTERN:-}" ]]; then
            local last_build=$(jenkins_api_call "GET" "/job/$BUILD_JOB/lastBuild/api/json?tree=number,result")
            local build_result=$(echo "$last_build" | jq -r '.result')
            
            if [[ "$build_result" == "SUCCESS" ]]; then
                local build_number=$(echo "$last_build" | jq -r '.number')
                download_artifacts "$BUILD_JOB" "$build_number" "$ARTIFACT_PATTERN"
            fi
        fi
    fi
    
    log "=== PIPELINE TERMINÉ ==="
}

# Exécution du pipeline
jenkins_automation_pipeline
```

## Conclusion : APIs comme fondation de l'automatisation DevOps

Les APIs transforment le DevOps d'une collection d'outils disparates en un écosystème intégré et programmable. De GitHub à Kubernetes, en passant par Jenkins et Docker, les APIs deviennent les interfaces invisibles qui relient et orchestrent tous les outils de la chaîne DevOps.

Dans le prochain chapitre, nous explorerons l'infrastructure as code avec Terraform et les outils de provisionnement cloud, complétant ainsi notre panorama de l'automatisation moderne.

---

**Exercice pratique :** Créez un système d'orchestration DevOps complet qui :
1. Surveille les repositories Git pour les changements
2. Déclenche automatiquement des builds CI/CD
3. Provisionne des environnements de test
4. Déploie des applications conteneurisées
5. Effectue des tests d'intégration automatisés
6. Génère des rapports de déploiement

**Challenge avancé :** Développez un tableau de bord DevOps unifié qui agrège :
- Métriques de performance des applications
- Statuts des pipelines CI/CD
- État des déploiements
- Alertes de sécurité
- Métriques d'infrastructure
- Indicateurs de qualité du code

**Réflexion :** Comment les APIs changent-elles fondamentalement l'approche DevOps, et quels sont les impacts sur la productivité et la fiabilité des équipes de développement ?

