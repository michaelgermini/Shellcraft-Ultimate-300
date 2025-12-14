# Chapitre 212 - Streaming et Server-Sent Events

## Table des matières
- [Introduction](#introduction)
- [Server-Sent Events](#server-sent-events)
- [Streaming de données](#streaming-de-données)
- [Conclusion](#conclusion)

## Introduction

Le streaming et les Server-Sent Events permettent de recevoir des données en temps réel depuis les serveurs.

## Server-Sent Events

**Réception SSE** :
```bash
#!/bin/bash
# Server-Sent Events

receive_sse() {
    local url="$1"
    
    curl -N -H "Accept: text/event-stream" "$url" | \
    while IFS= read -r line; do
        if [[ "$line" =~ ^data: ]]; then
            echo "${line#data: }"
        fi
    done
}
```

## Streaming de données

**Streaming avec cURL** :
```bash
#!/bin/bash
# Streaming de données

stream_data() {
    local url="$1"
    
    curl -N "$url" | while IFS= read -r chunk; do
        process_chunk "$chunk"
    done
}
```

## Conclusion

Le streaming permet de traiter des données en temps réel de manière efficace.

