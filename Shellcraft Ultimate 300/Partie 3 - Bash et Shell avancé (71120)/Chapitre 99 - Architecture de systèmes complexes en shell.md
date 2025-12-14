# Chapitre 99 - Architecture de systèmes complexes en shell

> "L'architecture n'est pas un ensemble de composants : c'est l'harmonie invisible qui transforme des éléments disparates en un tout cohérent, où chaque partie sert l'ensemble avec une précision d'horloger." - Architecte Shell Visionnaire

## Introduction : L'harmonie architecturale

Imaginez-vous architecte d'une cathédrale numérique : chaque pattern de conception est une colonne, chaque framework un arc-boutant, chaque système un vitrail coloré. L'architecture de systèmes complexes en shell ne consiste pas à empiler des fonctionnalités, mais à orchestrer une symphonie où chaque composant joue sa partition dans un ensemble parfaitement coordonné.

Dans ce chapitre, nous construirons les cathédrales de l'automatisation : architectures multicouches, systèmes distribués auto-gérants, frameworks d'entreprise évolutifs capables de gérer les défis les plus complexes avec élégance et robustesse.

## Section 1 : Architectures multicouches

### 1.1 Framework d'architecture en couches

Système modulaire avec séparation claire des responsabilités par couches :

```bash
#!/bin/bash

# Framework d'architecture en couches
echo "=== Framework d'architecture en couches ==="

# Layered Architecture Framework
LayeredArchitecture() {
    local self="$1"
    
    declare -A $self._layers
    declare -A $self._layer_dependencies
    declare -A $self._layer_interfaces
    declare -A $self._communication_bus
    
    # Définition d'une couche
    $self.define_layer() {
        local layer_name="$1"
        local layer_type="$2"  # presentation, business, data, infrastructure
        local responsibilities="$3"
        local interfaces="$4"
        
        $self._layers["${layer_name}_type"]="$layer_type"
        $self._layers["${layer_name}_responsibilities"]="$responsibilities"
        $self._layers["${layer_name}_interfaces"]="$interfaces"
        $self._layers["${layer_name}_status"]="defined"
        
        echo "✓ Couche définie: $layer_name ($layer_type)"
    }
    
    # Définition des dépendances entre couches
    $self.define_layer_dependency() {
        local from_layer="$1"
        local to_layer="$2"
        local dependency_type="$3"  # uses, implements, extends
        
        $self._layer_dependencies["${from_layer}_to_${to_layer}"]="$dependency_type"
        
        echo "✓ Dépendance définie: $from_layer -> $to_layer ($dependency_type)"
    }
    
    # Enregistrement d'une interface de couche
    $self.register_layer_interface() {
        local layer_name="$1"
        local interface_name="$2"
        local interface_contract="$3"
        
        $self._layer_interfaces["${layer_name}_${interface_name}"]="$interface_contract"
        
        echo "✓ Interface enregistrée: $layer_name.$interface_name"
    }
    
    # Communication inter-couches via bus
    $self.send_layer_message() {
        local from_layer="$1"
        local to_layer="$2"
        local message_type="$3"
        shift 3
        local -a message_data=("$@")
        
        # Validation de la dépendance
        local dependency="${$self._layer_dependencies[${from_layer}_to_${to_layer}]}"
        if [[ -z "$dependency" ]]; then
            echo "❌ Communication non autorisée: $from_layer -> $to_layer" >&2
            return 1
        fi
        
        local message_id="$(date +%s)_$$"
        local message_record="$message_id:$from_layer:$to_layer:$message_type:${message_data[*]}"
        
        $self._communication_bus["$message_id"]="$message_record"
        
        echo "📨 Message envoyé: $from_layer -> $to_layer ($message_type)"
        
        # Traitement du message par la couche destinataire
        $self._process_layer_message "$to_layer" "$message_type" "${message_data[@]}"
    }
    
    # Traitement d'un message par une couche
    $self._process_layer_message() {
        local layer_name="$1"
        local message_type="$2"
        shift 2
        local -a message_data=("$@")
        
        local layer_type="${$self._layers[${layer_name}_type]}"
        
        echo "Traitement dans $layer_name ($layer_type): $message_type"
        
        # Routage selon le type de couche et de message
        case "$layer_type" in
            presentation)
                $self._handle_presentation_message "$layer_name" "$message_type" "${message_data[@]}"
                ;;
                
            business)
                $self._handle_business_message "$layer_name" "$message_type" "${message_data[@]}"
                ;;
                
            data)
                $self._handle_data_message "$layer_name" "$message_type" "${message_data[@]}"
                ;;
                
            infrastructure)
                $self._handle_infrastructure_message "$layer_name" "$message_type" "${message_data[@]}"
                ;;
        esac
    }
    
    # Gestion des messages présentation
    $self._handle_presentation_message() {
        local layer_name="$1"
        local message_type="$2"
        shift 2
        
        case "$message_type" in
            user_input)
                echo "  [PRESENTATION] Traitement input utilisateur: $@"
                # Transmission à la couche business
                $self.send_layer_message "$layer_name" "business_logic" "process_request" "$@"
                ;;
                
            display_result)
                echo "  [PRESENTATION] Affichage résultat: $@"
                ;;
                
            show_error)
                echo "  [PRESENTATION] ❌ Erreur affichée: $@"
                ;;
        esac
    }
    
    # Gestion des messages business
    $self._handle_business_message() {
        local layer_name="$1"
        local message_type="$2"
        shift 2
        
        case "$message_type" in
            process_request)
                echo "  [BUSINESS] Traitement requête métier: $@"
                # Validation métier
                if $self._validate_business_rules "$@"; then
                    $self.send_layer_message "$layer_name" "data_access" "query_data" "$@"
                else
                    $self.send_layer_message "$layer_name" "presentation_layer" "show_error" "Règles métier violées"
                fi
                ;;
                
            business_result)
                echo "  [BUSINESS] Résultat métier: $@"
                $self.send_layer_message "$layer_name" "presentation_layer" "display_result" "$@"
                ;;
        esac
    }
    
    # Gestion des messages données
    $self._handle_data_message() {
        local layer_name="$1"
        local message_type="$2"
        shift 2
        
        case "$message_type" in
            query_data)
                echo "  [DATA] Exécution requête: $@"
                # Simulation d'accès aux données
                local result="résultat_de_la_requête_pour_$@"
                $self.send_layer_message "$layer_name" "business_logic" "business_result" "$result"
                ;;
                
            store_data)
                echo "  [DATA] Stockage données: $@"
                ;;
                
            delete_data)
                echo "  [DATA] Suppression données: $@"
                ;;
        esac
    }
    
    # Gestion des messages infrastructure
    $self._handle_infrastructure_message() {
        local layer_name="$1"
        local message_type="$2"
        shift 2
        
        case "$message_type" in
            resource_request)
                echo "  [INFRASTRUCTURE] Allocation ressource: $@"
                ;;
                
            monitoring_alert)
                echo "  [INFRASTRUCTURE] 🚨 Alerte monitoring: $@"
                ;;
                
            health_check)
                echo "  [INFRASTRUCTURE] ✅ Health check: $@"
                ;;
        esac
    }
    
    # Validation des règles métier
    $self._validate_business_rules() {
        local request="$1"
        
        # Règles métier simplifiées
        if [[ "$request" =~ ^(select|insert|update|delete) ]]; then
            echo "✓ Règles métier validées pour: $request"
            return 0
        else
            echo "❌ Règles métier violées pour: $request"
            return 1
        fi
    }
    
    # Démarrage de l'architecture
    $self.start_architecture() {
        echo "=== DÉMARRAGE ARCHITECTURE EN COUCHES ==="
        
        # Initialisation des couches
        for layer_key in "${!$self._layers[@]}"; do
            if [[ "$layer_key" =~ _status$ ]]; then
                local layer_name="${layer_key%_status}"
                $self._layers["${layer_name}_status"]="running"
                echo "✓ Couche démarrée: $layer_name"
            fi
        done
        
        echo "✓ Architecture démarrée"
    }
    
    # Arrêt propre de l'architecture
    $self.stop_architecture() {
        echo "=== ARRÊT ARCHITECTURE EN COUCHES ==="
        
        # Arrêt des couches dans l'ordre inverse
        for layer_key in "${!$self._layers[@]}"; do
            if [[ "$layer_key" =~ _status$ ]]; then
                local layer_name="${layer_key%_status}"
                $self._layers["${layer_name}_status"]="stopped"
                echo "✓ Couche arrêtée: $layer_name"
            fi
        done
        
        echo "✓ Architecture arrêtée"
    }
    
    # Diagnostic de l'architecture
    $self.diagnose_architecture() {
        echo "=== DIAGNOSTIC ARCHITECTURE ==="
        
        echo "État des couches:"
        for layer_key in "${!$self._layers[@]}"; do
            if [[ "$layer_key" =~ _status$ ]]; then
                local layer_name="${layer_key%_status}"
                local layer_type="${$self._layers[${layer_name}_type]}"
                local status="${$self._layers[${layer_name}_status]}"
                
                echo "  $layer_name ($layer_type): $status"
            fi
        done
        
        echo
        echo "Communications récentes:"
        local recent_comms=0
        for comm_key in "${!$self._communication_bus[@]}"; do
            local comm_record="${$self._communication_bus[$comm_key]}"
            echo "  $comm_record"
            ((recent_comms++))
            if (( recent_comms >= 5 )); then
                break
            fi
        done
        
        if (( recent_comms == 0 )); then
            echo "  Aucune communication récente"
        fi
    }
    
    # Génération de rapport d'architecture
    $self.generate_architecture_report() {
        local output_file="${1:-architecture_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT ARCHITECTURE EN COUCHES"
            echo "==============================="
            echo "Généré le: $(date)"
            echo
            
            echo "COUCHES DÉFINIES"
            echo "================"
            
            for layer_key in "${!$self._layers[@]}"; do
                if [[ "$layer_key" =~ _type$ ]]; then
                    local layer_name="${layer_key%_type}"
                    local layer_type="${$self._layers[$layer_key]}"
                    local responsibilities="${$self._layers[${layer_name}_responsibilities]}"
                    local interfaces="${$self._layers[${layer_name}_interfaces]}"
                    local status="${$self._layers[${layer_name}_status]}"
                    
                    echo "Couche: $layer_name"
                    echo "  Type: $layer_type"
                    echo "  Responsabilités: $responsibilities"
                    echo "  Interfaces: $interfaces"
                    echo "  Statut: $status"
                    echo
                fi
            done
            
            echo "DÉPENDANCES ENTRE COUCHES"
            echo "========================="
            
            for dep_key in "${!$self._layer_dependencies[@]}"; do
                local dependency="${$self._layer_dependencies[$dep_key]}"
                local from_layer to_layer
                from_layer="$(echo "$dep_key" | sed 's/_to_.*//')"
                to_layer="$(echo "$dep_key" | sed 's/.*_to_//')"
                
                echo "$from_layer -> $to_layer ($dependency)"
            done
            
            echo
            echo "INTERFACES"
            echo "=========="
            
            for interface_key in "${!$self._layer_interfaces[@]}"; do
                local contract="${$self._layer_interfaces[$interface_key]}"
                echo "$interface_key: $contract"
            done
            
            echo
            echo "COMMUNICATIONS"
            echo "=============="
            
            local total_messages="${#$self._communication_bus[@]}"
            echo "Total messages échangés: $total_messages"
            
            # Statistiques des communications
            local -A message_types
            for comm_key in "${!$self._communication_bus[@]}"; do
                local comm_record="${$self._communication_bus[$comm_key]}"
                local message_type="$(echo "$comm_record" | cut -d: -f4)"
                ((message_types["$message_type"]++))
            done
            
            for msg_type in "${!message_types[@]}"; do
                echo "  $msg_type: ${message_types[$msg_type]}"
            done
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Définition d'une architecture web complète
define_web_architecture() {
    local arch="$1"
    
    # Couche présentation
    $arch.define_layer "presentation_layer" "presentation" \
        "Gestion de l'interface utilisateur, traitement des requêtes HTTP, rendu des réponses" \
        "handle_request,render_response,validate_input"
    
    # Couche métier
    $arch.define_layer "business_logic" "business" \
        "Logique métier, validation des règles business, orchestration des services" \
        "process_business_logic,validate_business_rules,orchestrate_services"
    
    # Couche données
    $arch.define_layer "data_access" "data" \
        "Accès aux données, requêtes base de données, mapping objet-relationnel" \
        "execute_query,save_data,retrieve_data"
    
    # Couche infrastructure
    $arch.define_layer "infrastructure_layer" "infrastructure" \
        "Gestion des ressources système, monitoring, sécurité, logging" \
        "allocate_resources,monitor_system,log_events,handle_security"
    
    # Dépendances
    $arch.define_layer_dependency "presentation_layer" "business_logic" "uses"
    $arch.define_layer_dependency "business_logic" "data_access" "uses"
    $arch.define_layer_dependency "business_logic" "infrastructure_layer" "uses"
    $arch.define_layer_dependency "data_access" "infrastructure_layer" "uses"
    
    # Interfaces
    $arch.register_layer_interface "presentation_layer" "handle_request" "HTTP -> BusinessObject"
    $arch.register_layer_interface "business_logic" "process_business_logic" "BusinessObject -> DataObject"
    $arch.register_layer_interface "data_access" "execute_query" "DataObject -> DatabaseResult"
    $arch.register_layer_interface "infrastructure_layer" "monitor_system" "SystemMetrics -> Alerts"
}

# Démonstration du framework d'architecture en couches
echo "--- Framework d'architecture en couches ---"

LayeredArchitecture "web_architecture"

# Définition de l'architecture web
define_web_architecture "web_architecture"

echo
echo "--- Démarrage de l'architecture ---"
web_architecture.start_architecture

echo
echo "--- Simulation de requêtes ---"

echo "1. Requête utilisateur:"
web_architecture.send_layer_message "presentation_layer" "business_logic" "user_input" "select * from users"

echo
echo "2. Requête invalide:"
web_architecture.send_layer_message "presentation_layer" "business_logic" "user_input" "invalid command"

echo
echo "3. Alerte infrastructure:"
web_architecture.send_layer_message "infrastructure_layer" "presentation_layer" "monitoring_alert" "CPU > 90%"

echo
echo "--- Diagnostic de l'architecture ---"
web_architecture.diagnose_architecture

echo
echo "--- Génération de rapport ---"
web_architecture.generate_architecture_report

echo
echo "--- Arrêt de l'architecture ---"
web_architecture.stop_architecture

# Nettoyage
rm -f architecture_report_*.txt
```

