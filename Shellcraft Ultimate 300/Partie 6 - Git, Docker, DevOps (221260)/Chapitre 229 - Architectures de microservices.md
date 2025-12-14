# Chapitre 229 - Architectures de microservices

> "Les microservices sont comme des legos : chaque pièce est petite, indépendante, et peut être assemblée de différentes façons pour construire quelque chose de plus grand." - Sam Newman, auteur de "Building Microservices"

## Introduction : L'évolution vers les microservices

Les architectures de microservices représentent l'aboutissement de l'évolution des systèmes distribués. En décomposant les applications monolithiques en services indépendants, elles offrent scalabilité, résilience et agilité de développement. Cependant, cette approche introduit de nouveaux défis en termes de communication, de cohérence des données et de complexité opérationnelle.

Dans ce chapitre, nous explorerons les principes des microservices, les patterns de conception, les défis opérationnels, et les outils modernes pour les gérer.

## Section 1 : Principes fondamentaux des microservices

### 1.1 Caractéristiques des microservices

**Les 9 principes clés :**
```bash
echo "=== LES 9 PRINCIPES DES MICROSERVICES ==="
echo ""
echo "1. Responsabilité unique (Single Responsibility)"
echo "   • Un service = une responsabilité métier"
echo "   • Changements isolés et prévisibles"
echo "   • Maintenance et évolution simplifiées"
echo ""
echo "2. Autonomie (Autonomous)"
echo "   • Déploiement indépendant"
echo "   • Évolution technologique indépendante"
echo "   • Équipes autonomes"
echo ""
echo "3. Résilience (Resilient)"
echo "   • Tolérance aux pannes"
echo "   • Dégradation gracieuse"
echo "   • Isolation des défaillances"
echo ""
echo "4. Observabilité (Observable)"
echo "   • Métriques détaillées"
echo "   • Logs structurés"
echo "   • Tracing distribué"
echo ""
echo "5. Automatisation (Automated)"
echo "   • Tests automatisés"
echo "   • Déploiement automatisé"
echo "   • Monitoring automatisé"
echo ""
echo "6. Domain-Driven Design"
echo "   • Modélisation métier"
echo "   • Langage ubiquitaire"
echo "   • Contextes bornés"
echo ""
echo "7. API-First"
echo "   • Contrats d'API explicites"
echo "   • Versionnement d'API"
echo "   • Documentation automatique"
echo ""
echo "8. Infrastructure as Code"
echo "   • Infrastructure programmable"
echo "   • Environnements reproductibles"
echo "   • Gestion de configuration"
echo ""
echo "9. Culture DevOps"
echo "   • Collaboration Dev/Ops"
echo "   • Responsabilité partagée"
echo "   • Feedback continu"
```

**Avantages des microservices :**
- **Scalabilité** : Scale indépendamment chaque service
- **Technologique** : Choix technologiques par service
- **Déploiement** : Déploiement indépendant et fréquent
- **Résilience** : Isolation des pannes
- **Innovation** : Expérimentation et évolution rapide

**Inconvénients :**
- **Complexité** : Gestion de la distribution
- **Opérationnel** : Monitoring et debugging complexes
- **Données** : Cohérence et transactions distribuées
- **Communication** : Latence et fiabilité réseau

### 1.2 Décomposition en microservices

**Stratégies de décomposition :**
```bash
echo "=== STRATÉGIES DE DÉCOMPOSITION ==="
echo ""
echo "1. Par domaine métier (Business Domain)"
echo "   • Identification des bounded contexts"
echo "   • Regroupement des fonctionnalités cohérentes"
echo "   • Alignement avec l'organisation"
echo ""
echo "2. Par cas d'usage (Use Case)"
echo "   • Analyse des workflows utilisateur"
echo "   • Groupement des fonctionnalités fréquentes"
echo "   • Optimisation de l'expérience utilisateur"
echo ""
echo "3. Par ressources (Resource)"
echo "   • Services CRUD par entité"
echo "   • API RESTful cohérente"
echo "   • Évolution indépendante des modèles"
echo ""
echo "4. Par volume (Volume)"
echo "   • Séparation des services haute/fréquente charge"
echo "   • Isolation des goulots d'étranglement"
echo "   • Optimisation des performances"
echo ""
echo "5. Par équipe (Team)"
echo "   • Alignement avec la structure d'équipe"
echo "   • Ownership et responsabilité clairs"
echo "   • Réduction des dépendances inter-équipes"
```

**Anti-patterns à éviter :**
- **Microservices nano** : Services trop petits et nombreux
- **Mégaservices** : Services trop gros (mini-monolithes)
- **Services partagés** : Dépendances cachées
- **Couplage technologique** : Forcer la même techno partout
- **Couplage organisationnel** : Services transversaux à toutes les équipes

## Section 2 : Communication inter-services

### 2.1 Patterns synchrones

