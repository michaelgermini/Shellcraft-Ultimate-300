# Chapitre 253 - Sécurité DevOps

## Table des matières
- [Introduction](#introduction)
- [DevSecOps](#devsecops)
- [Scanning de sécurité](#scanning-de-sécurité)
- [Gestion des secrets](#gestion-des-secrets)
- [Conclusion](#conclusion)

## Introduction

La sécurité DevOps (DevSecOps) intègre la sécurité dans chaque étape du cycle de développement.

## DevSecOps

**Intégration sécurité** :
```yaml
# .github/workflows/security.yml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run security scan
        run: |
          trivy fs .
          bandit -r .
```

## Scanning de sécurité

**Scanning automatisé** :
```bash
#!/bin/bash
# Scanning de sécurité

security_scan() {
    # Scan des dépendances
    npm audit
    pip-audit
    
    # Scan des conteneurs
    trivy image myapp:latest
    
    # Scan de l'infrastructure
    checkov -d .
}
```

## Gestion des secrets

**Gestion sécurisée** :
```bash
#!/bin/bash
# Gestion des secrets

# Utiliser Vault
store_secret() {
    vault kv put secret/myapp/api_key value="$API_KEY"
}

retrieve_secret() {
    vault kv get -field=value secret/myapp/api_key
}
```

## Conclusion

DevSecOps garantit que la sécurité est intégrée dès le début du développement.

