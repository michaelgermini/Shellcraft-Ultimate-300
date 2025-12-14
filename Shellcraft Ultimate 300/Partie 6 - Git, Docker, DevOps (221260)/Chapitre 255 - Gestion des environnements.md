# Chapitre 255 - Gestion des environnements

## Table des matières
- [Introduction](#introduction)
- [Environnements multiples](#environnements-multiples)
- [Promotion entre environnements](#promotion-entre-environnements)
- [Gestion de configuration](#gestion-de-configuration)
- [Conclusion](#conclusion)

## Introduction

La gestion des environnements permet de gérer développement, staging, et production de manière cohérente et automatisée.

## Environnements multiples

**Configuration multi-environnement** :
```yaml
# environments.yaml
environments:
  dev:
    replicas: 1
    resources:
      cpu: "100m"
      memory: "128Mi"
  
  staging:
    replicas: 2
    resources:
      cpu: "200m"
      memory: "256Mi"
  
  production:
    replicas: 5
    resources:
      cpu: "500m"
      memory: "512Mi"
```

## Promotion entre environnements

**Script de promotion** :
```bash
#!/bin/bash
# Promotion entre environnements

promote_to_staging() {
    local version="$1"
    
    deploy_to_environment "staging" "$version"
    run_smoke_tests "staging"
}

promote_to_production() {
    local version="$1"
    
    deploy_to_environment "production" "$version"
    monitor_deployment "production"
}
```

## Gestion de configuration

**Configuration par environnement** :
```bash
#!/bin/bash
# Gestion de configuration

load_environment_config() {
    local env="$1"
    source "config/${env}.env"
    
    # Appliquer la configuration
    apply_config "$env"
}
```

## Conclusion

La gestion des environnements garantit des déploiements cohérents et contrôlés.