### 1.2 Architecture hexagonale pour la modularité

Pattern hexagonal (ports & adapters) adapté au shell pour une séparation claire des préoccupations :

```bash
#!/bin/bash

# Architecture hexagonale pour la modularité
echo "=== Architecture hexagonale ==="

# Hexagonal Architecture Framework
HexagonalArchitecture() {
    local self="$1"
    
    declare -A $self._core_business_logic
    declare -A $self._ports
    declare -A $self._adapters
    declare -A $self._port_bindings
    
    # Définition de la logique métier core
    $self.define_core_logic() {
        local domain_name="$1"
        local business_rules="$2"
        local domain_entities="$3"
        
        $self._core_business_logic["${domain_name}_rules"]="$business_rules"
        $self._core_business_logic["${domain_name}_entities"]="$domain_entities"
        
        echo "✓ Logique core définie: $domain_name"
    }
    
    # Définition d'un port (interface)
    $self.define_port() {
        local port_name="$1"
        local port_type="$2"  # driving (input), driven (output)
        local interface_contract="$3"
        
        $self._ports["${port_name}_type"]="$port_type"
        $self._ports["${port_name}_contract"]="$interface_contract"
        
        echo "✓ Port défini: $port_name ($port_type)"
    }
    
    # Définition d'un adaptateur
    $self.define_adapter() {
        local adapter_name="$1"
        local port_name="$2"
        local implementation="$3"
        local technology="$4"
        
        $self._adapters["${adapter_name}_port"]="$port_name"
        $self._adapters["${adapter_name}_implementation"]="$implementation"
        $self._adapters["${adapter_name}_technology"]="$technology"
        
        echo "✓ Adaptateur défini: $adapter_name ($technology)"
    }
    
    # Liaison d'un adaptateur à un port
    $self.bind_adapter_to_port() {
        local adapter_name="$1"
        local port_name="$2"
        
        # Vérification de la compatibilité
        local adapter_port="${$self._adapters[${adapter_name}_port]}"
        if [[ "$adapter_port" != "$port_name" ]]; then
            echo "❌ Adaptateur $adapter_name incompatible avec port $port_name" >&2
            return 1
        fi
        
        $self._port_bindings["$port_name"]="$adapter_name"
        
        echo "✓ Liaison établie: $adapter_name -> $port_name"
    }
    
    # Exécution via un port
    $self.execute_through_port() {
        local port_name="$1"
        local operation="$2"
        shift 2
        local -a params=("$@")
        
        local bound_adapter="${$self._port_bindings[$port_name]}"
        
        if [[ -z "$bound_adapter" ]]; then
            echo "❌ Aucun adaptateur lié au port: $port_name" >&2
            return 1
        fi
        
        local port_type="${$self._ports[${port_name}_type]}"
        local implementation="${$self._adapters[${bound_adapter}_implementation]}"
        
        echo "Exécution via port $port_name ($port_type) avec adaptateur $bound_adapter"
        
        # Exécution via l'adaptateur
        $self._execute_adapter_operation "$bound_adapter" "$operation" "${params[@]}"
    }
    
    # Exécution d'une opération d'adaptateur
    $self._execute_adapter_operation() {
        local adapter_name="$1"
        local operation="$2"
        shift 2
        
        local technology="${$self._adapters[${adapter_name}_technology]}"
        
        case "$technology" in
            rest_api)
                $self._execute_rest_adapter "$adapter_name" "$operation" "$@"
                ;;
                
            database)
                $self._execute_database_adapter "$adapter_name" "$operation" "$@"
                ;;
                
            filesystem)
                $self._execute_filesystem_adapter "$adapter_name" "$operation" "$@"
                ;;
                
            cli)
                $self._execute_cli_adapter "$adapter_name" "$operation" "$@"
                ;;
                
            *)
                echo "❌ Technologie d'adaptateur non supportée: $technology" >&2
                return 1
                ;;
        esac
    }
    
    # Adaptateur REST API
    $self._execute_rest_adapter() {
        local adapter_name="$1"
        local operation="$2"
        shift 2
        
        echo "  [REST] $operation: $@"
        
        case "$operation" in
            get_user)
                local user_id="$1"
                echo "  Récupération utilisateur $user_id via REST API"
                # Simulation: curl -s "https://api.example.com/users/$user_id"
                echo "  Résultat: {\"id\": $user_id, \"name\": \"User $user_id\"}"
                ;;
                
            create_user)
                local user_data="$1"
                echo "  Création utilisateur via REST API: $user_data"
                # Simulation: curl -X POST -d "$user_data" "https://api.example.com/users"
                echo "  Résultat: {\"id\": 123, \"status\": \"created\"}"
                ;;
                
            update_user)
                local user_id="$1" user_data="$2"
                echo "  Mise à jour utilisateur $user_id via REST API"
                # Simulation: curl -X PUT -d "$user_data" "https://api.example.com/users/$user_id"
                echo "  Résultat: {\"status\": \"updated\"}"
                ;;
        esac
    }
    
    # Adaptateur base de données
    $self._execute_database_adapter() {
        local adapter_name="$1"
        local operation="$2"
        shift 2
        
        echo "  [DATABASE] $operation: $@"
        
        case "$operation" in
            find_user)
                local user_id="$1"
                echo "  Recherche utilisateur $user_id en base"
                # Simulation: mysql -e "SELECT * FROM users WHERE id=$user_id"
                echo "  Résultat: id=$user_id, name=User$user_id, email=user$user_id@example.com"
                ;;
                
            save_user)
                local user_data="$1"
                echo "  Sauvegarde utilisateur en base: $user_data"
                # Simulation: mysql -e "INSERT INTO users VALUES ($user_data)"
                echo "  Résultat: INSERT successful, id=456"
                ;;
                
            delete_user)
                local user_id="$1"
                echo "  Suppression utilisateur $user_id"
                # Simulation: mysql -e "DELETE FROM users WHERE id=$user_id"
                echo "  Résultat: 1 row affected"
                ;;
        esac
    }
    
    # Adaptateur système de fichiers
    $self._execute_filesystem_adapter() {
        local adapter_name="$1"
        local operation="$2"
        shift 2
        
        echo "  [FILESYSTEM] $operation: $@"
        
        case "$operation" in
            read_file)
                local file_path="$1"
                echo "  Lecture fichier: $file_path"
                # Simulation: cat "$file_path"
                echo "  Résultat: Contenu du fichier $file_path"
                ;;
                
            write_file)
                local file_path="$1" content="$2"
                echo "  Écriture fichier: $file_path"
                # Simulation: echo "$content" > "$file_path"
                echo "  Résultat: Fichier écrit avec succès"
                ;;
                
            list_directory)
                local dir_path="$1"
                echo "  Liste répertoire: $dir_path"
                # Simulation: ls -la "$dir_path"
                echo "  Résultat: fichier1.txt, fichier2.txt, dossier1/"
                ;;
        esac
    }
    
    # Adaptateur CLI
    $self._execute_cli_adapter() {
        local adapter_name="$1"
        local operation="$2"
        shift 2
        
        echo "  [CLI] $operation: $@"
        
        case "$operation" in
            execute_command)
                local command="$1"
                echo "  Exécution commande: $command"
                # Simulation: bash -c "$command"
                echo "  Résultat: Commande exécutée avec succès"
                ;;
                
            check_service)
                local service="$1"
                echo "  Vérification service: $service"
                # Simulation: systemctl is-active "$service"
                echo "  Résultat: Service $service is active"
                ;;
                
            get_system_info)
                echo "  Récupération informations système"
                # Simulation: uname -a
                echo "  Résultat: Linux server 5.4.0 #1 SMP x86_64 GNU/Linux"
                ;;
        esac
    }
    
    # Validation de l'architecture hexagonale
    $self.validate_hexagonal_architecture() {
        echo "=== VALIDATION ARCHITECTURE HEXAGONALE ==="
        
        local errors=0
        
        # Vérification que tous les ports ont des adaptateurs
        for port_key in "${!$self._ports[@]}"; do
            if [[ "$port_key" =~ _type$ ]]; then
                local port_name="${port_key%_type}"
                local bound_adapter="${$self._port_bindings[$port_name]}"
                
                if [[ -z "$bound_adapter" ]]; then
                    echo "❌ Port sans adaptateur: $port_name"
                    ((errors++))
                else
                    echo "✅ Port lié: $port_name -> $bound_adapter"
                fi
            fi
        done
        
        # Vérification que tous les adaptateurs sont liés à des ports
        for adapter_key in "${!$self._adapters[@]}"; do
            if [[ "$adapter_key" =~ _port$ ]]; then
                local adapter_name="${adapter_key%_port}"
                local port_name="${$self._adapters[$adapter_key]}"
                
                if [[ -z "${$self._port_bindings[$port_name]}" ]]; then
                    echo "⚠️  Adaptateur non lié: $adapter_name"
                fi
            fi
        done
        
        # Vérification de la logique métier
        if [[ -z "${$self._core_business_logic[*]}" ]]; then
            echo "❌ Aucune logique métier core définie"
            ((errors++))
        else
            echo "✅ Logique métier core présente"
        fi
        
        echo
        echo "Résumé validation: $errors erreur(s)"
        
        return $(( errors > 0 ))
    }
    
    # Génération de diagramme ASCII de l'architecture
    $self.generate_architecture_diagram() {
        echo "=== DIAGRAMME ARCHITECTURE HEXAGONALE ==="
        echo
        
        echo "                    ┌─────────────────────────────────────┐"
        echo "                    │           LOGIQUE MÉTIER CORE       │"
        echo "                    │        (Business Logic)             │"
        echo "                    └──────────────────┬──────────────────┘"
        echo "                                       │"
        echo "                    ┌──────────────────┼──────────────────┐"
        echo "                    │                  │                  │"
        echo "            ┌───────▼──────┐   ┌───────▼──────┐   │"
        echo "            │   PORTS      │   │   PORTS      │   │"
        echo "            │   (Driving)  │   │   (Driven)   │   │"
        echo "            └───────▲──────┘   └───────▲──────┘   │"
        echo "                    │                  │          │"
        echo "            ┌───────▼──────┐   ┌───────▼──────┐   │"
        echo "            │ ADAPTERS     │   │ ADAPTERS     │   │"
        echo "            │ (REST API)   │   │ (Database)   │   │"
        echo "            └──────────────┘   └──────────────┘   │"
        echo "                                                   │"
        echo "            ┌─────────────────────────────────────┼─────┐"
        echo "            │              TECHNOLOGIES EXTERNES   ▼     │"
        echo "            │     (HTTP, SQL, Filesystem, CLI...)       │"
        echo "            └───────────────────────────────────────────┘"
        echo
        
        echo "Légende:"
        echo "  • Core Business Logic: Règles métier indépendantes"
        echo "  • Ports: Interfaces abstraites pour communiquer"
        echo "  • Adapters: Implémentations concrètes des ports"
        echo "  • Technologies Externes: Frameworks, APIs, bases de données..."
    }
    
    # Génération de rapport hexagonal
    $self.generate_hexagonal_report() {
        local output_file="${1:-hexagonal_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT ARCHITECTURE HEXAGONALE"
            echo "==============================="
            echo "Généré le: $(date)"
            echo
            
            echo "LOGIQUE MÉTIER CORE"
            echo "==================="
            
            for core_key in "${!$self._core_business_logic[@]}"; do
                if [[ "$core_key" =~ _rules$ ]]; then
                    local domain_name="${core_key%_rules}"
                    local rules="${$self._core_business_logic[$core_key]}"
                    local entities="${$self._core_business_logic[${domain_name}_entities]}"
                    
                    echo "Domaine: $domain_name"
                    echo "  Règles: $rules"
                    echo "  Entités: $entities"
                    echo
                fi
            done
            
            echo "PORTS DÉFINIS"
            echo "============="
            
            for port_key in "${!$self._ports[@]}"; do
                if [[ "$port_key" =~ _type$ ]]; then
                    local port_name="${port_key%_type}"
                    local port_type="${$self._ports[$port_key]}"
                    local contract="${$self._ports[${port_name}_contract]}"
                    
                    echo "Port: $port_name ($port_type)"
                    echo "  Contrat: $contract"
                    echo
                fi
            done
            
            echo "ADAPTATEURS DÉFINIS"
            echo "==================="
            
            for adapter_key in "${!$self._adapters[@]}"; do
                if [[ "$adapter_key" =~ _port$ ]]; then
                    local adapter_name="${adapter_key%_port}"
                    local port_name="${$self._adapters[$adapter_key]}"
                    local implementation="${$self._adapters[${adapter_name}_implementation]}"
                    local technology="${$self._adapters[${adapter_name}_technology]}"
                    
                    echo "Adaptateur: $adapter_name"
                    echo "  Port: $port_name"
                    echo "  Implémentation: $implementation"
                    echo "  Technologie: $technology"
                    echo
                fi
            done
            
            echo "LIAISONS PORT-ADAPTATEUR"
            echo "========================"
            
            for port_name in "${!self._port_bindings[@]}"; do
                local adapter_name="${$self._port_bindings[$port_name]}"
                echo "$port_name -> $adapter_name"
            done
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse des dépendances
            local unbound_ports=0
            for port_key in "${!$self._ports[@]}"; do
                if [[ "$port_key" =~ _type$ ]]; then
                    local port_name="${port_key%_type}"
                    if [[ -z "${$self._port_bindings[$port_name]}" ]]; then
                        ((unbound_ports++))
                    fi
                fi
            done
            
            if (( unbound_ports > 0 )); then
                echo "• $unbound_ports port(s) sans adaptateur - l'architecture n'est pas complète"
            fi
            
            local total_adapters="${#self._adapters[@]}"
            local total_ports="${#self._ports[@]}"
            
            if (( total_adapters < total_ports )); then
                echo "• Plus de ports que d'adaptateurs - envisager d'ajouter des adaptateurs"
            fi
            
            if (( total_adapters > total_ports * 2 )); then
                echo "• Beaucoup d'adaptateurs par rapport aux ports - vérifier la nécessité"
            fi
            
            echo "• La logique métier core doit rester indépendante des technologies externes"
            echo "• Les ports doivent définir des contrats clairs et stables"
            echo "• Les adaptateurs peuvent être remplacés sans affecter le core"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Définition d'une architecture hexagonale pour un système de gestion utilisateur
define_user_management_hexagon() {
    local hexagon="$1"
    
    # Logique métier core
    $hexagon.define_core_logic "user_management" \
        "validation_email_unique, password_complexity, role_permissions" \
        "User,Role,Permission"
    
    # Ports d'entrée (driving)
    $hexagon.define_port "user_commands" "driving" "create_user,update_user,delete_user,authenticate_user"
    $hexagon.define_port "user_queries" "driving" "find_user,get_user_list,search_users"
    
    # Ports de sortie (driven)
    $hexagon.define_port "user_repository" "driven" "save_user,find_user_by_id,find_user_by_email,delete_user"
    $hexagon.define_port "notification_service" "driven" "send_welcome_email,send_password_reset"
    $hexagon.define_port "audit_logger" "driven" "log_user_action,log_security_event"
    
    # Adaptateurs pour ports d'entrée
    $hexagon.define_adapter "rest_api_adapter" "user_commands" "RESTController" "rest_api"
    $hexagon.define_adapter "cli_adapter" "user_commands" "CLIHandler" "cli"
    $hexagon.define_adapter "graphql_adapter" "user_queries" "GraphQLResolver" "graphql"
    
    # Adaptateurs pour ports de sortie
    $hexagon.define_adapter "postgresql_adapter" "user_repository" "PostgreSQLUserRepository" "database"
    $hexagon.define_adapter "filesystem_cache" "user_repository" "FileSystemCache" "filesystem"
    $hexagon.define_adapter "smtp_adapter" "notification_service" "SMTPMailer" "smtp"
    $hexagon.define_adapter "syslog_adapter" "audit_logger" "SyslogAuditor" "syslog"
    
    # Liaisons
    $hexagon.bind_adapter_to_port "rest_api_adapter" "user_commands"
    $hexagon.bind_adapter_to_port "postgresql_adapter" "user_repository"
    $hexagon.bind_adapter_to_port "smtp_adapter" "notification_service"
    $hexagon.bind_adapter_to_port "syslog_adapter" "audit_logger"
}

# Démonstration de l'architecture hexagonale
echo "--- Architecture hexagonale ---"

HexagonalArchitecture "user_hexagon"

# Définition de l'architecture utilisateur
define_user_management_hexagon "user_hexagon"

echo
echo "--- Diagramme de l'architecture ---"
user_hexagon.generate_architecture_diagram

echo
echo "--- Validation de l'architecture ---"
user_hexagon.validate_hexagonal_architecture

echo
echo "--- Tests d'exécution ---"

echo "1. Création utilisateur via REST:"
user_hexagon.execute_through_port "user_commands" "create_user" "john.doe@example.com" "password123"

echo
echo "2. Recherche utilisateur via base de données:"
user_hexagon.execute_through_port "user_queries" "find_user" "123"

echo
echo "3. Suppression utilisateur:"
user_hexagon.execute_through_port "user_commands" "delete_user" "456"

echo
echo "4. Envoi notification:"
user_hexagon.execute_through_port "notification_service" "send_welcome_email" "john.doe@example.com"

echo
echo "5. Audit d'action:"
user_hexagon.execute_through_port "audit_logger" "log_user_action" "USER_DELETE" "456" "admin"

echo
echo "--- Génération de rapport ---"
user_hexagon.generate_hexagonal_report

# Nettoyage
rm -f hexagonal_report_*.txt
```

