# Chapitre 230 - IA et apprentissage automatique

> "L'IA n'est pas seulement un outil pour automatiser des tâches, c'est une force transformative qui change fondamentalement la façon dont nous concevons, développons et opérons nos systèmes." - Andrew Ng

## Introduction : L'IA comme catalyseur DevOps

L'intelligence artificielle et l'apprentissage automatique transforment radicalement les pratiques DevOps, de l'automatisation intelligente à la prédiction des pannes en passant par l'optimisation continue des performances. En intégrant l'IA dans les pipelines CI/CD et les opérations, les équipes peuvent atteindre de nouveaux niveaux d'efficacité et de fiabilité.

Dans ce chapitre, nous explorerons l'intégration de l'IA dans les workflows DevOps, des modèles de ML pour l'observabilité aux assistants IA pour le développement.

## Section 1 : IA dans l'observabilité et le monitoring

### 1.1 Détection d'anomalies avec ML

```python
# anomaly_detection.py - Détection d'anomalies avec ML
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
import joblib
import time
import logging
from typing import Dict, List, Any, Optional
from dataclasses import dataclass
import prometheus_client as prom
from prometheus_client import Gauge, Counter

logger = logging.getLogger(__name__)

# Métriques Prometheus
ANOMALY_SCORE = Gauge('anomaly_score', 'Score d\'anomalie détecté', ['service', 'metric'])
ANOMALY_COUNT = Counter('anomaly_detected_total', 'Nombre d\'anomalies détectées', ['service', 'metric', 'severity'])

@dataclass
class MetricData:
    """Structure pour les données de métriques"""
    timestamp: float
    value: float
    labels: Dict[str, str]

class AnomalyDetector:
    """Détecteur d'anomalies basé sur Isolation Forest"""
    
    def __init__(self, contamination: float = 0.1, window_size: int = 1000):
        self.contamination = contamination
        self.window_size = window_size
        self.models: Dict[str, IsolationForest] = {}
        self.scalers: Dict[str, StandardScaler] = {}
        self.data_windows: Dict[str, List[float]] = {}
        self.training_data: Dict[str, List[float]] = {}
    
    def add_metric_data(self, metric_name: str, data: MetricData):
        """Ajout de données de métriques pour analyse"""
        if metric_name not in self.data_windows:
            self.data_windows[metric_name] = []
            self.training_data[metric_name] = []
        
        # Ajout à la fenêtre glissante
        self.data_windows[metric_name].append(data.value)
        
        # Limitation de la taille de la fenêtre
        if len(self.data_windows[metric_name]) > self.window_size:
            self.data_windows[metric_name].pop(0)
        
        # Accumulation des données d'entraînement
        self.training_data[metric_name].append(data.value)
        
        # Limitation des données d'entraînement
        if len(self.training_data[metric_name]) > 10000:  # Max 10k points
            self.training_data[metric_name] = self.training_data[metric_name][-5000:]
    
    def train_model(self, metric_name: str) -> bool:
        """Entraînement du modèle pour une métrique"""
        if len(self.training_data[metric_name]) < 100:
            return False  # Pas assez de données
        
        try:
            # Préparation des données
            data = np.array(self.training_data[metric_name]).reshape(-1, 1)
            
            # Normalisation
            if metric_name not in self.scalers:
                self.scalers[metric_name] = StandardScaler()
            
            scaled_data = self.scalers[metric_name].fit_transform(data)
            
            # Entraînement du modèle
            model = IsolationForest(
                contamination=self.contamination,
                random_state=42,
                n_estimators=100
            )
            
            model.fit(scaled_data)
            self.models[metric_name] = model
            
            logger.info(f"Modèle entraîné pour {metric_name}")
            return True
            
        except Exception as e:
            logger.error(f"Erreur entraînement modèle {metric_name}: {e}")
            return False
    
    def detect_anomaly(self, metric_name: str, value: float) -> Optional[Dict[str, Any]]:
        """Détection d'anomalie pour une valeur"""
        if metric_name not in self.models:
            return None
        
        try:
            # Normalisation de la valeur
            scaled_value = self.scalers[metric_name].transform([[value]])
            
            # Prédiction
            prediction = self.models[metric_name].predict(scaled_value)[0]
            score = self.models[metric_name].score_samples(scaled_value)[0]
            
            # Isolation Forest: -1 = anomalie, 1 = normal
            is_anomaly = prediction == -1
            
            # Score d'anomalie (plus négatif = plus anormal)
            anomaly_score = -score
            
            return {
                'is_anomaly': is_anomaly,
                'anomaly_score': anomaly_score,
                'threshold': self.contamination,
                'prediction': prediction
            }
            
        except Exception as e:
            logger.error(f"Erreur détection anomalie {metric_name}: {e}")
            return None
    
    def get_anomaly_stats(self, metric_name: str) -> Dict[str, Any]:
        """Statistiques des anomalies pour une métrique"""
        if metric_name not in self.data_windows:
            return {}
        
        data = np.array(self.data_windows[metric_name])
        
        return {
            'count': len(data),
            'mean': float(np.mean(data)),
            'std': float(np.std(data)),
            'min': float(np.min(data)),
            'max': float(np.max(data)),
            'percentiles': {
                '25': float(np.percentile(data, 25)),
                '50': float(np.percentile(data, 50)),
                '75': float(np.percentile(data, 75)),
                '95': float(np.percentile(data, 95)),
                '99': float(np.percentile(data, 99))
            }
        }

class PrometheusAnomalyDetector:
    """Intégration avec Prometheus pour la détection d'anomalies en temps réel"""
    
    def __init__(self, prometheus_url: str = "http://prometheus:9090"):
        self.prometheus_url = prometheus_url
        self.detector = AnomalyDetector()
        self.last_check = {}
        
        # Métriques à surveiller
        self.monitored_metrics = [
            'http_request_duration_seconds',
            'http_requests_total',
            'cpu_usage_percent',
            'memory_usage_percent',
            'disk_usage_percent'
        ]
    
    def fetch_metric_data(self, metric_name: str, hours: int = 24) -> List[MetricData]:
        """Récupération des données de métriques depuis Prometheus"""
        import requests
        
        # Requête PromQL pour récupérer les données historiques
        query = f'{metric_name}{{job="myapp"}}[{hours}h]'
        
        try:
            response = requests.get(
                f"{self.prometheus_url}/api/v1/query",
                params={'query': query}
            )
            
            if response.status_code != 200:
                return []
            
            data = response.json()
            
            metric_data = []
            for result in data.get('data', {}).get('result', []):
                labels = result.get('metric', {})
                values = result.get('values', [])
                
                for timestamp, value in values:
                    try:
                        metric_data.append(MetricData(
                            timestamp=float(timestamp),
                            value=float(value),
                            labels=labels
                        ))
                    except (ValueError, TypeError):
                        continue
            
            return metric_data
            
        except Exception as e:
            logger.error(f"Erreur récupération métriques {metric_name}: {e}")
            return []
    
    def train_on_historical_data(self):
        """Entraînement sur les données historiques"""
        logger.info("Entraînement sur données historiques...")
        
        for metric_name in self.monitored_metrics:
            logger.info(f"Récupération données pour {metric_name}")
            data_points = self.fetch_metric_data(metric_name, hours=168)  # 7 jours
            
            if not data_points:
                logger.warning(f"Pas de données pour {metric_name}")
                continue
            
            # Ajout des données au détecteur
            for data_point in data_points:
                self.detector.add_metric_data(metric_name, data_point)
            
            # Entraînement du modèle
            if self.detector.train_model(metric_name):
                logger.info(f"Modèle entraîné pour {metric_name}")
            else:
                logger.warning(f"Échec entraînement modèle {metric_name}")
    
    def monitor_metrics(self):
        """Monitoring continu des métriques"""
        logger.info("Démarrage monitoring continu...")
        
        while True:
            try:
                for metric_name in self.monitored_metrics:
                    # Récupération de la dernière valeur
                    current_data = self.fetch_metric_data(metric_name, hours=1)
                    
                    if not current_data:
                        continue
                    
                    latest_data = max(current_data, key=lambda x: x.timestamp)
                    
                    # Vérification si on a déjà traité cette valeur
                    key = f"{metric_name}_{latest_data.timestamp}"
                    if key in self.last_check:
                        continue
                    
                    self.last_check[key] = True
                    
                    # Ajout aux données du détecteur
                    self.detector.add_metric_data(metric_name, latest_data)
                    
                    # Détection d'anomalie
                    anomaly_result = self.detector.detect_anomaly(metric_name, latest_data.value)
                    
                    if anomaly_result and anomaly_result['is_anomaly']:
                        severity = 'high' if anomaly_result['anomaly_score'] > 0.8 else 'medium'
                        
                        # Exposition des métriques Prometheus
                        ANOMALY_SCORE.labels(
                            service=latest_data.labels.get('service', 'unknown'),
                            metric=metric_name
                        ).set(anomaly_result['anomaly_score'])
                        
                        ANOMALY_COUNT.labels(
                            service=latest_data.labels.get('service', 'unknown'),
                            metric=metric_name,
                            severity=severity
                        ).inc()
                        
                        logger.warning(
                            f"ANOMALIE DÉTECTÉE: {metric_name} = {latest_data.value} "
                            f"(score: {anomaly_result['anomaly_score']:.3f}, "
                            f"sévérité: {severity})"
                        )
                        
                        # Ici, on pourrait déclencher des actions automatiques
                        # comme des alertes, des rollbacks, etc.
                
                # Nettoyage des anciennes vérifications (garde les 1000 dernières)
                if len(self.last_check) > 1000:
                    # Garde seulement les 500 plus récentes
                    sorted_checks = sorted(self.last_check.keys(), 
                                         key=lambda x: float(x.split('_')[-1]))
                    to_remove = sorted_checks[:-500]
                    for check in to_remove:
                        del self.last_check[check]
                
                time.sleep(60)  # Vérification toutes les minutes
                
            except Exception as e:
                logger.error(f"Erreur dans le monitoring: {e}")
                time.sleep(60)

def main():
    """Fonction principale"""
    logging.basicConfig(level=logging.INFO)
    
    # Démarrage des métriques Prometheus
    prom.start_http_server(8000)
    
    detector = PrometheusAnomalyDetector()
    
    # Entraînement initial
    detector.train_on_historical_data()
    
    # Monitoring continu
    detector.monitor_metrics()

if __name__ == '__main__':
    main()
```

