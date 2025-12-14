# Chapitre 186 - Curl GET POST PUT DELETE

## Table des matières
- [Introduction](#introduction)
- [Requêtes GET](#requêtes-get)
- [Requêtes POST](#requêtes-post)
- [Requêtes PUT](#requêtes-put)
- [Requêtes DELETE](#requêtes-delete)
- [Conclusion](#conclusion)

## Introduction

cURL est l'outil de ligne de commande le plus puissant pour interagir avec les APIs web. Ce chapitre couvre les méthodes HTTP fondamentales : GET, POST, PUT, et DELETE.

Imaginez cURL comme un navigateur web en ligne de commande : il peut faire tout ce qu'un navigateur fait, mais de manière automatisable et scriptable.

## Requêtes GET

**Requêtes GET de base** :
```bash
#!/bin/bash
# Requêtes GET avec cURL

# GET simple
simple_get() {
    curl https://api.example.com/users
}

# GET avec paramètres
get_with_params() {
    curl "https://api.example.com/users?page=1&limit=10"
}

# GET avec headers
get_with_headers() {
    curl -H "Accept: application/json" \
         -H "Authorization: Bearer $TOKEN" \
         https://api.example.com/users
}

# GET et sauvegarder la réponse
get_and_save() {
    curl -o response.json https://api.example.com/users
}

# GET avec affichage des headers
get_with_response_headers() {
    curl -i https://api.example.com/users
}
```

## Requêtes POST

**Requêtes POST** :
```bash
#!/bin/bash
# Requêtes POST avec cURL

# POST simple avec données JSON
post_json() {
    curl -X POST \
         -H "Content-Type: application/json" \
         -d '{"name":"John","email":"john@example.com"}' \
         https://api.example.com/users
}

# POST depuis un fichier
post_from_file() {
    curl -X POST \
         -H "Content-Type: application/json" \
         -d @data.json \
         https://api.example.com/users
}

# POST avec données form
post_form_data() {
    curl -X POST \
         -F "name=John" \
         -F "email=john@example.com" \
         https://api.example.com/users
}

# POST avec authentification
post_with_auth() {
    curl -X POST \
         -H "Authorization: Bearer $TOKEN" \
         -H "Content-Type: application/json" \
         -d '{"name":"John"}' \
         https://api.example.com/users
}
```

## Requêtes PUT

**Requêtes PUT** :
```bash
#!/bin/bash
# Requêtes PUT avec cURL

# PUT pour mettre à jour
put_update() {
    curl -X PUT \
         -H "Content-Type: application/json" \
         -d '{"name":"John Updated","email":"john.new@example.com"}' \
         https://api.example.com/users/123
}

# PUT partiel avec PATCH
patch_update() {
    curl -X PATCH \
         -H "Content-Type: application/json" \
         -d '{"email":"john.new@example.com"}' \
         https://api.example.com/users/123
}
```

## Requêtes DELETE

**Requêtes DELETE** :
```bash
#!/bin/bash
# Requêtes DELETE avec cURL

# DELETE simple
delete_resource() {
    curl -X DELETE \
         -H "Authorization: Bearer $TOKEN" \
         https://api.example.com/users/123
}

# DELETE avec confirmation
delete_with_confirmation() {
    read -p "Supprimer l'utilisateur 123? (y/N): " confirm
    if [[ "$confirm" =~ ^[Yy]$ ]]; then
        curl -X DELETE \
             -H "Authorization: Bearer $TOKEN" \
             https://api.example.com/users/123
    fi
}
```

## Conclusion

Les méthodes HTTP GET, POST, PUT, et DELETE sont les fondations de l'interaction avec les APIs REST. cURL les rend accessibles depuis la ligne de commande.

Dans le chapitre suivant, nous explorerons les headers et l'authentification, découvrant comment sécuriser et personnaliser vos requêtes.