```python
# api_gateway.py - Gateway API avec Flask
from flask import Flask, request, jsonify
import requests
import time
import logging

app = Flask(__name__)
logger = logging.getLogger(__name__)

# Configuration des services
SERVICES = {
    'user': 'http://user-service:8080',
    'product': 'http://product-service:8080',
    'order': 'http://order-service:8080',
    'payment': 'http://payment-service:8080'
}

class CircuitBreaker:
    """Circuit Breaker pattern"""
    
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failure_count = 0
        self.last_failure_time = 0
        self.state = 'CLOSED'  # CLOSED, OPEN, HALF_OPEN
    
    def call(self, func, *args, **kwargs):
        if self.state == 'OPEN':
            if time.time() - self.last_failure_time > self.timeout:
                self.state = 'HALF_OPEN'
                logger.info("Circuit breaker: HALF_OPEN")
            else:
                raise Exception("Circuit breaker is OPEN")
        
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        self.failure_count = 0
        self.state = 'CLOSED'
    
    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        if self.failure_count >= self.failure_threshold:
            self.state = 'OPEN'
            logger.warning(f"Circuit breaker: OPEN after {self.failure_count} failures")

# Instances de circuit breakers
circuit_breakers = {
    service: CircuitBreaker() for service in SERVICES
}

def call_service(service_name, endpoint, method='GET', data=None, timeout=5):
    """Appel sécurisé d'un service avec circuit breaker"""
    if service_name not in SERVICES:
        raise ValueError(f"Service inconnu: {service_name}")
    
    url = f"{SERVICES[service_name]}{endpoint}"
    
    def _call():
        response = requests.request(
            method=method,
            url=url,
            json=data,
            timeout=timeout,
            headers={'X-Correlation-ID': request.headers.get('X-Correlation-ID', '')}
        )
        response.raise_for_status()
        return response.json()
    
    return circuit_breakers[service_name].call(_call)

@app.route('/api/users/<int:user_id>')
def get_user_profile(user_id):
    """Agrégation de données utilisateur"""
    correlation_id = request.headers.get('X-Correlation-ID', '')
    logger.info(f"[{correlation_id}] Récupération profil utilisateur {user_id}")
    
    try:
        # Récupération des données utilisateur
        user_data = call_service('user', f'/users/{user_id}')
        
        # Récupération des commandes récentes (en parallèle)
        import threading
        import queue
        
        result_queue = queue.Queue()
        
        def fetch_orders():
            try:
                orders = call_service('order', f'/orders/user/{user_id}?limit=5')
                result_queue.put(('orders', orders))
            except Exception as e:
                result_queue.put(('orders_error', str(e)))
        
        def fetch_payments():
            try:
                payments = call_service('payment', f'/payments/user/{user_id}?limit=5')
                result_queue.put(('payments', payments))
            except Exception as e:
                result_queue.put(('payments_error', str(e)))
        
        # Lancement des appels parallèles
        threads = [
            threading.Thread(target=fetch_orders),
            threading.Thread(target=fetch_payments)
        ]
        
        for t in threads:
            t.start()
        
        # Collecte des résultats avec timeout
        results = {}
        for t in threads:
            t.join(timeout=3)  # 3 secondes timeout
            if t.is_alive():
                logger.warning(f"[{correlation_id}] Timeout sur un appel parallèle")
        
        while not result_queue.empty():
            key, value = result_queue.get()
            results[key] = value
        
        # Construction de la réponse
        response = {
            'user': user_data,
            'recent_orders': results.get('orders', []),
            'recent_payments': results.get('payments', []),
            'errors': {
                'orders': results.get('orders_error'),
                'payments': results.get('payments_error')
            }
        }
        
        logger.info(f"[{correlation_id}] Profil utilisateur {user_id} récupéré")
        return jsonify(response)
        
    except Exception as e:
        logger.error(f"[{correlation_id}] Erreur récupération profil {user_id}: {e}")
        return jsonify({'error': 'Service temporairement indisponible'}), 503

@app.route('/api/products/<int:product_id>')
def get_product_details(product_id):
    """Détails produit avec recommandations"""
    correlation_id = request.headers.get('X-Correlation-ID', '')
    logger.info(f"[{correlation_id}] Récupération détails produit {product_id}")
    
    try:
        # Récupération du produit
        product = call_service('product', f'/products/{product_id}')
        
        # Recommandations (appel asynchrone - ne bloque pas)
        recommendations = []
        try:
            rec_thread = threading.Thread(
                target=lambda: recommendations.extend(
                    call_service('product', f'/products/{product_id}/recommendations')
                )
            )
            rec_thread.start()
            rec_thread.join(timeout=1)  # Timeout court pour les recommandations
        except:
            pass  # Les recommandations sont optionnelles
        
        response = {
            'product': product,
            'recommendations': recommendations
        }
        
        return jsonify(response)
        
    except requests.exceptions.RequestException as e:
        if e.response and e.response.status_code == 404:
            return jsonify({'error': 'Produit non trouvé'}), 404
        else:
            logger.error(f"[{correlation_id}] Erreur produit {product_id}: {e}")
            return jsonify({'error': 'Service temporairement indisponible'}), 503

@app.route('/health')
def health_check():
    """Health check avec vérification des dépendances"""
    health_status = {
        'status': 'healthy',
        'timestamp': time.time(),
        'services': {}
    }
    
    for service_name, service_url in SERVICES.items():
        try:
            # Test rapide de connectivité
            response = requests.get(f"{service_url}/health", timeout=2)
            if response.status_code == 200:
                health_status['services'][service_name] = 'healthy'
            else:
                health_status['services'][service_name] = 'unhealthy'
                health_status['status'] = 'degraded'
        except:
            health_status['services'][service_name] = 'unreachable'
            health_status['status'] = 'unhealthy'
    
    status_code = 200 if health_status['status'] == 'healthy' else 503
    return jsonify(health_status), status_code

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

### 2.2 Patterns asynchrones

```python
# event_driven_service.py - Service piloté par événements
from flask import Flask, request, jsonify
import pika
import json
import threading
import time
import logging
from dataclasses import dataclass
from typing import Dict, Any

app = Flask(__name__)
logger = logging.getLogger(__name__)

@dataclass
class Event:
    """Structure d'événement"""
    id: str
    type: str
    source: str
    timestamp: float
    correlation_id: str
    data: Dict[str, Any]