## Section 2 : Systèmes auto-gérants et adaptatifs

### 2.1 Architecture auto-gérante avec apprentissage

Système capable d'apprendre de son environnement et de s'adapter automatiquement :

```bash
#!/bin/bash

# Architecture auto-gérante avec apprentissage
echo "=== Architecture auto-gérante avec apprentissage ==="

# Self-Managing Architecture Framework
SelfManagingArchitecture() {
    local self="$1"
    
    declare -A $self._system_components
    declare -A $self._performance_metrics
    declare -A $self._learning_patterns
    declare -A $self._adaptation_rules
    declare -A $self._system_knowledge
    
    # Enregistrement d'un composant système
    $self.register_component() {
        local component_name="$1"
        local component_type="$2"
        local configuration="$3"
        local health_check="$4"
        
        $self._system_components["${component_name}_type"]="$component_type"
        $self._system_components["${component_name}_config"]="$configuration"
        $self._system_components["${component_name}_health"]="$health_check"
        $self._system_components["${component_name}_status"]="registered"
        $self._system_components["${component_name}_performance_score"]=50
        
        echo "✓ Composant enregistré: $component_name ($component_type)"
    }
    
    # Collecte des métriques de performance
    $self.collect_performance_metrics() {
        echo "Collecte des métriques de performance..."
        
        for component_key in "${!$self._system_components[@]}"; do
            if [[ "$component_key" =~ _type$ ]]; then
                local component_name="${component_key%_type}"
                local component_type="${$self._system_components[$component_key]}"
                
                # Collecte selon le type de composant
                local performance_score
                performance_score="$($self._measure_component_performance "$component_name" "$component_type")"
                
                $self._system_components["${component_name}_performance_score"]="$performance_score"
                $self._performance_metrics["${component_name}_$(date +%s)"]="$performance_score"
                
                echo "  $component_name: $performance_score/100"
            fi
        done
    }
    
    # Mesure de performance d'un composant
    $self._measure_component_performance() {
        local component_name="$1"
        local component_type="$2"
        
        case "$component_type" in
            database)
                # Simulation de métriques base de données
                local connection_time=$(( 10 + RANDOM % 50 ))
                local query_time=$(( 5 + RANDOM % 20 ))
                local score=$(( 100 - connection_time - query_time ))
                echo "$score"
                ;;
                
            web_server)
                # Simulation de métriques serveur web
                local response_time=$(( 20 + RANDOM % 80 ))
                local error_rate=$(( RANDOM % 5 ))
                local score=$(( 100 - response_time/2 - error_rate*10 ))
                echo "$score"
                ;;
                
            cache)
                # Simulation de métriques cache
                local hit_rate=$(( 70 + RANDOM % 30 ))
                local score="$hit_rate"
                echo "$score"
                ;;
                
            filesystem)
                # Simulation de métriques système de fichiers
                local disk_usage=$(( 20 + RANDOM % 60 ))
                local score=$(( 100 - disk_usage ))
                echo "$score"
                ;;
                
            *)
                echo "50"  # Score par défaut
                ;;
        esac
    }
    
    # Apprentissage basé sur les performances
    $self.learn_from_performance() {
        echo "Apprentissage basé sur les performances..."
        
        local -A performance_trends
        
        # Analyse des tendances de performance
        for component_key in "${!$self._system_components[@]}"; do
            if [[ "$component_key" =~ _type$ ]]; then
                local component_name="${component_key%_type}"
                
                # Récupération des dernières métriques
                local recent_scores=""
                for metric_key in "${!$self._performance_metrics[@]}"; do
                    if [[ "$metric_key" =~ ^${component_name}_ ]]; then
                        local score="${$self._performance_metrics[$metric_key]}"
                        recent_scores="${recent_scores:+$recent_scores }$score"
                    fi
                done
                
                # Analyse de tendance simple
                if [[ -n "$recent_scores" ]]; then
                    local trend
                    trend="$($self._analyze_performance_trend "$recent_scores")"
                    performance_trends["$component_name"]="$trend"
                    
                    echo "  $component_name: tendance $trend"
                fi
            fi
        done
        
        # Apprentissage et adaptation
        for component in "${!performance_trends[@]}"; do
            local trend="${performance_trends[$component]}"
            
            case "$trend" in
                degrading)
                    $self._apply_performance_optimization "$component"
                    ;;
                    
                improving)
                    $self._reinforce_successful_pattern "$component"
                    ;;
                    
                stable)
                    # Pas de changement nécessaire
                    ;;
            esac
        done
    }
    
    # Analyse de tendance de performance
    $self._analyze_performance_trend() {
        local scores="$1"
        
        # Conversion en tableau
        local -a score_array=($scores)
        local len="${#score_array[@]}"
        
        if (( len < 3 )); then
            echo "insufficient_data"
            return
        fi
        
        # Comparaison première moitié vs deuxième moitié
        local mid=$(( len / 2 ))
        local first_half_sum=0 second_half_sum=0
        
        for ((i=0; i<mid; i++)); do
            first_half_sum=$(( first_half_sum + score_array[i] ))
        done
        
        for ((i=mid; i<len; i++)); do
            second_half_sum=$(( second_half_sum + score_array[i] ))
        done
        
        local first_avg=$(( first_half_sum / mid ))
        local second_avg=$(( second_half_sum / (len - mid) ))
        
        local diff=$(( second_avg - first_avg ))
        
        if (( diff > 5 )); then
            echo "improving"
        elif (( diff < -5 )); then
            echo "degrading"
        else
            echo "stable"
        fi
    }
    
    # Application d'optimisations de performance
    $self._apply_performance_optimization() {
        local component_name="$1"
        
        local component_type="${$self._system_components[${component_name}_type]}"
        
        echo "Application d'optimisations pour $component_name ($component_type)"
        
        case "$component_type" in
            database)
                echo "  Optimisation base de données:"
                echo "    • Augmentation pool de connexions"
                echo "    • Optimisation des requêtes"
                echo "    • Ajout d'index"
                ;;
                
            web_server)
                echo "  Optimisation serveur web:"
                echo "    • Activation compression gzip"
                echo "    • Configuration cache HTTP"
                echo "    • Ajustement worker processes"
                ;;
                
            cache)
                echo "  Optimisation cache:"
                echo "    • Augmentation taille cache"
                echo "    • Ajustement politique d'éviction"
                echo "    • Optimisation sérialisation"
                ;;
        esac
        
        # Enregistrement de l'adaptation
        $self._system_knowledge["adaptation_$(date +%s)"]="$component_name:performance_optimization"
    }
    
    # Renforcement des patterns réussis
    $self._reinforce_successful_pattern() {
        local component_name="$1"
        
        echo "Renforcement du pattern réussi pour $component_name"
        
        # Analyse de ce qui fonctionne bien
        local component_config="${$self._system_components[${component_name}_config]}"
        
        echo "  Configuration actuelle conservée et documentée comme référence"
        
        # Enregistrement du succès
        $self._system_knowledge["success_$(date +%s)"]="$component_name:pattern_reinforcement"
    }
    
    # Prédiction des besoins futurs
    $self.predict_future_needs() {
        echo "Prédiction des besoins futurs basée sur l'historique..."
        
        local -A usage_patterns
        
        # Analyse des patterns d'usage
        for metric_key in "${!$self._performance_metrics[@]}"; do
            local timestamp="${metric_key##*_}"
            local hour=$(( timestamp % 86400 / 3600 ))
            
            if [[ -z "${usage_patterns[$hour]}" ]]; then
                usage_patterns["$hour"]=0
            fi
            
            ((usage_patterns["$hour"]++))
        done
        
        # Identification des heures de pointe
        local peak_hours=""
        for hour in "${!usage_patterns[@]}"; do
            local count="${usage_patterns[$hour]}"
            if (( count > 10 )); then  # Seuil arbitraire
                peak_hours="${peak_hours:+$peak_hours, }$hour:00"
            fi
        done
        
        if [[ -n "$peak_hours" ]]; then
            echo "Heures de pointe prédites: $peak_hours"
            echo "Recommandations:"
            echo "  • Augmenter les ressources pendant ces périodes"
            echo "  • Mettre en place l'auto-scaling"
            echo "  • Précharger les caches"
        else
            echo "Aucun pattern d'usage clair identifié"
        fi
    }
    
    # Auto-guérison du système
    $self.self_heal() {
        echo "Vérification de l'état système pour auto-guérison..."
        
        local issues_found=0
        
        for component_key in "${!$self._system_components[@]}"; do
            if [[ "$component_key" =~ _health$ ]]; then
                local component_name="${component_key%_health}"
                local health_check="${$self._system_components[$component_key]}"
                local performance_score="${$self._system_components[${component_name}_performance_score]}"
                
                # Vérification de santé
                if [[ -n "$health_check" ]]; then
                    if ! eval "$health_check" 2>/dev/null; then
                        echo "❌ Composant défaillant détecté: $component_name"
                        $self._apply_self_healing "$component_name"
                        ((issues_found++))
                    elif (( performance_score < 30 )); then
                        echo "⚠️  Performance dégradée: $component_name ($performance_score/100)"
                        $self._apply_performance_recovery "$component_name"
                        ((issues_found++))
                    fi
                fi
            fi
        done
        
        if (( issues_found == 0 )); then
            echo "✅ Tous les composants sont sains"
        else
            echo "Auto-guérison appliquée à $issues_found composant(s)"
        fi
    }
    
    # Application de l'auto-guérison
    $self._apply_self_healing() {
        local component_name="$1"
        
        local component_type="${$self._system_components[${component_name}_type]}"
        
        echo "Application de l'auto-guérison pour $component_name ($component_type)"
        
        case "$component_type" in
            database)
                echo "  Tentative de redémarrage du service base de données..."
                echo "  Reconstruction des index corrompus..."
                ;;
                
            web_server)
                echo "  Redémarrage graceful du serveur web..."
                echo "  Vérification des processus worker..."
                ;;
                
            cache)
                echo "  Vidage du cache corrompu..."
                echo "  Redémarrage du service de cache..."
                ;;
        esac
        
        # Enregistrement de l'action de guérison
        $self._system_knowledge["healing_$(date +%s)"]="$component_name:self_healing_applied"
    }
    
    # Récupération de performance
    $self._apply_performance_recovery() {
        local component_name="$1"
        
        echo "Application de la récupération de performance pour $component_name"
        
        echo "  • Redémarrage du composant"
        echo "  • Libération des ressources"
        echo "  • Réinitialisation des connexions"
        
        # Enregistrement
        $self._system_knowledge["recovery_$(date +%s)"]="$component_name:performance_recovery"
    }
    
    # Évolution adaptative du système
    $self.adaptive_evolution() {
        local environmental_factor="$1"
        
        echo "=== ÉVOLUTION ADAPTATIVE ==="
        echo "Facteur environnemental: $environmental_factor"
        
        case "$environmental_factor" in
            high_load)
                echo "Évolution pour haute charge:"
                echo "  • Ajout de composants de cache"
                echo "  • Activation de la compression"
                echo "  • Optimisation des requêtes"
                ;;
                
            low_resources)
                echo "Évolution pour ressources limitées:"
                echo "  • Activation de la mise en veille"
                echo "  • Réduction de la verbosité des logs"
                echo "  • Optimisation de la mémoire"
                ;;
                
            security_threat)
                echo "Évolution pour menaces de sécurité:"
                echo "  • Renforcement des contrôles d'accès"
                echo "  • Activation de l'audit étendu"
                echo "  • Mise à jour des signatures"
                ;;
                
            new_requirements)
                echo "Évolution pour nouveaux besoins:"
                echo "  • Ajout de nouvelles interfaces"
                echo "  • Extension des capacités de stockage"
                echo "  • Intégration de nouveaux protocoles"
                ;;
        esac
        
        # Enregistrement de l'évolution
        $self._system_knowledge["evolution_$(date +%s)"]="$environmental_factor:adaptive_evolution"
    }
    
    # Génération de rapport d'auto-gestion
    $self.generate_self_management_report() {
        local output_file="${1:-self_management_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'AUTO-GESTION SYSTÈME"
            echo "==============================="
            echo "Généré le: $(date)"
            echo
            
            echo "COMPOSANTS SYSTÈME"
            echo "=================="
            
            for component_key in "${!$self._system_components[@]}"; do
                if [[ "$component_key" =~ _type$ ]]; then
                    local component_name="${component_key%_type}"
                    local component_type="${$self._system_components[$component_key]}"
                    local status="${$self._system_components[${component_name}_status]}"
                    local performance="${$self._system_components[${component_name}_performance_score]}"
                    
                    echo "Composant: $component_name ($component_type)"
                    echo "  Statut: $status"
                    echo "  Performance: $performance/100"
                    echo
                fi
            done
            
            echo "MÉTRIQUES DE PERFORMANCE"
            echo "========================"
            
            local total_metrics="${#$self._performance_metrics[@]}"
            echo "Total métriques collectées: $total_metrics"
            
            # Statistiques récentes
            local recent_metrics=0
            local avg_performance=0
            
            for metric_key in "${!$self._performance_metrics[@]}"; do
                local score="${$self._performance_metrics[$metric_key]}"
                avg_performance=$(( avg_performance + score ))
                ((recent_metrics++))
            done
            
            if (( recent_metrics > 0 )); then
                avg_performance=$(( avg_performance / recent_metrics ))
                echo "Performance moyenne récente: $avg_performance/100"
            fi
            
            echo
            echo "CONNAISSANCES ACQUISES"
            echo "======================"
            
            for knowledge_key in "${!$self._system_knowledge[@]}"; do
                local knowledge="${$self._system_knowledge[$knowledge_key]}"
                echo "$(date -d "@${knowledge_key##*_}" '+%Y-%m-%d %H:%M:%S'): $knowledge"
            done
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse basée sur les connaissances acquises
            local healing_actions=0
            local adaptation_actions=0
            
            for knowledge_key in "${!$self._system_knowledge[@]}"; do
                local knowledge="${$self._system_knowledge[$knowledge_key]}"
                
                case "$knowledge" in
                    *:self_healing_applied)
                        ((healing_actions++))
                        ;;
                    *:adaptive_evolution)
                        ((adaptation_actions++))
                        ;;
                esac
            done
            
            if (( healing_actions > 5 )); then
                echo "• Fréquentes actions de guérison - investigation des causes profondes recommandée"
            fi
            
            if (( adaptation_actions > 3 )); then
                echo "• Système très adaptatif - documenter les patterns réussis"
            fi
            
            if (( avg_performance < 60 )); then
                echo "• Performance générale dégradée - optimisation globale recommandée"
            elif (( avg_performance > 90 )); then
                echo "• Excellente performance - maintenir les bonnes pratiques"
            fi
            
            echo "• Collecte continue des métriques pour l'apprentissage"
            echo "• Surveillance des évolutions adaptatives"
            echo "• Documentation des connaissances acquises"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Cycle de vie auto-gérant
    $self.autonomous_lifecycle() {
        echo "=== CYCLE DE VIE AUTONOME ==="
        
        local cycle_count=0
        local max_cycles=5
        
        while (( cycle_count < max_cycles )); do
            ((cycle_count++))
            echo "--- Cycle $cycle_count/$max_cycles ---"
            
            # 1. Collecte des métriques
            $self.collect_performance_metrics
            
            # 2. Apprentissage
            $self.learn_from_performance
            
            # 3. Auto-guérison
            $self.self_heal
            
            # 4. Prédiction
            $self.predict_future_needs
            
            # 5. Évolution adaptative
            if (( cycle_count % 2 == 0 )); then
                $self.adaptive_evolution "high_load"
            fi
            
            echo "Cycle $cycle_count terminé"
            echo
            
            # Pause entre les cycles
            sleep 2
        done
        
        echo "✓ Cycle de vie autonome terminé"
    }
}

# Définition des composants système
define_system_components() {
    local arch="$1"
    
    $arch.register_component "main_database" "database" "postgresql:5432/mydb" "pg_isready -h localhost -p 5432"
    $arch.register_component "web_server" "web_server" "nginx:80" "curl -f http://localhost/ >/dev/null 2>&1"
    $arch.register_component "redis_cache" "cache" "redis:6379" "redis-cli ping >/dev/null 2>&1"
    $arch.register_component "file_storage" "filesystem" "/var/data" "df /var/data >/dev/null 2>&1"
}

# Démonstration de l'architecture auto-gérante
echo "--- Architecture auto-gérante ---"

SelfManagingArchitecture "autonomous_system"

# Définition des composants
define_system_components "autonomous_system"

echo
echo "--- Cycle de vie autonome ---"
autonomous_system.autonomous_lifecycle

echo
echo "--- Génération de rapport final ---"
autonomous_system.generate_self_management_report

# Nettoyage
rm -f self_management_report_*.txt
```

