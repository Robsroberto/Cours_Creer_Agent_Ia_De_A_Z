## Surveillance de l’agent en production  

### Centralisation des logs  

En production, chaque composant de l’agent (API FastAPI, workers LangChain, tâches asynchrones) doit écrire des logs structurés.  
```python
import logging, json, sys
from pythonjsonlogger import jsonlogger

logger = logging.getLogger("agent")
handler = logging.StreamHandler(sys.stdout)
formatter = jsonlogger.JsonFormatter(
    "%(asctime)s %(levelname)s %(name)s %(message)s %(module)s %(funcName)s %(lineno)d"
)
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Exemple d’utilisation
logger.info("Requête reçue", extra={"user_id": 42, "intent": "weather_query"})
```
Les logs JSON sont faciles à ingérer dans **ELK** (Elasticsearch‑Logstash‑Kibana) ou **Grafana Loki**.  
- **Logstash** ou **Promtail** récupèrent les flux stdout des conteneurs Docker.  
- **Kibana** ou **Grafana** permettent de filtrer par `user_id`, `intent` ou `level`.  

Pour les développeurs africains, **Grafana Cloud** propose un plan gratuit qui suffit largement à monitorer quelques dizaines de services sans frais supplémentaires.

### Métriques d’exploitation  

Les métriques clés à suivre sont :  

| Métrique | Description | Exemple de collecte |
|----------|-------------|----------------------|
| `request_latency_seconds` | Temps moyen de réponse d’une requête | `prometheus_client.Histogram` |
| `error_rate` | Pourcentage de réponses 5xx | `Counter` + `Gauge` |
| `tokens_per_minute` | Volume de tokens consommés (coût) | `Counter` |
| `active_sessions` | Nombre de conversations simultanées | `Gauge` |

```python
from prometheus_client import Counter, Histogram, Gauge, start_http_server

REQUEST_LATENCY = Histogram("request_latency_seconds", "Latency of API calls")
ERROR_COUNTER = Counter("error_total", "Number of errors", ["code"])
TOKENS = Counter("tokens_used", "Tokens consumed")
ACTIVE_SESSIONS = Gauge("active_sessions", "Current active user sessions")

start_http_server(8000)   # expose /metrics
```
Exposez `/metrics` via le même conteneur que l’API ou via un side‑car. Prometheus scrappe régulièrement ces endpoints et alimente Grafana pour visualiser les tendances.

## Alertes et seuils  

### Définir les seuils critiques  

| Métrique | Seuil d’alerte | Action |
|----------|----------------|--------|
| `request_latency_seconds` (p95) | > 1 s | Slack + SMS |
| `error_rate` (5xx) | > 2 % pendant 5 min | PagerDuty / WhatsApp |
| `tokens_used` (heure) | > 80 % du budget | Notification finance |

### Configuration d’Alertmanager  

```yaml
# alertmanager.yml
route:
  receiver: "team"
receivers:
- name: "team"
  slack_configs:
  - api_url: "https://hooks.slack.com/services/XXXXX/XXXXX/XXXXX"
    channel: "#incidents"
  webhook_configs:
  - url: "https://api.whatsapp.com/send?phone=+22512345678"
```
Les alertes sont déclenchées par Prometheus :

```yaml
# prometheus.yml (excerpt)
rule_files:
  - "rules.yml"
```

```yaml
# rules.yml
groups:
- name: agent.rules
  rules:
  - alert: HighLatency
    expr: histogram_quantile(0.95, request_latency_seconds_bucket) > 1
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Latence élevée sur l’agent"
      description: "La latence 95e percentile dépasse 1 s."
```

## Gestion des incidents  

### Playbook d’incident  

1. **Détection** – Alertmanager notifie le canal Slack et le groupe WhatsApp.  
2. **Triage** – Le responsable on‑call consulte les logs (`kubectl logs …`) et les métriques Grafana.  
3. **Isolation** – Si le problème provient d’un micro‑service, on le redémarre (`kubectl rollout restart deployment/agent‑api`).  
4. **Mitigation** – Si le modèle consomme trop de tokens, on bascule temporairement sur le **mode « lite »** (voir chapitre 5).  
5. **Résolution** – Correction du bug ou rollback du déploiement.  
6. **Post‑mortem** – Documenter le timeline, les causes racines et les actions préventives dans Confluence ou Notion.  