class EventBus:
    """Bus d'événements avec RabbitMQ"""
    
    def __init__(self, host='rabbitmq', exchange='events'):
        self.host = host
        self.exchange = exchange
        self.connection = None
        self.channel = None
        self.connect()
    
    def connect(self):
        """Connexion à RabbitMQ"""
        try:
            self.connection = pika.BlockingConnection(
                pika.ConnectionParameters(host=self.host)
            )
            self.channel = self.connection.channel()
            self.channel.exchange_declare(
                exchange=self.exchange,
                exchange_type='topic',
                durable=True
            )
            logger.info("Connecté au bus d'événements")
        except Exception as e:
            logger.error(f"Erreur connexion RabbitMQ: {e}")
    
    def publish(self, event: Event, routing_key: str):
        """Publication d'événement"""
        if not self.channel:
            self.connect()
        
        try:
            self.channel.basic_publish(
                exchange=self.exchange,
                routing_key=routing_key,
                body=json.dumps({
                    'id': event.id,
                    'type': event.type,
                    'source': event.source,
                    'timestamp': event.timestamp,
                    'correlation_id': event.correlation_id,
                    'data': event.data
                }),
                properties=pika.BasicProperties(
                    delivery_mode=2,  # Message persistant
                    content_type='application/json'
                )
            )
            logger.info(f"Événement publié: {event.type}")
        except Exception as e:
            logger.error(f"Erreur publication événement: {e}")
    
    def subscribe(self, routing_key: str, callback):
        """Souscription à des événements"""
        if not self.channel:
            self.connect()
        
        # Création d'une queue exclusive
        result = self.channel.queue_declare(queue='', exclusive=True)
        queue_name = result.method.queue
        
        # Binding de la queue
        self.channel.queue_bind(
            exchange=self.exchange,
            queue=queue_name,
            routing_key=routing_key
        )
        
        # Consommation des messages
        self.channel.basic_consume(
            queue=queue_name,
            on_message_callback=callback,
            auto_ack=True
        )
        
        logger.info(f"Souscrit aux événements: {routing_key}")
        return queue_name
    
    def start_consuming(self):
        """Démarrage de la consommation"""
        if self.channel:
            self.channel.start_consuming()

# Instance globale du bus d'événements
event_bus = EventBus()

class SagaManager:
    """Gestionnaire de sagas pour cohérence éventuelle"""
    
    def __init__(self):
        self.sagas = {}  # saga_id -> état
    
    def start_saga(self, saga_id: str, steps: list):
        """Démarrage d'une saga"""
        self.sagas[saga_id] = {
            'id': saga_id,
            'steps': steps,
            'current_step': 0,
            'status': 'running',
            'compensation_steps': [],
            'correlation_id': ''
        }
        logger.info(f"Saga démarrée: {saga_id}")
    
    def complete_step(self, saga_id: str, step_result: Any = None):
        """Complétion d'une étape de saga"""
        if saga_id not in self.sagas:
            return
        
        saga = self.sagas[saga_id]
        saga['current_step'] += 1
        
        # Enregistrement de l'étape de compensation
        if step_result:
            saga['compensation_steps'].append(step_result)
        
        if saga['current_step'] >= len(saga['steps']):
            saga['status'] = 'completed'
            logger.info(f"Saga complétée: {saga_id}")
            self._publish_saga_event(saga_id, 'completed')
        else:
            self._execute_next_step(saga_id)
    
    def fail_saga(self, saga_id: str, error: str):
        """Échec d'une saga - rollback"""
        if saga_id not in self.sagas:
            return
        
        saga = self.sagas[saga_id]
        saga['status'] = 'failed'
        saga['error'] = error
        
        logger.error(f"Saga échouée: {saga_id} - {error}")
        
        # Exécution des compensations en ordre inverse
        for compensation in reversed(saga['compensation_steps']):
            try:
                self._execute_compensation(compensation)
            except Exception as e:
                logger.error(f"Erreur compensation: {e}")
        
        self._publish_saga_event(saga_id, 'failed')
    
    def _execute_next_step(self, saga_id: str):
        """Exécution de la prochaine étape"""
        saga = self.sagas[saga_id]
        step = saga['steps'][saga['current_step']]
        
        event_bus.publish(
            Event(
                id=f"{saga_id}-{saga['current_step']}",
                type=step['event_type'],
                source='saga-manager',
                timestamp=time.time(),
                correlation_id=saga['correlation_id'],
                data=step['data']
            ),
            step['routing_key']
        )
    
    def _execute_compensation(self, compensation_data: Dict):
        """Exécution d'une compensation"""
        event_bus.publish(
            Event(
                id=f"compensation-{time.time()}",
                type=compensation_data['type'],
                source='saga-manager',
                timestamp=time.time(),
                correlation_id=compensation_data['correlation_id'],
                data=compensation_data['data']
            ),
            compensation_data['routing_key']
        )
    
    def _publish_saga_event(self, saga_id: str, status: str):
        """Publication d'événement de saga"""
        saga = self.sagas[saga_id]
        
        event_bus.publish(
            Event(
                id=f"saga-{saga_id}-{status}",
                type=f'saga.{status}',
                source='saga-manager',
                timestamp=time.time(),
                correlation_id=saga['correlation_id'],
                data={'saga_id': saga_id, 'status': status}
            ),
            'saga.events'
        )

# Instance globale du gestionnaire de sagas
saga_manager = SagaManager()

# Service de commandes (Command Service)
@app.route('/api/orders', methods=['POST'])
def create_order():
    """Création d'une commande avec saga"""
    data = request.json
    correlation_id = request.headers.get('X-Correlation-ID', str(time.time()))
    
    # Validation de base
    if not data.get('user_id') or not data.get('items'):
        return jsonify({'error': 'Données invalides'}), 400
    
    # Génération d'ID de commande
    order_id = f"order-{int(time.time())}-{data['user_id']}"
    
    # Définition des étapes de la saga
    saga_steps = [
        {
            'event_type': 'order.reserved',
            'routing_key': 'inventory.reserve',
            'data': {
                'order_id': order_id,
                'items': data['items']
            }
        },
        {
            'event_type': 'payment.authorized',
            'routing_key': 'payment.authorize',
            'data': {
                'order_id': order_id,
                'user_id': data['user_id'],
                'amount': data['total_amount']
            }
        },
        {
            'event_type': 'order.confirmed',
            'routing_key': 'order.confirm',
            'data': {
                'order_id': order_id
            }
        }
    ]
    
    # Démarrage de la saga
    saga_manager.start_saga(order_id, saga_steps)
    saga_manager.sagas[order_id]['correlation_id'] = correlation_id
    
    # Publication de l'événement initial
    event_bus.publish(
        Event(
            id=f"order-{order_id}-created",
            type='order.created',
            source='order-service',
            timestamp=time.time(),
            correlation_id=correlation_id,
            data={
                'order_id': order_id,
                'user_id': data['user_id'],
                'items': data['items'],
                'total_amount': data['total_amount']
            }
        ),
        'order.created'
    )
    
    return jsonify({
        'order_id': order_id,
        'status': 'processing',
        'correlation_id': correlation_id
    }), 202