## Conclusion : L'architecture comme organisme cybernétique

Les architectures de systèmes complexes en shell transcendent les simples assemblages de scripts pour devenir des écosystèmes cybernétiques capables d'auto-observation, d'auto-adaptation, et d'évolution autonome. Ces architectures ne sont pas de simples programmes - ce sont des organismes logiciels qui apprennent, s'adaptent, et évoluent dans leur environnement.

**Points clés à retenir :**

1. **Architectures en couches** : Séparation claire des responsabilités avec communication inter-couches structurée
2. **Pattern hexagonal** : Ports et adaptateurs pour une séparation technologique et une testabilité maximale
3. **Systèmes auto-gérants** : Apprentissage continu, auto-guérison, et évolution adaptative basée sur les métriques

Dans le prochain chapitre, nous conclurons notre exploration des techniques avancées de shell avec une réflexion sur les patterns émergents et l'avenir de l'automatisation en shell.

---

**Exercice pratique :** Créez une architecture complète en couches pour un système de gestion de contenu incluant :
- Couche présentation avec API REST
- Couche business avec logique de validation et autorisation
- Couche données avec cache et persistance
- Couche infrastructure avec monitoring et logging
- Pattern hexagonal pour la modularité et la testabilité

**Réflexion :** Comment ces architectures complexes pourraient-elles évoluer pour inclure l'intelligence artificielle, permettant aux systèmes de prendre des décisions autonomes et d'optimiser leur propre architecture en temps réel ?