### 1.2 Prédiction des pannes

```python
# failure_prediction.py - Prédiction des pannes avec ML
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.preprocessing import StandardScaler, LabelEncoder
from sklearn.metrics import classification_report, confusion_matrix
import joblib
import time
import logging
from typing import Dict, List, Any, Optional
from datetime import datetime, timedelta
import psycopg2
from psycopg2.extras import RealDictCursor

logger = logging.getLogger(__name__)

class FailurePredictor:
    """Prédicteur de pannes basé sur l'historique des incidents"""
    
    def __init__(self, db_config: Dict[str, str]):
        self.db_config = db_config
        self.model = None
        self.scaler = StandardScaler()
        self.label_encoders = {}
        self.feature_columns = [
            'cpu_usage_avg', 'cpu_usage_max', 'cpu_usage_std',
            'memory_usage_avg', 'memory_usage_max', 'memory_usage_std',
            'disk_usage_avg', 'disk_usage_max', 'disk_usage_std',
            'network_rx_avg', 'network_tx_avg',
            'request_count', 'error_count', 'error_rate',
            'response_time_avg', 'response_time_p95', 'response_time_p99',
            'service_replicas', 'pod_restarts',
            'hour_of_day', 'day_of_week'
        ]
    
    def connect_db(self):
        """Connexion à la base de données"""
        return psycopg2.connect(**self.db_config)
    
    def extract_features(self, start_date: datetime, end_date: datetime) -> pd.DataFrame:
        """Extraction des features depuis la base de données"""
        
        with self.connect_db() as conn:
            cursor = conn.cursor(cursor_factory=RealDictCursor)
            
            # Requête pour extraire les métriques et incidents
            query = """
            WITH metrics AS (
                SELECT 
                    service_name,
                    DATE_TRUNC('hour', timestamp) as hour,
                    AVG(cpu_usage) as cpu_usage_avg,
                    MAX(cpu_usage) as cpu_usage_max,
                    STDDEV(cpu_usage) as cpu_usage_std,
                    AVG(memory_usage) as memory_usage_avg,
                    MAX(memory_usage) as memory_usage_max,
                    STDDEV(memory_usage) as memory_usage_std,
                    AVG(disk_usage) as disk_usage_avg,
                    MAX(disk_usage) as disk_usage_max,
                    STDDEV(disk_usage) as disk_usage_std,
                    AVG(network_rx) as network_rx_avg,
                    AVG(network_tx) as network_tx_avg,
                    COUNT(*) as request_count,
                    SUM(CASE WHEN status_code >= 500 THEN 1 ELSE 0 END) as error_count,
                    AVG(response_time) as response_time_avg,
                    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time) as response_time_p95,
                    PERCENTILE_CONT(0.99) WITHIN GROUP (ORDER BY response_time) as response_time_p99
                FROM service_metrics
                WHERE timestamp BETWEEN %s AND %s
                GROUP BY service_name, DATE_TRUNC('hour', timestamp)
            ),
            incidents AS (
                SELECT 
                    service_name,
                    DATE_TRUNC('hour', timestamp) as hour,
                    1 as had_failure
                FROM service_incidents
                WHERE timestamp BETWEEN %s AND %s
                GROUP BY service_name, DATE_TRUNC('hour', timestamp)
            ),
            k8s_metrics AS (
                SELECT 
                    service_name,
                    DATE_TRUNC('hour', timestamp) as hour,
                    AVG(replicas) as service_replicas,
                    SUM(pod_restarts) as pod_restarts
                FROM kubernetes_metrics
                WHERE timestamp BETWEEN %s AND %s
                GROUP BY service_name, DATE_TRUNC('hour', timestamp)
            )
            SELECT 
                m.service_name,
                m.hour,
                m.cpu_usage_avg, m.cpu_usage_max, m.cpu_usage_std,
                m.memory_usage_avg, m.memory_usage_max, m.memory_usage_std,
                m.disk_usage_avg, m.disk_usage_max, m.disk_usage_std,
                m.network_rx_avg, m.network_tx_avg,
                m.request_count, 
                COALESCE(i.error_count, 0) as error_count,
                CASE WHEN m.request_count > 0 THEN COALESCE(i.error_count, 0)::float / m.request_count ELSE 0 END as error_rate,
                m.response_time_avg, m.response_time_p95, m.response_time_p99,
                COALESCE(k.service_replicas, 1) as service_replicas,
                COALESCE(k.pod_restarts, 0) as pod_restarts,
                EXTRACT(hour FROM m.hour) as hour_of_day,
                EXTRACT(dow FROM m.hour) as day_of_week,
                CASE WHEN i.service_name IS NOT NULL THEN 1 ELSE 0 END as target
            FROM metrics m
            LEFT JOIN incidents i ON m.service_name = i.service_name AND m.hour = i.hour
            LEFT JOIN k8s_metrics k ON m.service_name = k.service_name AND m.hour = k.hour
            ORDER BY m.hour
            """
            
            cursor.execute(query, (start_date, end_date, start_date, end_date, start_date, end_date))
            rows = cursor.fetchall()
        
        # Conversion en DataFrame
        df = pd.DataFrame(rows)
        
        # Remplacement des NaN
        df = df.fillna(0)
        
        return df
    
    def preprocess_data(self, df: pd.DataFrame) -> tuple:
        """Préprocessing des données"""
        
        # Encodage des variables catégorielles
        if 'service_name' not in self.label_encoders:
            self.label_encoders['service_name'] = LabelEncoder()
        
        df['service_name_encoded'] = self.label_encoders['service_name'].fit_transform(df['service_name'])
        
        # Features et target
        X = df[self.feature_columns + ['service_name_encoded']]
        y = df['target']
        
        # Normalisation
        X_scaled = self.scaler.fit_transform(X)
        
        return X_scaled, y
    
    def train_model(self, start_date: datetime, end_date: datetime):
        """Entraînement du modèle"""
        logger.info(f"Entraînement du modèle du {start_date} au {end_date}")
        
        # Extraction des données
        df = self.extract_features(start_date, end_date)
        
        if len(df) < 100:
            raise ValueError("Pas assez de données pour l'entraînement")
        
        # Préprocessing
        X, y = self.preprocess_data(df)
        
        # Split train/test
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42, stratify=y
        )
        
        # Entraînement
        self.model = RandomForestClassifier(
            n_estimators=100,
            max_depth=10,
            random_state=42,
            class_weight='balanced'
        )
        
        self.model.fit(X_train, y_train)
        
        # Évaluation
        y_pred = self.model.predict(X_test)
        
        logger.info("Rapport de classification:")
        logger.info(classification_report(y_test, y_pred))
        
        logger.info("Matrice de confusion:")
        logger.info(confusion_matrix(y_test, y_pred))
        
        # Sauvegarde du modèle
        joblib.dump(self.model, 'failure_predictor_model.pkl')
        joblib.dump(self.scaler, 'failure_predictor_scaler.pkl')
        joblib.dump(self.label_encoders, 'failure_predictor_encoders.pkl')
        
        logger.info("Modèle sauvegardé")
    
    def load_model(self):
        """Chargement du modèle"""
        try:
            self.model = joblib.load('failure_predictor_model.pkl')
            self.scaler = joblib.load('failure_predictor_scaler.pkl')
            self.label_encoders = joblib.load('failure_predictor_encoders.pkl')
            logger.info("Modèle chargé")
            return True
        except FileNotFoundError:
            logger.warning("Modèle non trouvé")
            return False
    
    def predict_failure_probability(self, service_name: str, metrics: Dict[str, Any]) -> float:
        """Prédiction de la probabilité de panne"""
        if not self.model:
            if not self.load_model():
                return 0.0
        
        try:
            # Préparation des features
            features = []
            for col in self.feature_columns:
                if col in metrics:
                    features.append(metrics[col])
                else:
                    features.append(0)  # Valeur par défaut
            
            # Encodage du service
            if service_name in self.label_encoders['service_name'].classes_:
                service_encoded = self.label_encoders['service_name'].transform([service_name])[0]
            else:
                service_encoded = 0  # Service inconnu
            
            features.append(service_encoded)
            
            # Normalisation
            features_scaled = self.scaler.transform([features])
            
            # Prédiction
            probability = self.model.predict_proba(features_scaled)[0][1]
            
            return float(probability)
            
        except Exception as e:
            logger.error(f"Erreur prédiction pour {service_name}: {e}")
            return 0.0
    
    def get_feature_importance(self) -> Dict[str, float]:
        """Importance des features"""
        if not self.model:
            return {}
        
        importance = {}
        for i, col in enumerate(self.feature_columns + ['service_name_encoded']):
            importance[col] = float(self.model.feature_importances_[i])
        
        return dict(sorted(importance.items(), key=lambda x: x[1], reverse=True))

class ProactiveFailureManager:
    """Gestionnaire proactif des pannes"""
    
    def __init__(self, predictor: FailurePredictor, threshold: float = 0.7):
        self.predictor = predictor
        self.threshold = threshold
        self.mitigation_actions = {
            'scale_up': self.scale_up_service,
            'restart_pods': self.restart_pods,
            'drain_node': self.drain_node,
            'alert_team': self.alert_team
        }
    
    def monitor_and_mitigate(self):
        """Monitoring et mitigation proactifs"""
        logger.info("Démarrage monitoring proactif...")
        
        while True:
            try:
                # Récupération des métriques actuelles pour tous les services
                current_metrics = self.get_current_service_metrics()
                
                for service_name, metrics in current_metrics.items():
                    # Prédiction du risque de panne
                    failure_prob = self.predictor.predict_failure_probability(service_name, metrics)
                    
                    logger.info(f"Service {service_name}: probabilité de panne = {failure_prob:.3f}")
                    
                    if failure_prob > self.threshold:
                        logger.warning(f"Risque élevé détecté pour {service_name} (prob={failure_prob:.3f})")
                        
                        # Analyse des causes probables
                        root_causes = self.analyze_root_causes(metrics)
                        
                        # Actions de mitigation
                        self.execute_mitigation_actions(service_name, root_causes, failure_prob)
                
                time.sleep(300)  # Vérification toutes les 5 minutes
                
            except Exception as e:
                logger.error(f"Erreur dans le monitoring proactif: {e}")
                time.sleep(60)
    
    def get_current_service_metrics(self) -> Dict[str, Dict[str, Any]]:
        """Récupération des métriques actuelles des services"""
        # Simulation - en vrai, récupérer depuis Prometheus/monitoring
        return {
            'web-api': {
                'cpu_usage_avg': 0.8,
                'memory_usage_avg': 0.9,
                'error_rate': 0.05,
                'response_time_avg': 200,
                'request_count': 1000
            },
            'database': {
                'cpu_usage_avg': 0.6,
                'memory_usage_avg': 0.7,
                'disk_usage_avg': 0.85,
                'connection_count': 50
            }
        }
    
    def analyze_root_causes(self, metrics: Dict[str, Any]) -> List[str]:
        """Analyse des causes racines probables"""
        causes = []
        
        if metrics.get('cpu_usage_avg', 0) > 0.8:
            causes.append('high_cpu')
        
        if metrics.get('memory_usage_avg', 0) > 0.8:
            causes.append('high_memory')
        
        if metrics.get('error_rate', 0) > 0.05:
            causes.append('high_error_rate')
        
        if metrics.get('response_time_avg', 0) > 1000:
            causes.append('slow_responses')
        
        return causes
    
    def execute_mitigation_actions(self, service_name: str, causes: List[str], probability: float):
        """Exécution des actions de mitigation"""
        
        # Logique de décision basée sur les causes
        if 'high_cpu' in causes or 'high_memory' in causes:
            if probability > 0.9:
                self.mitigation_actions['scale_up'](service_name)
            else:
                self.mitigation_actions['alert_team'](service_name, f"Ressources élevées détectées")
        
        elif 'high_error_rate' in causes:
            self.mitigation_actions['restart_pods'](service_name)
            self.mitigation_actions['alert_team'](service_name, f"Taux d'erreur élevé détecté")
        
        elif 'slow_responses' in causes:
            self.mitigation_actions['alert_team'](service_name, f"Réponses lentes détectées")
    
    def scale_up_service(self, service_name: str):
        """Scale up d'un service"""
        logger.info(f"Scale up automatique de {service_name}")
        # kubectl scale deployment service-name --replicas=5
        pass
    
    def restart_pods(self, service_name: str):
        """Redémarrage des pods"""
        logger.info(f"Redémarrage automatique des pods de {service_name}")
        # kubectl rollout restart deployment service-name
        pass
    
    def drain_node(self, service_name: str):
        """Drain d'un nœud problématique"""
        logger.info(f"Drain automatique d'un nœud pour {service_name}")
        # kubectl drain node-name
        pass
    
    def alert_team(self, service_name: str, message: str):
        """Alerte l'équipe"""
        logger.warning(f"ALERTE: {service_name} - {message}")
        # Envoyer notification Slack/email/pager
        pass

def main():
    """Fonction principale"""
    logging.basicConfig(level=logging.INFO)
    
    # Configuration DB
    db_config = {
        'host': 'localhost',
        'database': 'monitoring',
        'user': 'monitor',
        'password': 'secret'
    }
    
    # Création du prédicteur
    predictor = FailurePredictor(db_config)
    
    # Entraînement du modèle (si nécessaire)
    try:
        predictor.train_model(
            start_date=datetime.now() - timedelta(days=30),
            end_date=datetime.now()
        )
    except Exception as e:
        logger.error(f"Erreur entraînement: {e}")
        return
    
    # Gestionnaire proactif
    manager = ProactiveFailureManager(predictor, threshold=0.7)
    
    # Démarrage du monitoring proactif
    manager.monitor_and_mitigate()

if __name__ == '__main__':
    main()
```

