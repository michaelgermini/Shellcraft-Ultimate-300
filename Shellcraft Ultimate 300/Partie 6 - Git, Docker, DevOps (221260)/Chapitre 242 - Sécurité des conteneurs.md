# Chapitre 242 - Sécurité des conteneurs

## Table des matières
- [Introduction](#introduction)
- [Bonnes pratiques de sécurité](#bonnes-pratiques-de-sécurité)
- [Scanning de vulnérabilités](#scanning-de-vulnérabilités)
- [Politiques de sécurité](#politiques-de-sécurité)
- [Conclusion](#conclusion)

## Introduction

La sécurité des conteneurs est essentielle pour protéger vos applications et données. Ce chapitre couvre les meilleures pratiques pour sécuriser vos conteneurs Docker et Kubernetes.

## Bonnes pratiques de sécurité

**Checklist de sécurité** :
```bash
#!/bin/bash
# Checklist de sécurité des conteneurs

security_checklist() {
    local dockerfile="$1"
    
    echo "=== Vérification de sécurité ==="
    
    # Vérifier l'utilisation d'un utilisateur non-root
    if grep -q "USER root" "$dockerfile"; then
        echo "⚠ Utilisez un utilisateur non-root"
    fi
    
    # Vérifier les secrets hardcodés
    if grep -qiE "(password|secret|key|token)" "$dockerfile"; then
        echo "⚠ Secrets potentiellement exposés"
    fi
    
    # Vérifier les images de base
    if grep -q "FROM.*:latest" "$dockerfile"; then
        echo "⚠ Évitez les tags 'latest'"
    fi
}
```

## Scanning de vulnérabilités

**Scanner de vulnérabilités** :
```bash
#!/bin/bash
# Scanning de vulnérabilités

scan_image() {
    local image="$1"
    
    # Utiliser Trivy
    if command -v trivy &> /dev/null; then
        trivy image "$image"
    fi
    
    # Utiliser Docker Scout
    docker scout cves "$image"
}
```

## Politiques de sécurité

**Politiques Kubernetes** :
```yaml
# security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

## Conclusion

La sécurité des conteneurs nécessite une attention constante. Utilisez des outils de scanning, suivez les bonnes pratiques, et implémentez des politiques de sécurité strictes.