# Gestionnaire d'événements
def handle_inventory_reserved(ch, method, properties, body):
    """Gestionnaire pour inventaire réservé"""
    try:
        event_data = json.loads(body)
        correlation_id = event_data.get('correlation_id')
        
        logger.info(f"[{correlation_id}] Inventaire réservé pour {event_data['order_id']}")
        
        # Enregistrement pour compensation (libération de l'inventaire)
        compensation_data = {
            'type': 'inventory.release',
            'correlation_id': correlation_id,
            'routing_key': 'inventory.release',
            'data': event_data
        }
        
        saga_manager.complete_step(event_data['order_id'], compensation_data)
        
    except Exception as e:
        logger.error(f"Erreur traitement inventaire réservé: {e}")

def handle_payment_failed(ch, method, properties, body):
    """Gestionnaire pour paiement échoué"""
    try:
        event_data = json.loads(body)
        correlation_id = event_data.get('correlation_id')
        
        logger.warning(f"[{correlation_id}] Paiement échoué pour {event_data['order_id']}")
        
        # Échec de la saga
        saga_manager.fail_saga(event_data['order_id'], 'payment_failed')
        
    except Exception as e:
        logger.error(f"Erreur traitement paiement échoué: {e}")

def handle_payment_authorized(ch, method, properties, body):
    """Gestionnaire pour paiement autorisé"""
    try:
        event_data = json.loads(body)
        correlation_id = event_data.get('correlation_id')
        
        logger.info(f"[{correlation_id}] Paiement autorisé pour {event_data['order_id']}")
        
        # Enregistrement pour compensation (remboursement)
        compensation_data = {
            'type': 'payment.refund',
            'correlation_id': correlation_id,
            'routing_key': 'payment.refund',
            'data': event_data
        }
        
        saga_manager.complete_step(event_data['order_id'], compensation_data)
        
    except Exception as e:
        logger.error(f"Erreur traitement paiement autorisé: {e}")

# Démarrage des consommateurs d'événements
def start_event_consumers():
    """Démarrage des consommateurs d'événements en arrière-plan"""
    
    # Inventaire réservé
    event_bus.subscribe('inventory.reserved', handle_inventory_reserved)
    
    # Paiement échoué
    event_bus.subscribe('payment.failed', handle_payment_failed)
    
    # Paiement autorisé
    event_bus.subscribe('payment.authorized', handle_payment_authorized)
    
    # Démarrage de la consommation
    threading.Thread(target=event_bus.start_consuming, daemon=True).start()

# Démarrage des consommateurs au lancement
start_event_consumers()