## Section 2 : IA dans les pipelines CI/CD

### 2.1 Tests intelligents et génération de code

```python
# ai_test_generator.py - Génération de tests avec IA
import openai
import os
import ast
import inspect
from typing import List, Dict, Any, Optional
import logging

logger = logging.getLogger(__name__)

class AITestGenerator:
    """Générateur de tests utilisant l'IA"""
    
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.model = "gpt-4"  # ou "gpt-3.5-turbo"
    
    def analyze_code(self, code: str) -> Dict[str, Any]:
        """Analyse du code pour comprendre sa structure et ses fonctionnalités"""
        
        try:
            tree = ast.parse(code)
            
            functions = []
            classes = []
            
            for node in ast.walk(tree):
                if isinstance(node, ast.FunctionDef):
                    functions.append({
                        'name': node.name,
                        'args': [arg.arg for arg in node.args.args],
                        'returns': self._get_return_annotation(node),
                        'docstring': ast.get_docstring(node) or ""
                    })
                elif isinstance(node, ast.ClassDef):
                    classes.append({
                        'name': node.name,
                        'methods': []
                    })
            
            return {
                'functions': functions,
                'classes': classes,
                'imports': self._extract_imports(tree)
            }
            
        except SyntaxError as e:
            logger.error(f"Erreur de syntaxe dans le code: {e}")
            return {}
    
    def _get_return_annotation(self, node: ast.FunctionDef) -> str:
        """Extraction de l'annotation de retour"""
        if node.returns:
            return ast.unparse(node.returns) if hasattr(ast, 'unparse') else str(node.returns)
        return "Any"
    
    def _extract_imports(self, tree: ast.AST) -> List[str]:
        """Extraction des imports"""
        imports = []
        for node in ast.walk(tree):
            if isinstance(node, ast.Import):
                imports.extend([alias.name for alias in node.names])
            elif isinstance(node, ast.ImportFrom):
                module = node.module or ""
                imports.extend([f"{module}.{alias.name}" if module else alias.name 
                              for alias in node.names])
        return imports
    
    def generate_unit_tests(self, code: str, function_name: str = None) -> str:
        """Génération de tests unitaires"""
        
        analysis = self.analyze_code(code)
        
        if not analysis:
            return ""
        
        # Construction du prompt pour l'IA
        prompt = f"""
        Analyse ce code Python et génère des tests unitaires complets avec pytest.
        
        Code à tester:
        ```python
        {code}
        ```
        
        Analyse du code:
        - Fonctions: {[f['name'] for f in analysis['functions']]}
        - Imports: {analysis['imports']}
        
        Génère des tests unitaires qui couvrent:
        1. Les cas normaux de fonctionnement
        2. Les cas d'erreur et exceptions
        3. Les edge cases
        4. Les tests paramétrisés si nécessaire
        
        Utilise:
        - pytest comme framework de test
        - Des mocks pour les dépendances externes
        - Des assertions appropriées
        - Des fixtures si nécessaire
        
        Structure le code avec des commentaires explicatifs.
        """
        
        if function_name:
            prompt += f"\nConcentre-toi particulièrement sur la fonction: {function_name}"
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en tests logiciels Python. Génère des tests unitaires de haute qualité."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.3
            )
            
            generated_tests = response.choices[0].message.content.strip()
            
            # Nettoyage du code généré (suppression des ```python)
            if generated_tests.startswith("```python"):
                generated_tests = generated_tests[9:]
            if generated_tests.endswith("```"):
                generated_tests = generated_tests[:-3]
            
            return generated_tests.strip()
            
        except Exception as e:
            logger.error(f"Erreur génération tests: {e}")
            return ""
    
    def generate_integration_tests(self, code: str, dependencies: List[str] = None) -> str:
        """Génération de tests d'intégration"""
        
        dependencies = dependencies or []
        
        prompt = f"""
        Analyse ce code Python et génère des tests d'intégration complets.
        
        Code à tester:
        ```python
        {code}
        ```
        
        Dépendances identifiées: {dependencies}
        
        Génère des tests d'intégration qui vérifient:
        1. L'intégration avec les bases de données
        2. L'intégration avec les services externes (APIs)
        3. Les workflows complets
        4. La gestion des erreurs dans les intégrations
        
        Utilise:
        - pytest ou unittest
        - Des conteneurs Docker pour les dépendances (si nécessaire)
        - Des fixtures pour la configuration des tests
        - Des assertions sur les effets de bord
        
        Inclue des exemples de configuration pour les environnements de test.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en tests d'intégration Python. Génère des tests complets et réalistes."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.3
            )
            
            return response.choices[0].message.content.strip()
            
        except Exception as e:
            logger.error(f"Erreur génération tests d'intégration: {e}")
            return ""
    
    def generate_api_tests(self, api_spec: Dict[str, Any]) -> str:
        """Génération de tests API à partir d'une spécification OpenAPI"""
        
        prompt = f"""
        Génère des tests API complets en Python pour cette spécification API:
        
        {api_spec}
        
        Les tests doivent couvrir:
        1. Tous les endpoints définis
        2. Toutes les méthodes HTTP (GET, POST, PUT, DELETE)
        3. Les paramètres de requête et de corps
        4. Les codes de réponse attendus
        5. Les schémas de validation des réponses
        6. Les cas d'erreur (400, 401, 403, 404, 500)
        
        Utilise:
        - requests pour les appels HTTP
        - pytest pour le framework de test
        - jsonschema pour la validation des réponses
        - Des fixtures pour la configuration de base
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en tests API. Génère des tests complets et maintenables."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.3
            )
            
            return response.choices[0].message.content.strip()
            
        except Exception as e:
            logger.error(f"Erreur génération tests API: {e}")
            return ""

class AICodeReviewer:
    """Relecteur de code utilisant l'IA"""
    
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.model = "gpt-4"
    
    def review_code(self, code: str, language: str = "python") -> Dict[str, Any]:
        """Revue de code automatique"""
        
        prompt = f"""
        Analyse ce code {language} et fournis une revue détaillée:
        
        ```{language}
        {code}
        ```
        
        Pour chaque aspect, évalue:
        1. **Sécurité**: Vulnérabilités, bonnes pratiques de sécurité
        2. **Performance**: Optimisations possibles, goulots d'étranglement
        3. **Maintenabilité**: Lisibilité, structure, documentation
        4. **Robustesse**: Gestion d'erreur, edge cases
        5. **Conformité**: Standards de code, patterns établis
        
        Structure la réponse en JSON avec:
        - score global (0-10)
        - scores par catégorie (0-10)
        - problèmes critiques
        - suggestions d'amélioration
        - points positifs
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert senior en revue de code. Fournis des analyses constructives et actionnables."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.2
            )
            
            # Parsing de la réponse JSON
            import json
            review_text = response.choices[0].message.content.strip()
            
            # Nettoyage (suppression des ```json si présent)
            if review_text.startswith("```json"):
                review_text = review_text[7:]
            if review_text.endswith("```"):
                review_text = review_text[:-3]
            
            try:
                return json.loads(review_text.strip())
            except json.JSONDecodeError:
                # Si pas de JSON valide, retourner une structure basique
                return {
                    "score_global": 7,
                    "review": review_text,
                    "error": "Format JSON invalide"
                }
            
        except Exception as e:
            logger.error(f"Erreur revue de code: {e}")
            return {
                "score_global": 0,
                "error": str(e)
            }

# Exemple d'utilisation
def demo_ai_testing():
    """Démonstration de la génération de tests avec IA"""
    
    # Code exemple à tester
    sample_code = '''
def calculate_average(numbers):
    """Calcule la moyenne d'une liste de nombres"""
    if not numbers:
        raise ValueError("La liste ne peut pas être vide")
    
    total = sum(numbers)
    return total / len(numbers)