### Exemple de run‑book (extrait)  

```
# Run‑book – Latence > 2 s pendant > 5 min
1. Vérifier le tableau de bord Grafana → `request_latency_seconds`.
2. Exécuter : kubectl top pods – namespace=agent
   - Si CPU > 80 % → envisager un scaling horizontal.
3. Lire les 200 dernières lignes du log du pod concerné.
4. Si le log montre “LLM inference timeout”, augmenter le timeout FastAPI.
5. Déployer le correctif via le pipeline CI/CD (voir section CI/CD).
```

## Mise à jour du modèle sans downtime  

### Stratégie Blue‑Green  

- **Blue** : version actuelle en production.  
- **Green** : nouvelle version (nouveau checkpoint, nouvelles fonctions).  

Déploiement :

```yaml
# docker‑compose.yml (simplifié)
services:
  agent-blue:
    image: registry.example.com/agent:1.0
    ports:
      - "8080:80"
    labels:
      - "traefik.http.routers.agent.rule=Host(`api.monagent.africa`) && PathPrefix(`/v1`)"
      - "traefik.http.services.agent.loadbalancer.server.port=80"
  agent-green:
    image: registry.example.com/agent:1.1
    ports:
      - "8081:80"
    labels:
      - "traefik.http.routers.agent-green.rule=Host(`api.monagent.africa`) && PathPrefix(`/v2`)"
      - "traefik.http.services.agent-green.loadbalancer.server.port=80"
```

Traefik (ou Nginx) dirige le trafic **v1** vers `agent-blue`. Une fois les tests de santé validés, on change la règle pour pointer `v1` vers `agent-green`. Aucun client ne subit d’interruption.

### Canary Release avec Kubernetes  

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: agent
        image: registry.example.com/agent:1.1
        env:
        - name: MODEL_VERSION
          value: "v1.1"
```

Kubernetes met à jour un pod à la fois, surveille les métriques (via l’annotation `prometheus.io/scrape`) et, si aucune alerte n’est déclenchée, poursuit le rollout.

### Versionning du modèle  

Utilisez **MLflow** ou un registre simple dans un bucket S3 / MinIO :

```bash
mlflow models register -m s3://models/agent/v1.1 -n agent-model
mlflow models transition-stage -r 2 -s Staging
```

Le service charge le modèle à partir du tag **Production**, ce qui rend le basculement instantané.

## Boucle de feedback et amélioration continue  

### Capture des retours utilisateurs  

Dans le code de l’agent, ajoutez un endpoint qui accepte les corrections :

```python
@app.post("/feedback")
async def receive_feedback(feedback: Feedback):
    # Persist dans PostgreSQL
    async with async_session() as session:
        session.add(FeedbackORM(**feedback.dict()))
        await session.commit()
    return {"status": "recorded"}
```

Les données sont ensuite exploitées pour :  

- **Fine‑tuning incrémental** (ex. LoRA) sur les dialogues où le modèle a échoué.  
- **Analyse de couverture linguistique** afin d’identifier les langues sous‑représentées (voir chapitre 4).  

### Apprentissage actif  

Programme quotidien :  

1. Sélectionner les 100 % des interactions avec le plus faible score de confiance (`response_confidence < 0.3`).  
2. Envoyer ces extraits à des annotateurs locaux (via une interface web simple).  
3. Retrainer le modèle pendant la nuit et publier la version `vX.Y+1`.  

## Évolution fonctionnelle de l’agent  

### Ajout de nouvelles langues locales  

1. **Tokenisation** – Utilisez **SentencePiece** entraîné sur un corpus Swahili ou Wolof.  
2. **Fine‑tuning** – Appliquez le même script de LoRA que pour le français, mais avec le jeu de données multilingue.  

```bash
python finetune.py \
  --model LLaMA-7B \
  --train_file data/sw_swahili.jsonl \
  --output_dir models/agent_sw \
  --lora_r 8
