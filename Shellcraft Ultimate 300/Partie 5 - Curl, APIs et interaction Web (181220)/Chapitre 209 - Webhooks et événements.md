# Chapitre 209 - Webhooks et événements

## Table des matières
- [Introduction](#introduction)
- [Réception de webhooks](#réception-de-webhooks)
- [Envoi de webhooks](#envoi-de-webhooks)
- [Validation de webhooks](#validation-de-webhooks)
- [Conclusion](#conclusion)

## Introduction

Les webhooks permettent la communication en temps réel entre services via des callbacks HTTP.

## Réception de webhooks

**Serveur webhook simple** :
```bash
#!/bin/bash
# Serveur webhook avec netcat

webhook_server() {
    local port="${1:-8080}"
    
    while true; do
        nc -l "$port" | while read -r line; do
            echo "[$(date)] Webhook reçu: $line"
            process_webhook "$line"
        done
    done
}
```

## Envoi de webhooks

**Envoi de webhooks** :
```bash
#!/bin/bash
# Envoi de webhooks

send_webhook() {
    local url="$1"
    local payload="$2"
    local secret="${3:-}"
    
    local signature=""
    if [ -n "$secret" ]; then
        signature=$(echo -n "$payload" | openssl dgst -sha256 -hmac "$secret" | cut -d' ' -f2)
    fi
    
    curl -X POST "$url" \
         -H "Content-Type: application/json" \
         ${signature:+-H "X-Signature: $signature"} \
         -d "$payload"
}
```

## Validation de webhooks

**Validation de signature** :
```bash
#!/bin/bash
# Validation de webhook

validate_webhook() {
    local payload="$1"
    local signature="$2"
    local secret="$3"
    
    local expected=$(echo -n "$payload" | openssl dgst -sha256 -hmac "$secret" | cut -d' ' -f2)
    
    if [ "$signature" == "$expected" ]; then
        return 0
    else
        return 1
    fi
}
```

## Conclusion

Les webhooks permettent une communication en temps réel entre services de manière efficace.