def find_max_value(data):
    """Trouve la valeur maximale dans une liste ou dict"""
    if isinstance(data, list):
        return max(data) if data else None
    elif isinstance(data, dict):
        return max(data.values()) if data else None
    else:
        raise TypeError("Type non supporté")
'''
    
    # Configuration
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        print("OPENAI_API_KEY non configurée")
        return
    
    generator = AITestGenerator(api_key)
    reviewer = AICodeReviewer(api_key)
    
    # Génération de tests
    print("=== GÉNÉRATION DE TESTS ===")
    tests = generator.generate_unit_tests(sample_code)
    print(tests)
    
    # Revue de code
    print("\n=== REVUE DE CODE ===")
    review = reviewer.review_code(sample_code)
    print(f"Score global: {review.get('score_global', 'N/A')}")
    
    for category, score in review.get('scores_par_categorie', {}).items():
        print(f"{category}: {score}/10")

if __name__ == '__main__':
    demo_ai_testing()
```

### 2.2 Optimisation automatique des performances

```python
# ai_performance_optimizer.py - Optimisation automatique des performances
import ast
import inspect
from typing import Dict, List, Any, Optional, Tuple
import time
import cProfile
import pstats
import io
import logging
import openai
import os

logger = logging.getLogger(__name__)

class PerformanceProfiler:
    """Profileur de performance pour identifier les goulots d'étranglement"""
    
    def __init__(self):
        self.profiles = {}
    
    def profile_function(self, func, *args, **kwargs):
        """Profilage d'une fonction"""
        pr = cProfile.Profile()
        pr.enable()
        
        start_time = time.time()
        result = func(*args, **kwargs)
        execution_time = time.time() - start_time
        
        pr.disable()
        
        # Analyse du profil
        s = io.StringIO()
        ps = pstats.Stats(pr, stream=s).sort_stats('cumulative')
        ps.print_stats()
        
        profile_data = s.getvalue()
        
        self.profiles[func.__name__] = {
            'execution_time': execution_time,
            'profile_data': profile_data,
            'call_count': ps.total_calls
        }
        
        return result
    
    def get_bottlenecks(self, function_name: str) -> List[Dict[str, Any]]:
        """Identification des goulots d'étranglement"""
        if function_name not in self.profiles:
            return []
        
        profile = self.profiles[function_name]
        
        # Analyse simple du profil pour identifier les fonctions lentes
        bottlenecks = []
        lines = profile['profile_data'].split('\n')
        
        for line in lines:
            if 'function calls' in line or not line.strip():
                continue
                
            parts = line.split()
            if len(parts) >= 6:
                try:
                    ncalls = parts[0]
                    tottime = float(parts[1])
                    percall = float(parts[2])
                    cumtime = float(parts[3])
                    percall_cum = float(parts[4])
                    filename_lineno_function = ' '.join(parts[5:])
                    
                    # Identifier les fonctions lentes (>10% du temps total)
                    if cumtime > profile['execution_time'] * 0.1:
                        bottlenecks.append({
                            'function': filename_lineno_function,
                            'cumulative_time': cumtime,
                            'calls': ncalls,
                            'percentage': (cumtime / profile['execution_time']) * 100
                        })
                except (ValueError, IndexError):
                    continue
        
        return sorted(bottlenecks, key=lambda x: x['cumulative_time'], reverse=True)

class AIOptimizer:
    """Optimiseur de code utilisant l'IA"""
    
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.model = "gpt-4"
    
    def analyze_performance_issue(self, code: str, bottlenecks: List[Dict[str, Any]]) -> Dict[str, Any]:
        """Analyse d'un problème de performance"""
        
        bottlenecks_summary = "\n".join([
            f"- {b['function']}: {b['cumulative_time']:.3f}s ({b['percentage']:.1f}%)"
            for b in bottlenecks[:5]  # Top 5
        ])
        
        prompt = f"""
        Analyse ce code Python et les goulots d'étranglement identifiés, puis propose des optimisations:
        
        Code à analyser:
        ```python
        {code}
        ```
        
        Goulets d'étranglement identifiés:
        {bottlenecks_summary}
        
        Fournis une analyse détaillée avec:
        1. **Diagnostic**: Cause racine des problèmes de performance
        2. **Optimisations proposées**: Améliorations spécifiques avec code
        3. **Estimation d'impact**: Amélioration de performance attendue
        4. **Trade-offs**: Conséquences des optimisations
        5. **Code optimisé**: Version améliorée du code
        
        Sois spécifique et fournit des exemples de code concrets.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en optimisation de performance Python. Fournis des analyses techniques précises et des optimisations actionnables."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2500,
                temperature=0.2
            )
            
            analysis = response.choices[0].message.content.strip()
            
            return {
                'analysis': analysis,
                'bottlenecks_analyzed': len(bottlenecks),
                'optimization_suggestions': self._extract_suggestions(analysis)
            }
            
        except Exception as e:
            logger.error(f"Erreur analyse performance: {e}")
            return {'error': str(e)}
    
    def _extract_suggestions(self, analysis: str) -> List[str]:
        """Extraction des suggestions d'optimisation"""
        suggestions = []
        lines = analysis.split('\n')
        
        for line in lines:
            line = line.strip()
            if line.startswith('-') or line.startswith('•'):
                if any(keyword in line.lower() for keyword in 
                    ['optimiser', 'améliorer', 'utiliser', 'remplacer', 'cacher', 'paralléliser']):
                    suggestions.append(line)
        
        return suggestions
    
    def generate_optimized_code(self, original_code: str, analysis: Dict[str, Any]) -> str:
        """Génération de code optimisé"""
        
        prompt = f"""
        À partir de cette analyse de performance, génère une version optimisée du code:
        
        Analyse:
        {analysis.get('analysis', '')}
        
        Code original:
        ```python
        {original_code}
        ```
        
        Génère une version optimisée qui implémente les suggestions de l'analyse.
        Le code doit être:
        - Fonctionnellement équivalent
        - Plus performant
        - Bien documenté
        - Maintenable
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en optimisation de code Python. Génère des implémentations optimisées et maintenables."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.1
            )
            
            optimized_code = response.choices[0].message.content.strip()
            
            # Nettoyage
            if optimized_code.startswith("```python"):
                optimized_code = optimized_code[9:]
            if optimized_code.endswith("```"):
                optimized_code = optimized_code[:-3]
            
            return optimized_code.strip()
            
        except Exception as e:
            logger.error(f"Erreur génération code optimisé: {e}")
            return original_code

class AutoOptimizer:
    """Optimiseur automatique intégrant profilage et IA"""
    
    def __init__(self, api_key: str):
        self.profiler = PerformanceProfiler()
        self.ai_optimizer = AIOptimizer(api_key)
    
    def optimize_function(self, func, *args, **kwargs) -> Dict[str, Any]:
        """Optimisation automatique d'une fonction"""
        
        # Récupération du code source
        source_code = inspect.getsource(func)
        
        # Profilage
        logger.info(f"Profilage de {func.__name__}...")
        result = self.profiler.profile_function(func, *args, **kwargs)
        
        # Analyse des goulots d'étranglement
        bottlenecks = self.profiler.get_bottlenecks(func.__name__)
        
        if not bottlenecks:
            logger.info("Aucun goulot d'étranglement significatif détecté")
            return {
                'original_result': result,
                'bottlenecks': [],
                'optimized_code': None
            }
        
        logger.info(f"Goulots d'étranglement trouvés: {len(bottlenecks)}")
        
        # Analyse IA
        logger.info("Analyse IA des problèmes de performance...")
        analysis = self.ai_optimizer.analyze_performance_issue(source_code, bottlenecks)
        
        # Génération de code optimisé
        logger.info("Génération de code optimisé...")
        optimized_code = self.ai_optimizer.generate_optimized_code(source_code, analysis)
        
        return {
            'original_result': result,
            'bottlenecks': bottlenecks,
            'analysis': analysis,
            'optimized_code': optimized_code
        }

# Fonction exemple à optimiser
def slow_function_example():
    """Fonction lente exemple pour démonstration"""
    
    # Simulation d'une opération coûteuse
    data = []
    for i in range(10000):
        data.append(i * 2)
    
    # Simulation de recherche inefficace
    result = []
    for item in data:
        if item % 3 == 0:
            result.append(item)
    
    # Simulation de calcul répétitif
    total = 0
    for item in result:
        total += item ** 2
    
    return total

def demo_auto_optimization():
    """Démonstration de l'optimisation automatique"""
    
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        print("OPENAI_API_KEY non configurée")
        return
    
    optimizer = AutoOptimizer(api_key)
    
    print("=== OPTIMISATION AUTOMATIQUE ===")
    print("Profilage de slow_function_example...")
    
    # Optimisation
    result = optimizer.optimize_function(slow_function_example)
    
    print(f"Résultat original: {result['original_result']}")
    print(f"Goulots d'étranglement: {len(result['bottlenecks'])}")
    
    if result['bottlenecks']:
        print("\nTop goulots d'étranglement:")
        for i, bottleneck in enumerate(result['bottlenecks'][:3]):
            print(f"{i+1}. {bottleneck['function']}: {bottleneck['cumulative_time']:.3f}s")
    
    if result['analysis']:
        print(f"\nAnalyse IA: {len(result['analysis'].get('analysis', ''))} caractères")
    
    if result['optimized_code']:
        print("\n=== CODE OPTIMISÉ ===")
        print(result['optimized_code'])

if __name__ == '__main__':
    demo_auto_optimization()
```

## Section 3 : Assistants IA pour le développement

### 3.1 Génération automatique de code

```python
# ai_code_assistant.py - Assistant IA pour le développement
import openai
import os
import ast
import re
from typing import Dict, List, Any, Optional, Tuple
import logging

logger = logging.getLogger(__name__)

class AICodeAssistant:
    """Assistant IA pour le développement logiciel"""
    
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.model = "gpt-4"
        self.conversation_history = []
    
    def generate_function(self, description: str, language: str = "python", 
                         constraints: List[str] = None) -> str:
        """Génération d'une fonction à partir d'une description"""
        
        constraints = constraints or []
        
        prompt = f"""
        Génère une fonction {language} complète qui implémente: {description}
        
        Contraintes et exigences:
        {chr(10).join(f"- {c}" for c in constraints)}
        
        La fonction doit:
        1. Être correctement typée (si {language} supporte le typing)
        2. Inclure une docstring complète
        3. Gérer les erreurs appropriées
        4. Être testable unitairement
        5. Suivre les bonnes pratiques de {language}
        
        Inclue des exemples d'utilisation.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"Tu es un expert développeur {language}. Génère du code de haute qualité, maintenable et bien documenté."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.3
            )
            
            code = response.choices[0].message.content.strip()
            
            # Nettoyage du code généré
            code = self._clean_generated_code(code, language)
            
            return code
            
        except Exception as e:
            logger.error(f"Erreur génération fonction: {e}")
            return ""
    
    def refactor_code(self, code: str, improvement_goal: str, language: str = "python") -> str:
        """Refactorisation de code existant"""
        
        prompt = f"""
        Refactorise ce code {language} pour: {improvement_goal}
        
        Code original:
        ```{language}
        {code}
        ```
        
        Fournis:
        1. Le code refactorisé
        2. Les changements apportés et leur justification
        3. Les bénéfices de la refactorisation
        4. Les tests à ajouter/modifier
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"Tu es un expert en refactorisation {language}. Améliore la qualité du code tout en préservant la fonctionnalité."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.2
            )
            
            refactored = response.choices[0].message.content.strip()
            return refactored
            
        except Exception as e:
            logger.error(f"Erreur refactorisation: {e}")
            return code
    
    def explain_code(self, code: str, language: str = "python") -> str:
        """Explication détaillée du fonctionnement d'un code"""
        
        prompt = f"""
        Explique en détail le fonctionnement de ce code {language}:
        
        ```{language}
        {code}
        ```
        
        Pour chaque partie du code, explique:
        1. Ce qu'elle fait
        2. Pourquoi elle est nécessaire
        3. Comment elle s'intègre au reste
        4. Les alternatives possibles
        
        Utilise un langage accessible et des exemples concrets.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"Tu es un excellent pédagogue en {language}. Explique les concepts techniques de manière claire et accessible."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.1
            )
            
            explanation = response.choices[0].message.content.strip()
            return explanation
            
        except Exception as e:
            logger.error(f"Erreur explication code: {e}")
            return "Erreur lors de l'explication du code."
    
    def generate_documentation(self, code: str, language: str = "python") -> str:
        """Génération automatique de documentation"""
        
        prompt = f"""
        Génère une documentation complète pour ce code {language}:
        
        ```{language}
        {code}
        ```
        
        La documentation doit inclure:
        1. Description générale du module/fonction/classe
        2. Paramètres d'entrée et leur type
        3. Valeurs de retour et leur type
        4. Exceptions pouvant être levées
        5. Exemples d'utilisation détaillés
        6. Notes sur les dépendances ou prérequis
        
        Utilise le format de documentation standard pour {language}.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"Tu es un expert en documentation {language}. Génère des documentations claires, complètes et bien structurées."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.2
            )
            
            docs = response.choices[0].message.content.strip()
            return docs
            
        except Exception as e:
            logger.error(f"Erreur génération documentation: {e}")
            return ""
    
    def suggest_improvements(self, code: str, context: str = "", language: str = "python") -> List[str]:
        """Suggestions d'améliorations pour un code"""
        
        prompt = f"""
        Analyse ce code {language} et suggère des améliorations:
        
        Contexte: {context}
        
        Code:
        ```{language}
        {code}
        ```
        
        Suggère des améliorations dans ces catégories:
        1. Performance et optimisation
        2. Sécurité et robustesse
        3. Maintenabilité et lisibilité
        4. Tests et couverture
        5. Bonnes pratiques et patterns
        
        Pour chaque suggestion, explique pourquoi elle est bénéfique.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": f"Tu es un expert senior en {language}. Fournis des suggestions d'amélioration constructives et justifiées."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.3
            )
            
            suggestions_text = response.choices[0].message.content.strip()
            
            # Parsing des suggestions (une par ligne commençant par - ou numéro)
            suggestions = []
            for line in suggestions_text.split('\n'):
                line = line.strip()
                if line and (line.startswith('-') or line[0].isdigit()):
                    suggestions.append(line)
            
            return suggestions
            
        except Exception as e:
            logger.error(f"Erreur suggestions améliorations: {e}")
            return []
    
    def _clean_generated_code(self, code: str, language: str) -> str:
        """Nettoyage du code généré par l'IA"""
        
        # Suppression des balises markdown
        code = re.sub(r'```' + language + r'\n?', '', code, flags=re.IGNORECASE)
        code = re.sub(r'```\n?', '', code)
        
        # Suppression des commentaires d'IA
        lines = code.split('\n')
        cleaned_lines = []
        
        for line in lines:
            # Supprimer les lignes qui semblent être des commentaires d'IA
            if not any(phrase in line.lower() for phrase in [
                'here is', 'here\'s', 'voici', 'généré par',
                'created by', 'ai-generated', 'note:'
            ]):
                cleaned_lines.append(line)
        
        return '\n'.join(cleaned_lines).strip()

class InteractiveCodeAssistant:
    """Assistant interactif pour le développement"""
    
    def __init__(self, api_key: str):
        self.assistant = AICodeAssistant(api_key)
        self.session_context = {}
    
    def start_interactive_session(self):
        """Session interactive avec l'assistant"""
        print("🤖 Assistant IA de Développement")
        print("Tapez 'help' pour voir les commandes disponibles, 'quit' pour quitter")
        
        while True:
            try:
                user_input = input("\nVous: ").strip()
                
                if user_input.lower() in ['quit', 'exit', 'q']:
                    print("Au revoir! 👋")
                    break
                    
                elif user_input.lower() == 'help':
                    self._show_help()
                    
                elif user_input.startswith('generate '):
                    description = user_input[9:].strip()
                    self._handle_generate(description)
                    
                elif user_input.startswith('explain '):
                    code = user_input[8:].strip()
                    self._handle_explain(code)
                    
                elif user_input.startswith('refactor '):
                    parts = user_input[9:].split(' for ')
                    if len(parts) == 2:
                        code, goal = parts
                        self._handle_refactor(code.strip(), goal.strip())
                    else:
                        print("Format: refactor <code> for <goal>")
                        
                elif user_input.startswith('document '):
                    code = user_input[10:].strip()
                    self._handle_document(code)
                    
                elif user_input.startswith('improve '):
                    code = user_input[8:].strip()
                    self._handle_improve(code)
                    
                else:
                    # Traitement comme une question générale
                    self._handle_general_question(user_input)
                    
            except KeyboardInterrupt:
                print("\nAu revoir! 👋")
                break
            except Exception as e:
                print(f"Erreur: {e}")
    
    def _show_help(self):
        """Affichage de l'aide"""
        print("""
Commandes disponibles:
• generate <description> - Génère une fonction
• explain <code> - Explique le fonctionnement d'un code  
• refactor <code> for <goal> - Refactorise un code
• document <code> - Génère la documentation
• improve <code> - Suggère des améliorations
• help - Affiche cette aide
• quit - Quitte l'assistant
        """)
    
    def _handle_generate(self, description):
        """Gestion de la génération de code"""
        print("Génération en cours...")
        code = self.assistant.generate_function(description)
        if code:
            print("\n=== CODE GÉNÉRÉ ===")
            print(code)
        else:
            print("Erreur lors de la génération")
    
    def _handle_explain(self, code):
        """Gestion de l'explication de code"""
        print("Analyse en cours...")
        explanation = self.assistant.explain_code(code)
        print("\n=== EXPLICATION ===")
        print(explanation)
    
    def _handle_refactor(self, code, goal):
        """Gestion de la refactorisation"""
        print("Refactorisation en cours...")
        refactored = self.assistant.refactor_code(code, goal)
        print("\n=== CODE REFACTORISÉ ===")
        print(refactored)
    
    def _handle_document(self, code):
        """Gestion de la documentation"""
        print("Génération de documentation...")
        docs = self.assistant.generate_documentation(code)
        print("\n=== DOCUMENTATION ===")
        print(docs)
    
    def _handle_improve(self, code):
        """Gestion des suggestions d'amélioration"""
        print("Analyse des améliorations...")
        suggestions = self.assistant.suggest_improvements(code)
        print(f"\n=== SUGGESTIONS ({len(suggestions)}) ===")
        for i, suggestion in enumerate(suggestions, 1):
            print(f"{i}. {suggestion}")
    
    def _handle_general_question(self, question):
        """Gestion des questions générales"""
        print("Réflexion en cours...")
        # Ici, on pourrait utiliser l'API pour répondre à des questions générales
        print("Fonctionnalité en développement. Utilisez les commandes spécialisées.")

def demo_ai_assistant():
    """Démonstration de l'assistant IA"""
    
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        print("OPENAI_API_KEY non configurée")
        return
    
    assistant = AICodeAssistant(api_key)
    
    # Exemple de génération de fonction
    print("=== GÉNÉRATION DE FONCTION ===")
    func = assistant.generate_function(
        "une fonction qui calcule le factoriel d'un nombre de manière récursive",
        constraints=["Gérer les cas d'erreur", "Utiliser le typing", "Inclure des tests"]
    )
    print(func)
    
    # Exemple d'explication
    print("\n=== EXPLICATION DE CODE ===")
    code = "def factorial(n): return 1 if n <= 1 else n * factorial(n-1)"
    explanation = assistant.explain_code(code)
    print(explanation)

if __name__ == '__main__':
    # Mode démo
    if len(os.sys.argv) > 1 and os.sys.argv[1] == '--demo':
        demo_ai_assistant()
    else:
        # Mode interactif
        api_key = os.getenv("OPENAI_API_KEY")
        if not api_key:
            print("OPENAI_API_KEY non configurée")
            exit(1)
        
        interactive = InteractiveCodeAssistant(api_key)
        interactive.start_interactive_session()
```

## Section 4 : IA et sécurité

### 4.1 Analyse de sécurité assistée par IA

```python
# ai_security_analyzer.py - Analyseur de sécurité IA
import openai
import os
import re
import ast
from typing import Dict, List, Any, Optional, Tuple
import logging

logger = logging.getLogger(__name__)

class AISecurityAnalyzer:
    """Analyseur de sécurité utilisant l'IA"""
    
    def __init__(self, api_key: str):
        openai.api_key = api_key
        self.model = "gpt-4"
    
    def analyze_code_security(self, code: str, language: str = "python") -> Dict[str, Any]:
        """Analyse de sécurité complète du code"""
        
        prompt = f"""
        Effectue une analyse de sécurité complète de ce code {language}:
        
        ```{language}
        {code}
        ```
        
        Identifie et classe les vulnérabilités selon:
        
        **Vulnérabilités critiques (CVSS 9-10):**
        - Injection (SQL, Command, etc.)
        - RCE (Remote Code Execution)
        - Authentification compromise
        - Escalade de privilèges
        
        **Vulnérabilités élevées (CVSS 7-8.9):**
        - XSS (Cross-Site Scripting)
        - CSRF (Cross-Site Request Forgery)
        - Exposition de données sensibles
        - Vulnérabilités de configuration
        
        **Vulnérabilités moyennes (CVSS 4-6.9):**
        - Déni de service
        - Race conditions
        - Weak cryptography
        - Information disclosure
        
        Pour chaque vulnérabilité trouvée, fournis:
        1. **Description**: Ce qui peut être exploité
        2. **Impact**: Conséquences possibles
        3. **Exploitation**: Comment l'exploiter
        4. **Remédiation**: Comment corriger
        5. **Code sécurisé**: Exemple de correction
        
        Structure la réponse en JSON avec les catégories de sévérité.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en cybersécurité spécialisé dans l'analyse de code. Identifie toutes les vulnérabilités potentielles avec précision."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=2000,
                temperature=0.1
            )
            
            analysis_text = response.choices[0].message.content.strip()
            
            # Parsing du JSON
            import json
            if analysis_text.startswith("```json"):
                analysis_text = analysis_text[7:]
            if analysis_text.endswith("```"):
                analysis_text = analysis_text[:-3]
            
            try:
                analysis = json.loads(analysis_text.strip())
                return analysis
            except json.JSONDecodeError:
                # Si JSON invalide, créer une structure basique
                return {
                    "analysis": analysis_text,
                    "parsed": False,
                    "error": "Format JSON invalide"
                }
            
        except Exception as e:
            logger.error(f"Erreur analyse sécurité: {e}")
            return {"error": str(e)}
    
    def generate_secure_code(self, vulnerable_code: str, vulnerability: str, 
                           language: str = "python") -> str:
        """Génération de code sécurisé"""
        
        prompt = f"""
        Ce code {language} contient une vulnérabilité: {vulnerability}
        
        Code vulnérable:
        ```{language}
        {vulnerable_code}
        ```
        
        Génère une version sécurisée qui:
        1. Corrige la vulnérabilité identifiée
        2. Suit les bonnes pratiques de sécurité
        3. Préserve la fonctionnalité originale
        4. Inclut des validations d'entrée
        5. Utilise des bibliothèques sécurisées
        
        Explique les changements apportés et pourquoi ils améliorent la sécurité.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en sécurisation de code. Génère uniquement des implémentations sécurisées et explique les protections ajoutées."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.1
            )
            
            secure_code = response.choices[0].message.content.strip()
            return secure_code
            
        except Exception as e:
            logger.error(f"Erreur génération code sécurisé: {e}")
            return vulnerable_code
    
    def assess_dependencies_security(self, dependencies: List[str], 
                                   language: str = "python") -> Dict[str, Any]:
        """Évaluation de la sécurité des dépendances"""
        
        deps_text = "\n".join(f"- {dep}" for dep in dependencies)
        
        prompt = f"""
        Évalue la sécurité de ces dépendances {language}:
        
        {deps_text}
        
        Pour chaque dépendance, vérifie:
        1. **Vulnérabilités connues**: CVEs, advisories de sécurité
        2. **Maintenance**: Mises à jour récentes, communauté active
        3. **Trust**: Source fiable, signature des releases
        4. **Alternatives**: Options plus sécurisées si nécessaire
        
        Fournis des recommandations spécifiques pour chaque dépendance problématique.
        
        Structure la réponse avec le niveau de risque pour chaque dépendance.
        """
        
        try:
            response = openai.ChatCompletion.create(
                model=self.model,
                messages=[
                    {"role": "system", "content": "Tu es un expert en sécurité des dépendances logicielles. Évalue les risques et fournis des recommandations pratiques."},
                    {"role": "user", "content": prompt}
                ],
                max_tokens=1500,
                temperature=0.2
            )
            
            assessment = response.choices[0].message.content.strip()
            return {"assessment": assessment, "dependencies": dependencies}
            
        except Exception as e:
            logger.error(f"Erreur évaluation dépendances: {e}")
            return {"error": str(e)}

class AISecurityAuditor:
    """Auditeur de sécurité IA intégré dans les pipelines"""
    
    def __init__(self, api_key: str):
        self.analyzer = AISecurityAnalyzer(api_key)
        self.scan_results = []
    
    def audit_codebase(self, codebase_path: str, language: str = "python") -> Dict[str, Any]:
        """Audit complet d'une base de code"""
        
        findings = {
            "critical": [],
            "high": [],
            "medium": [],
            "low": [],
            "info": []
        }
        
        # Scan des fichiers
        import glob
        pattern = "*.py" if language == "python" else "*.*"
        files = glob.glob(f"{codebase_path}/**/{pattern}", recursive=True)
        
        logger.info(f"Audit de {len(files)} fichiers...")
        
        for file_path in files[:10]:  # Limiter pour la démo
            try:
                with open(file_path, 'r', encoding='utf-8') as f:
                    code = f.read()
                
                if len(code.strip()) < 50:  # Fichiers trop petits
                    continue
                
                logger.info(f"Analyse de {file_path}")
                
                # Analyse de sécurité
                analysis = self.analyzer.analyze_code_security(code, language)
                
                if "error" in analysis:
                    continue
                
                # Classification des findings
                for severity in ["critical", "high", "medium"]:
                    if severity in analysis:
                        for vuln in analysis[severity]:
                            findings[severity].append({
                                "file": file_path,
                                "vulnerability": vuln,
                                "severity": severity
                            })
                
            except Exception as e:
                logger.error(f"Erreur analyse {file_path}: {e}")
        
        # Rapport final
        total_findings = sum(len(vulns) for vulns in findings.values())
        
        report = {
            "summary": {
                "files_scanned": len(files),
                "total_findings": total_findings,
                "critical_count": len(findings["critical"]),
                "high_count": len(findings["high"]),
                "medium_count": len(findings["medium"])
            },
            "findings": findings,
            "recommendations": self._generate_recommendations(findings)
        }
        
        return report
    
    def _generate_recommendations(self, findings: Dict[str, List]) -> List[str]:
        """Génération de recommandations"""
        recommendations = []
        
        if findings["critical"]:
            recommendations.append("🔴 Corriger immédiatement les vulnérabilités critiques")
        
        if findings["high"]:
            recommendations.append("🟠 Planifier la correction des vulnérabilités élevées dans les 30 jours")
        
        if findings["medium"]:
            recommendations.append("🟡 Adresser les vulnérabilités moyennes lors des prochains sprints")
        
        if not any(findings.values()):
            recommendations.append("✅ Aucune vulnérabilité critique détectée")
        
        return recommendations
    
    def generate_security_report(self, report: Dict[str, Any]) -> str:
        """Génération de rapport de sécurité formaté"""
        
        output = []
        output.append("# 🔒 Rapport d'Audit de Sécurité IA")
        output.append("")
        output.append("## 📊 Résumé")
        output.append(f"- **Fichiers analysés**: {report['summary']['files_scanned']}")
        output.append(f"- **Vulnérabilités totales**: {report['summary']['total_findings']}")
        output.append("")
        
        severity_emojis = {
            "critical": "🔴",
            "high": "🟠", 
            "medium": "🟡",
            "low": "🟢",
            "info": "ℹ️"
        }
        
        for severity in ["critical", "high", "medium", "low", "info"]:
            count = len(report['findings'][severity])
            if count > 0:
                output.append(f"- **{severity_emojis[severity]} {severity.title()}**: {count}")
        
        output.append("")
        output.append("## 📋 Recommandations")
        for rec in report['recommendations']:
            output.append(f"- {rec}")
        
        output.append("")
        output.append("## 🔍 Détails des vulnérabilités")
        
        for severity in ["critical", "high", "medium", "low", "info"]:
            vulns = report['findings'][severity]
            if vulns:
                output.append(f"### {severity_emojis[severity]} {severity.title()}")
                for vuln in vulns[:5]:  # Limiter l'affichage
                    output.append(f"**Fichier**: {vuln['file']}")
                    output.append(f"**Vulnérabilité**: {vuln['vulnerability'].get('description', 'N/A')}")
                    output.append("")
        
        return "\n".join(output)

# Exemple d'utilisation
def demo_security_analysis():
    """Démonstration de l'analyse de sécurité IA"""
    
    api_key = os.getenv("OPENAI_API_KEY")
    if not api_key:
        print("OPENAI_API_KEY non configurée")
        return
    
    analyzer = AISecurityAnalyzer(api_key)
    auditor = AISecurityAuditor(api_key)
    
    # Code vulnérable exemple
    vulnerable_code = '''