@app.route('/health')
def health():
    """Health check"""
    return jsonify({
        'status': 'healthy',
        'service': 'order-service',
        'event_bus': 'connected' if event_bus.connection else 'disconnected'
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

## Section 3 : Gestion des données distribuées

### 3.1 Patterns de données

```python
# data_patterns.py - Patterns de gestion des données distribuées
from abc import ABC, abstractmethod
from typing import Dict, List, Any, Optional
import time
import logging
import threading
from dataclasses import dataclass

logger = logging.getLogger(__name__)

@dataclass
class Event:
    """Événement de domaine"""
    aggregate_id: str
    event_type: str
    data: Dict[str, Any]
    timestamp: float
    version: int

class EventStore(ABC):
    """Store d'événements abstrait"""
    
    @abstractmethod
    def save_events(self, aggregate_id: str, events: List[Event], expected_version: int):
        """Sauvegarde des événements"""
        pass
    
    @abstractmethod
    def get_events(self, aggregate_id: str) -> List[Event]:
        """Récupération des événements"""
        pass

class AggregateRoot(ABC):
    """Racine d'agrégat pour Event Sourcing"""
    
    def __init__(self, aggregate_id: str):
        self.aggregate_id = aggregate_id
        self.version = 0
        self.uncommitted_events: List[Event] = []
    
    @abstractmethod
    def apply_event(self, event: Event):
        """Application d'un événement à l'état"""
        pass
    
    def load_from_events(self, events: List[Event]):
        """Reconstruction de l'état depuis les événements"""
        for event in events:
            self.apply_event(event)
            self.version = event.version
    
    def raise_event(self, event_type: str, data: Dict[str, Any]):
        """Déclenchement d'un nouvel événement"""
        event = Event(
            aggregate_id=self.aggregate_id,
            event_type=event_type,
            data=data,
            timestamp=time.time(),
            version=self.version + 1
        )
        
        self.apply_event(event)
        self.uncommitted_events.append(event)
        self.version += 1
    
    def commit_events(self, event_store: EventStore):
        """Validation des événements dans le store"""
        if self.uncommitted_events:
            event_store.save_events(
                self.aggregate_id,
                self.uncommitted_events,
                self.version - len(self.uncommitted_events)
            )
            self.uncommitted_events.clear()

class Order(AggregateRoot):
    """Agrégat Commande avec Event Sourcing"""
    
    def __init__(self, order_id: str):
        super().__init__(order_id)
        self.status = 'created'
        self.items: List[Dict] = []
        self.total_amount = 0.0
    
    def apply_event(self, event: Event):
        """Application des événements à l'état"""
        if event.event_type == 'order_created':
            self.items = event.data['items']
            self.total_amount = event.data['total_amount']
        elif event.event_type == 'order_confirmed':
            self.status = 'confirmed'
        elif event.event_type == 'order_cancelled':
            self.status = 'cancelled'
        elif event.event_type == 'order_shipped':
            self.status = 'shipped'
    
    def create_order(self, items: List[Dict], total_amount: float):
        """Création de commande"""
        if self.status != 'created':
            raise ValueError("Commande déjà traitée")
        
        self.raise_event('order_created', {
            'items': items,
            'total_amount': total_amount
        })
    
    def confirm_order(self):
        """Confirmation de commande"""
        if self.status != 'created':
            raise ValueError("Commande ne peut pas être confirmée")
        
        self.raise_event('order_confirmed', {})
    
    def cancel_order(self, reason: str):
        """Annulation de commande"""
        if self.status in ['shipped', 'cancelled']:
            raise ValueError("Commande ne peut pas être annulée")
        
        self.raise_event('order_cancelled', {'reason': reason})
    
    def ship_order(self, tracking_number: str):
        """Expédition de commande"""
        if self.status != 'confirmed':
            raise ValueError("Commande doit être confirmée avant expédition")
        
        self.raise_event('order_shipped', {'tracking_number': tracking_number})

class ReadModel:
    """Modèle de lecture pour CQRS"""
    
    def __init__(self):
        self.orders: Dict[str, Dict] = {}
        self.lock = threading.RLock()
    
    def update_from_event(self, event: Event):
        """Mise à jour du modèle de lecture"""
        with self.lock:
            order_id = event.aggregate_id
            
            if order_id not in self.orders:
                self.orders[order_id] = {
                    'id': order_id,
                    'status': 'created',
                    'items': [],
                    'total_amount': 0.0,
                    'created_at': event.timestamp,
                    'updated_at': event.timestamp
                }
            
            order = self.orders[order_id]
            
            if event.event_type == 'order_created':
                order.update({
                    'items': event.data['items'],
                    'total_amount': event.data['total_amount']
                })
            elif event.event_type == 'order_confirmed':
                order['status'] = 'confirmed'
            elif event.event_type == 'order_cancelled':
                order['status'] = 'cancelled'
            elif event.event_type == 'order_shipped':
                order['status'] = 'shipped'
            
            order['updated_at'] = event.timestamp
    
    def get_order(self, order_id: str) -> Optional[Dict]:
        """Récupération d'une commande"""
        with self.lock:
            return self.orders.get(order_id)
    
    def get_orders_by_status(self, status: str) -> List[Dict]:
        """Récupération des commandes par statut"""
        with self.lock:
            return [order for order in self.orders.values() if order['status'] == status]

class EventProcessor:
    """Processeur d'événements pour mise à jour des read models"""
    
    def __init__(self, event_store: EventStore, read_model: ReadModel):
        self.event_store = event_store
        self.read_model = read_model
        self.processed_events = set()
    
    def process_events(self):
        """Traitement des nouveaux événements"""
        # Dans un vrai système, on utiliserait des subscriptions
        # Ici, simulation avec un poll simple
        
        # Récupération de tous les agrégats (simplifié)
        aggregate_ids = ['order-1', 'order-2']  # En vrai, depuis un index
        
        for aggregate_id in aggregate_ids:
            events = self.event_store.get_events(aggregate_id)
            
            for event in events:
                event_key = f"{event.aggregate_id}:{event.version}"
                
                if event_key not in self.processed_events:
                    self.read_model.update_from_event(event)
                    self.processed_events.add(event_key)
                    logger.info(f"Événement traité: {event.event_type} pour {event.aggregate_id}")

class OrderService:
    """Service de commandes avec CQRS + Event Sourcing"""
    
    def __init__(self, event_store: EventStore, read_model: ReadModel):
        self.event_store = event_store
        self.read_model = read_model
        self.event_processor = EventProcessor(event_store, read_model)
    
    def create_order(self, order_id: str, items: List[Dict], total_amount: float) -> Dict:
        """Création d'une commande"""
        order = Order(order_id)
        
        # Rechargement depuis l'event store
        events = self.event_store.get_events(order_id)
        if events:
            order.load_from_events(events)
        
        # Création
        order.create_order(items, total_amount)
        
        # Sauvegarde
        order.commit_events(self.event_store)
        
        # Mise à jour du read model
        self.event_processor.process_events()
        
        return self.read_model.get_order(order_id)
    
    def get_order(self, order_id: str) -> Optional[Dict]:
        """Récupération d'une commande"""
        return self.read_model.get_order(order_id)
    
    def confirm_order(self, order_id: str) -> Dict:
        """Confirmation d'une commande"""
        order = Order(order_id)
        events = self.event_store.get_events(order_id)
        order.load_from_events(events)
        
        order.confirm_order()
        order.commit_events(self.event_store)
        
        self.event_processor.process_events()
        
        return self.read_model.get_order(order_id)
    
    def get_orders_by_status(self, status: str) -> List[Dict]:
        """Récupération des commandes par statut"""
        return self.read_model.get_orders_by_status(status)

# Exemple d'implémentation d'EventStore en mémoire (pour démonstration)
class InMemoryEventStore(EventStore):
    """Implémentation simple d'EventStore"""
    
    def __init__(self):
        self.events: Dict[str, List[Event]] = {}
        self.lock = threading.RLock()
    
    def save_events(self, aggregate_id: str, events: List[Event], expected_version: int):
        """Sauvegarde avec contrôle de concurrence optimiste"""
        with self.lock:
            current_events = self.events.get(aggregate_id, [])
            current_version = len(current_events)
            
            if current_version != expected_version:
                raise ValueError(f"Conflit de version pour {aggregate_id}: attendu {expected_version}, actuel {current_version}")
            
            self.events[aggregate_id] = current_events + events
    
    def get_events(self, aggregate_id: str) -> List[Event]:
        """Récupération des événements"""
        with self.lock:
            return self.events.get(aggregate_id, []).copy()

# Utilisation
if __name__ == '__main__':
    # Configuration
    event_store = InMemoryEventStore()
    read_model = ReadModel()
    order_service = OrderService(event_store, read_model)
    
    # Création d'une commande
    order = order_service.create_order('order-123', [
        {'product_id': 'prod-1', 'quantity': 2, 'price': 10.0},
        {'product_id': 'prod-2', 'quantity': 1, 'price': 25.0}
    ], 45.0)
    
    print("Commande créée:", order)
    
    # Confirmation
    confirmed_order = order_service.confirm_order('order-123')
    print("Commande confirmée:", confirmed_order)
    
    # Récupération
    retrieved_order = order_service.get_order('order-123')
    print("Commande récupérée:", retrieved_order)
```

### 3.2 Cohérence et transactions

```python
# distributed_transactions.py - Gestion des transactions distribuées
import time
import logging
from typing import Dict, List, Any, Callable
from dataclasses import dataclass, field
from enum import Enum
import threading

logger = logging.getLogger(__name__)

class TransactionState(Enum):
    """États d'une transaction distribuée"""
    PENDING = "pending"
    PREPARED = "prepared"
    COMMITTED = "committed"
    ABORTED = "aborted"

@dataclass
class TransactionParticipant:
    """Participant à une transaction distribuée"""
    service_name: str
    operation: str
    data: Dict[str, Any]
    prepare_callback: Callable[[], bool]
    commit_callback: Callable[[], bool]
    rollback_callback: Callable[[], bool]
    state: TransactionState = TransactionState.PENDING

@dataclass
class DistributedTransaction:
    """Transaction distribuée avec 2PC"""
    transaction_id: str
    participants: List[TransactionParticipant] = field(default_factory=list)
    state: TransactionState = TransactionState.PENDING
    created_at: float = field(default_factory=time.time)
    coordinator_service: str = ""

class TwoPhaseCommitCoordinator:
    """Coordinateur 2PC"""
    
    def __init__(self, timeout: int = 30):
        self.transactions: Dict[str, DistributedTransaction] = {}
        self.timeout = timeout
        self.lock = threading.RLock()
    
    def begin_transaction(self, transaction_id: str, coordinator_service: str) -> DistributedTransaction:
        """Démarrage d'une transaction distribuée"""
        with self.lock:
            if transaction_id in self.transactions:
                raise ValueError(f"Transaction {transaction_id} déjà existante")
            
            transaction = DistributedTransaction(
                transaction_id=transaction_id,
                coordinator_service=coordinator_service
            )
            
            self.transactions[transaction_id] = transaction
            logger.info(f"Transaction {transaction_id} démarrée")
            
            return transaction
    
    def add_participant(self, transaction_id: str, participant: TransactionParticipant):
        """Ajout d'un participant à la transaction"""
        with self.lock:
            if transaction_id not in self.transactions:
                raise ValueError(f"Transaction {transaction_id} inexistante")
            
            transaction = self.transactions[transaction_id]
            
            if transaction.state != TransactionState.PENDING:
                raise ValueError(f"Transaction {transaction_id} n'est plus en cours")
            
            transaction.participants.append(participant)
            logger.info(f"Participant {participant.service_name} ajouté à {transaction_id}")
    
    def prepare_transaction(self, transaction_id: str) -> bool:
        """Phase 1: Préparation"""
        with self.lock:
            if transaction_id not in self.transactions:
                raise ValueError(f"Transaction {transaction_id} inexistante")
            
            transaction = self.transactions[transaction_id]
            
            if transaction.state != TransactionState.PENDING:
                raise ValueError(f"Transaction {transaction_id} pas en état PENDING")
            
            logger.info(f"Début phase de préparation pour {transaction_id}")
            
            # Envoi des demandes de préparation à tous les participants
            prepared_participants = []
            
            for participant in transaction.participants:
                try:
                    logger.info(f"Préparation de {participant.service_name}")
                    
                    # Simulation d'appel réseau avec timeout
                    success = self._call_with_timeout(
                        participant.prepare_callback,
                        self.timeout
                    )
                    
                    if success:
                        participant.state = TransactionState.PREPARED
                        prepared_participants.append(participant)
                        logger.info(f"{participant.service_name} préparé avec succès")
                    else:
                        logger.warning(f"Échec préparation {participant.service_name}")
                        break
                        
                except Exception as e:
                    logger.error(f"Erreur préparation {participant.service_name}: {e}")
                    break
            
            # Si tous les participants sont préparés, passer à COMMITTED
            if len(prepared_participants) == len(transaction.participants):
                transaction.state = TransactionState.PREPARED
                logger.info(f"Phase de préparation réussie pour {transaction_id}")
                return True
            else:
                # Annulation de la transaction
                transaction.state = TransactionState.ABORTED
                self._rollback_transaction(transaction)
                logger.warning(f"Phase de préparation échouée pour {transaction_id}")
                return False
    
    def commit_transaction(self, transaction_id: str) -> bool:
        """Phase 2: Validation"""
        with self.lock:
            if transaction_id not in self.transactions:
                raise ValueError(f"Transaction {transaction_id} inexistante")
            
            transaction = self.transactions[transaction_id]
            
            if transaction.state != TransactionState.PREPARED:
                raise ValueError(f"Transaction {transaction_id} pas en état PREPARED")
            
            logger.info(f"Début phase de validation pour {transaction_id}")
            
            # Validation chez tous les participants
            committed_participants = []
            
            for participant in transaction.participants:
                try:
                    logger.info(f"Validation chez {participant.service_name}")
                    
                    success = self._call_with_timeout(
                        participant.commit_callback,
                        self.timeout
                    )
                    
                    if success:
                        participant.state = TransactionState.COMMITTED
                        committed_participants.append(participant)
                        logger.info(f"{participant.service_name} validé avec succès")
                    else:
                        logger.error(f"Échec validation {participant.service_name}")
                        # En cas d'échec en phase 2, c'est critique !
                        break
                        
                except Exception as e:
                    logger.error(f"Erreur validation {participant.service_name}: {e}")
                    break
            
            # Si tous validés
            if len(committed_participants) == len(transaction.participants):
                transaction.state = TransactionState.COMMITTED
                logger.info(f"Transaction {transaction_id} validée avec succès")
                return True
            else:
                # Situation critique - certains participants ont pu être validés
                logger.critical(f"Échec critique de validation pour {transaction_id}")
                # En vrai, il faudrait une procédure de récupération
                return False
    
    def rollback_transaction(self, transaction_id: str):
        """Annulation de transaction"""
        with self.lock:
            if transaction_id not in self.transactions:
                raise ValueError(f"Transaction {transaction_id} inexistante")
            
            transaction = self.transactions[transaction_id]
            
            if transaction.state == TransactionState.COMMITTED:
                raise ValueError(f"Transaction {transaction_id} déjà validée")
            
            logger.info(f"Annulation de {transaction_id}")
            
            self._rollback_transaction(transaction)
            transaction.state = TransactionState.ABORTED
    
    def _rollback_transaction(self, transaction: DistributedTransaction):
        """Exécution du rollback chez tous les participants"""
        for participant in transaction.participants:
            if participant.state == TransactionState.PREPARED:
                try:
                    logger.info(f"Rollback chez {participant.service_name}")
                    
                    success = self._call_with_timeout(
                        participant.rollback_callback,
                        self.timeout
                    )
                    
                    if success:
                        participant.state = TransactionState.ABORTED
                        logger.info(f"Rollback réussi chez {participant.service_name}")
                    else:
                        logger.error(f"Échec rollback chez {participant.service_name}")
                        
                except Exception as e:
                    logger.error(f"Erreur rollback {participant.service_name}: {e}")
    
    def _call_with_timeout(self, callback: Callable, timeout: int) -> bool:
        """Appel d'une fonction avec timeout"""
        result = [False]
        
        def call_callback():
            try:
                result[0] = callback()
            except Exception as e:
                logger.error(f"Erreur dans callback: {e}")
                result[0] = False
        
        thread = threading.Thread(target=call_callback)
        thread.start()
        thread.join(timeout)
        
        if thread.is_alive():
            logger.error("Timeout dans callback")
            return False
        
        return result[0]
    
    def cleanup_old_transactions(self, max_age: int = 3600):
        """Nettoyage des vieilles transactions"""
        with self.lock:
            current_time = time.time()
            to_remove = []
            
            for tx_id, transaction in self.transactions.items():
                if current_time - transaction.created_at > max_age:
                    to_remove.append(tx_id)
            
            for tx_id in to_remove:
                del self.transactions[tx_id]
                logger.info(f"Transaction {tx_id} nettoyée")

# Exemple d'utilisation
def demo_distributed_transaction():
    """Démonstration d'une transaction distribuée"""
    
    coordinator = TwoPhaseCommitCoordinator()
    
    # Création d'une transaction pour un transfert bancaire
    transaction = coordinator.begin_transaction("transfer-123", "banking-service")
    
    # Définition des participants (comptes débiteur et créditeur)
    def prepare_debit():
        logger.info("Préparation débit compte source")
        # Vérification du solde, etc.
        return True
    
    def commit_debit():
        logger.info("Validation débit compte source")
        # Débit effectif
        return True
    
    def rollback_debit():
        logger.info("Annulation débit compte source")
        # Remise en place des fonds
        return True
    
    def prepare_credit():
        logger.info("Préparation crédit compte destination")
        # Vérification compte destination
        return True
    
    def commit_credit():
        logger.info("Validation crédit compte destination")
        # Crédit effectif
        return True
    
    def rollback_credit():
        logger.info("Annulation crédit compte destination")
        # Annulation du crédit
        return True
    
    # Ajout des participants
    coordinator.add_participant("transfer-123", TransactionParticipant(
        service_name="account-service-debit",
        operation="debit",
        data={"account_id": "acc-123", "amount": 100.0},
        prepare_callback=prepare_debit,
        commit_callback=commit_debit,
        rollback_callback=rollback_debit
    ))
    
    coordinator.add_participant("transfer-123", TransactionParticipant(
        service_name="account-service-credit",
        operation="credit", 
        data={"account_id": "acc-456", "amount": 100.0},
        prepare_callback=prepare_credit,
        commit_callback=commit_credit,
        rollback_callback=rollback_credit
    ))
    
    # Exécution de la transaction
    logger.info("Démarrage transaction distribuée")
    
    if coordinator.prepare_transaction("transfer-123"):
        logger.info("Phase de préparation réussie, validation...")
        
        if coordinator.commit_transaction("transfer-123"):
            logger.info("Transaction distribuée réussie!")
        else:
            logger.error("Échec de validation de la transaction")
    else:
        logger.error("Échec de préparation de la transaction")
    
    # Nettoyage
    coordinator.cleanup_old_transactions()

if __name__ == '__main__':
    demo_distributed_transaction()
```

## Section 4 : Déploiement et orchestration

### 4.1 Service mesh avec Istio

```yaml
# istio-config.yml - Configuration Istio pour microservices
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: api-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: api-tls
    hosts:
    - "api.example.com"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: api-routing
spec:
  hosts:
  - "api.example.com"
  gateways:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: "/api/v1/users"
    route:
    - destination:
        host: user-service
        subset: v1
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
  - match:
    - uri:
        prefix: "/api/v1/products"
    route:
    - destination:
        host: product-service
        subset: v2
    timeout: 3s
  - match:
    - uri:
        prefix: "/api/v1/orders"
    route:
    - destination:
        host: order-service
        subset: v1
    timeout: 10s
    corsPolicy:
      allowOrigin:
      - "*"
      allowMethods:
      - GET
      - POST
      - PUT
      - DELETE
      allowHeaders:
      - "authorization"
      - "content-type"
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: service-routing
spec:
  host: "*.example.com"
  trafficPolicy:
    loadBalancer:
      simple: ROUND_ROBIN
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
---
apiVersion: networking.istio.io/v1alpha3
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: PERMISSIVE
---
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: api-auth
  namespace: default
spec:
  selector:
    matchLabels:
      app: api-gateway
  action: ALLOW
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]
    to:
    - operation:
        methods: ["GET"]
        paths: ["/api/v1/products/*"]
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/api-user"]
    to:
    - operation:
        methods: ["*"]
        paths: ["/api/v1/*"]
---
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: mesh-default
  namespace: istio-system
spec:
  tracing:
  - providers:
    - name: jaeger
  metrics:
  - providers:
    - name: prometheus
---
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-api
spec:
  hosts:
  - api.stripe.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  endpoints:
  - address: api.stripe.com
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: stripe-api
spec:
  hosts:
  - api.stripe.com
  http:
  - timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
    route:
    - destination:
        host: api.stripe.com
```

### 4.2 Gestion du trafic avancé

```yaml
# traffic-management.yml - Gestion avancée du trafic
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: canary-deployment
spec:
  hosts:
  - web-app
  http:
  - match:
    - headers:
        user-agent:
          regex: ".*Chrome.*"
    route:
    - destination:
        host: web-app
        subset: v2
      weight: 100
  - match:
    - headers:
        cookie:
          regex: ".*canary=true.*"
    route:
    - destination:
        host: web-app
        subset: v2
      weight: 100
  - route:
    - destination:
        host: web-app
        subset: v1
      weight: 90
    - destination:
        host: web-app
        subset: v2
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: fault-injection
spec:
  hosts:
  - payment-service
  http:
  - fault:
      delay:
        percentage:
          value: 10.0
        fixedDelay: 2s
      abort:
        percentage:
          value: 5.0
        httpStatus: 503
    route:
    - destination:
        host: payment-service
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: rate-limiting
spec:
  hosts:
  - api-gateway
  http:
  - match:
    - uri:
        prefix: "/api/v1"
    route:
    - destination:
        host: api-gateway
    rateLimit:
      dimensions:
        remoteAddress:
          value: $remoteAddress
      limit:
        requestsPerUnit: 100
        unit: MINUTE
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: circuit-breaker
spec:
  hosts:
  - external-service
  http:
  - route:
    - destination:
        host: external-service
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: "5xx,gateway-error,connect-failure,refused-stream"
  trafficPolicy:
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: mirroring
spec:
  hosts:
  - recommendation-service
  http:
  - route:
    - destination:
        host: recommendation-service
        subset: v1
      weight: 100
    mirror:
      host: recommendation-service
      subset: v2
    mirrorPercentage:
      value: 20.0
```

### 4.3 Observabilité distribuée

```yaml
# observability.yml - Configuration observabilité microservices
apiVersion: telemetry.istio.io/v1alpha1
kind: Telemetry
metadata:
  name: service-observability
spec:
  selector:
    matchLabels:
      app: web-app
  tracing:
  - providers:
    - name: jaeger
    randomSamplingPercentage: 10.0
    customTags:
      environment:
        literal:
          value: production
      version:
        header:
          name: x-app-version
  metrics:
  - providers:
    - name: prometheus
    overrides:
    - match:
        metric: REQUEST_COUNT
        mode: CLIENT
      tagOverrides:
        request_operation:
          value: "method"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    
    scrape_configs:
    - job_name: 'istio-mesh'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - istio-system
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: 'istiod'
    
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-config
data:
  jaeger.yml: |
    disabled: false
    reporter:
      queueSize: 100
      bufferFlushInterval: 10
      logSpans: false
      localAgentHostPort: "jaeger-agent.istio-system:6831"
    sampler:
      type: probabilistic
      param: 0.1
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kiali
spec:
  template:
    spec:
      containers:
      - name: kiali
        image: quay.io/kiali/kiali:v1.57
        env:
        - name: LOGIN_TOKEN_SIGNING_KEY
          value: "change-me"
        ports:
        - containerPort: 20001
        volumeMounts:
        - name: kiali-configuration
          mountPath: /kiali-configuration
      volumes:
      - name: kiali-configuration
        configMap:
          name: kiali
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: kiali
data:
  config.yaml: |
    istio_namespace: istio-system
    auth:
      strategy: anonymous
    external_services:
      prometheus:
        url: http://prometheus.istio-system:9090
      grafana:
        url: http://grafana.istio-system:3000
      jaeger:
        url: http://jaeger-query.istio-system:16686
```

## Conclusion : L'architecture microservices comme évolution

Les microservices représentent une évolution majeure dans la conception d'applications distribuées, permettant aux organisations de scaler leurs équipes et leurs systèmes de manière indépendante. Cependant, cette approche nécessite une solide compréhension des patterns de communication, de gestion des données et d'observabilité.

Dans le prochain chapitre, nous explorerons l'IA et l'intégration de l'intelligence artificielle dans les workflows DevOps modernes.

---

**Exercice pratique :** Créez une architecture microservices complète pour une plateforme e-commerce incluant :
1. Service de catalogue de produits
2. Service de gestion des utilisateurs
3. Service de commandes avec saga pattern
4. Service de paiement avec 2PC
5. API Gateway avec routage intelligent
6. Service mesh avec Istio
7. Observabilité complète (métriques, logs, traces)

**Challenge avancé :** Développez un framework de microservices qui inclut :
- Service discovery automatique
- Configuration centralisée
- Gestion des secrets
- Circuit breakers et retry logic
- Event sourcing et CQRS
- API composition et BFF pattern
- Tests de contrats et integration testing
- Rollback et deployment strategies

**Réflexion :** Comment les architectures microservices transforment-elles la façon dont nous concevons, développons et opérons les systèmes logiciels modernes ? Quels sont les compromis entre complexité et scalabilité ?