```

3. **Routing dynamique** – Dans le chain LangChain, insérez une étape de détection de langue (fasttext) et redirigez vers le sous‑modèle approprié.

```python
def route_by_language(text):
    lang = fasttext.predict(text)[0]
    if lang == "sw":
        return "agent_sw"
    return "agent_fr"
```

### Extension fonctionnelle (ex. paiement mobile)  

- Créez un **plugin** qui expose une fonction `process_payment(amount, phone_number)`.  
- Enregistrez‑le dans le **ToolStore** de LangChain (voir chapitre 6).  
- Autorisez les appels via un scope OAuth : les utilisateurs qui ont le rôle `payer` peuvent invoquer la fonction.

```python
from langchain.tools import BaseTool

class MobilePaymentTool(BaseTool):
    name = "mobile_payment"
    description = "Effectue un paiement via Mobile Money"

    def _run(self, amount: float, phone: str):
        # Appel à l’API de l’opérateur (MTN, Orange)
        response = requests.post("https://api.mtn.com/pay", json={"amount": amount, "phone": phone})
        return response.json()
```

## Sécurité, conformité et bonnes pratiques  

- **Rotation des secrets** : utilisez **HashiCorp Vault** ou **AWS Secrets Manager**. Intégrez le dans le démarrage du conteneur (`vault kv get secret/agent`).  
- **Audit logs** : chaque appel à l’API doit être signé (JWT) et enregistré avec l’ID utilisateur, l’heure et le hash de la requête.  
- **RGPD et protection des données locales** : stockez les données personnelles (numéros de téléphone, adresses) dans un schéma chiffré (`pgcrypto`).  
- **Limitation de débit** : `fastapi-limiter` avec Redis pour éviter les abus, notamment sur les endpoints de génération de texte qui sont coûteux.

## CI/CD orienté production continue  

### Pipeline GitHub Actions (exemple)  

```yaml
name: CI/CD Agent

on:
  push:
    branches: [ main ]

jobs:
  build-test-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: |
          python -m venv venv
          source venv/bin/activate
          pip install -r requirements.txt

      - name: Lint & Test
        run: |
          flake8 .
          pytest -q

      - name: Build Docker image
        run: |
          docker build -t registry.example.com/agent:${{ github.sha }} .
          docker push registry.example.com/agent:${{ github.sha }}

      - name: Deploy to Kubernetes
        uses: azure/k8s-deploy@v4
        with:
          manifests: |
            k8s/deployment.yml
          images: |
            registry.example.com/agent:${{ github.sha }}
          namespace: production
```

Le pipeline exécute les tests, construit une image Docker, la pousse dans un registre privé, puis déclenche un **rolling update** Kubernetes. En cas d’échec, l’étape `Deploy` est annulée et l’on garde la version précédente.

## Outils adaptés aux contextes africains  

| Besoin | Solution locale / low‑cost |
|--------|---------------------------|
| Monitoring | Grafana Cloud (free tier) + Loki |
| Stockage d’artefacts | **Scaleway Object Storage** (tarifs européens mais proche) |
| Base de données | **Supabase** (PostgreSQL hébergé, plan gratuit) |
| Notification | **WhatsApp Business API** via Twilio ou **SMS Gateway** local (MTN, Orange) |
| CI/CD | **GitHub Actions** (minutes gratuites) ou **GitLab CI** auto‑hébergé sur un VPS OVH |

Ces services offrent une latence raisonnable pour les utilisateurs d’Afrique de l’Ouest et du Centre, tout en limitant les coûts d’infrastructure.

## Points clés  

- **Logs structurés + agrégation** (ELK / Loki) permettent d’analyser rapidement les incidents.  
- **Métriques** (latence, erreurs, tokens) sont exposées via Prometheus et visualisées dans Grafana.  
- **Alertes** configur