def authenticate_user(username, password):
    # Vulnérabilité SQL Injection
    query = f"SELECT * FROM users WHERE username='{username}' AND password='{password}'"
    cursor.execute(query)
    return cursor.fetchone()

def execute_command(user_input):
    # Vulnérabilité Command Injection
    os.system(f"echo {user_input}")

def handle_file_upload(file):
    # Vulnérabilité Path Traversal
    with open(f"/uploads/{file.filename}", "w") as f:
        f.write(file.content)
'''
    
    print("=== ANALYSE DE SÉCURITÉ ===")
    analysis = analyzer.analyze_code_security(vulnerable_code)
    print("Analyse terminée")
    
    # Génération de code sécurisé
    print("\n=== CODE SÉCURISÉ ===")
    secure_code = analyzer.generate_secure_code(vulnerable_code, "SQL Injection")
    print(secure_code)

if __name__ == '__main__':
    demo_security_analysis()
```

## Conclusion : L'IA comme catalyseur DevOps

L'intelligence artificielle transforme fondamentalement les pratiques DevOps en automatisant les tâches complexes, en améliorant la qualité du code, et en prévenant les problèmes avant qu'ils ne surviennent. Des détecteurs d'anomalies aux assistants de développement, l'IA devient un partenaire essentiel dans l'accélération et la sécurisation des déploiements.

Dans le prochain chapitre, nous conclurons cette encyclopédie en explorant les tendances émergentes et l'avenir du DevOps.

---

**Exercice pratique :** Créez un système complet d'observabilité IA qui inclut :
1. Détection d'anomalies en temps réel avec ML
2. Prédiction des pannes basée sur l'historique
3. Génération automatique de tests avec IA
4. Optimisation des performances assistée par IA
5. Analyse de sécurité automatisée
6. Assistant de développement interactif

**Challenge avancé :** Développez une plateforme DevOps autonome qui :
- Utilise l'IA pour l'optimisation continue des pipelines
- Prédit et prévient automatiquement les incidents
- Génère du code et des tests automatiquement
- S'adapte dynamiquement à la charge et aux patterns de trafic
- Assure la conformité sécurité automatiquement
- Fournit des insights prédictifs sur les performances et la stabilité

**Réflexion :** Comment l'IA transforme-t-elle le rôle des équipes DevOps ? Quels sont les nouveaux défis éthiques et techniques que cette intégration soulève ? L'IA va-t-elle remplacer les développeurs et les ops, ou les augmentera-t-elle ?

