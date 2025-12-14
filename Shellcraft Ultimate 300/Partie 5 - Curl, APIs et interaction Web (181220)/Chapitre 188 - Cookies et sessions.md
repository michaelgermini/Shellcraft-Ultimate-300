# Chapitre 188 - Cookies et sessions

## Table des matières
- [Introduction](#introduction)
- [Gestion des cookies](#gestion-des-cookies)
- [Sessions web](#sessions-web)
- [Persistance de session](#persistance-de-session)
- [Conclusion](#conclusion)

## Introduction

Les cookies et les sessions permettent de maintenir l'état entre les requêtes HTTP, essentiel pour l'authentification et la personnalisation.

## Gestion des cookies

**Cookies avec cURL** :
```bash
#!/bin/bash
# Gestion des cookies

# Sauvegarder les cookies
save_cookies() {
    curl -c cookies.txt https://api.example.com/login \
         -d "username=user&password=pass"
}

# Utiliser les cookies sauvegardés
use_cookies() {
    curl -b cookies.txt https://api.example.com/data
}

# Envoyer des cookies personnalisés
send_custom_cookies() {
    curl -b "sessionid=abc123; csrftoken=xyz789" \
         https://api.example.com/data
}
```

## Sessions web

**Gestion de session** :
```bash
#!/bin/bash
# Gestion de session web

# Créer une session
create_session() {
    curl -c session.txt \
         -X POST https://api.example.com/login \
         -d "username=user&password=pass"
}

# Utiliser la session
use_session() {
    curl -b session.txt https://api.example.com/protected
}
```

## Persistance de session

**Script de session persistante** :
```bash
#!/bin/bash
# Session persistante

SESSION_FILE=".session"

maintain_session() {
    if [ ! -f "$SESSION_FILE" ]; then
        login_and_save_session
    fi
    
    # Vérifier si la session est valide
    if ! validate_session; then
        login_and_save_session
    fi
    
    # Utiliser la session
    curl -b "$SESSION_FILE" "$@"
}

login_and_save_session() {
    curl -c "$SESSION_FILE" \
         -X POST https://api.example.com/login \
         -d "username=$USERNAME&password=$PASSWORD"
}
```

## Conclusion

Les cookies et sessions sont essentiels pour maintenir l'état dans les applications web.

