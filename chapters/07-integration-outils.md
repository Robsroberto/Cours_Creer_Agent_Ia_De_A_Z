## Connexion à des API tierces  

### Comprendre les API REST  

Les API REST (Representational State Transfer) sont le modèle le plus répandu pour exposer des services web. Elles s’appuient sur les verbes HTTP : **GET** pour récupérer des données, **POST** pour en créer, **PUT/PATCH** pour les mettre à jour, **DELETE** pour les supprimer. Les réponses sont généralement encodées en JSON, ce qui les rend faciles à manipuler en Python.  

Dans le contexte africain, on retrouve souvent des API publiques (météo, taux de change, données d’OpenStreetMap) et des services privés (paiement mobile, banques locales). La première étape consiste à lire la documentation : points d’accès, paramètres obligatoires, limites de taux (rate‑limit) et méthode d’authentification.  

### Consommer une API REST avec `requests`  

```python
import os
import requests
from dotenv import load_dotenv

load_dotenv()                     # charge les variables d’environnement depuis .env

API_KEY = os.getenv("OPENWEATHER_KEY")
BASE_URL = "https://api.openweathermap.org/data/2.5/weather"

def get_weather(city: str) -> dict:
    params = {
        "q": city,
        "appid": API_KEY,
        "units": "metric",
        "lang": "fr"
    }
    response = requests.get(BASE_URL, params=params, timeout=5)
    response.raise_for_status()   # lève une exception en cas de code 4xx/5xx
    return response.json()
```

Cet exemple montre comment récupérer la météo d’une ville. Le `timeout` protège votre agent contre les blocages réseau, un point crucial dans des environnements où la latence peut être élevée.  

### Gestion des quotas et limites d’appels  

De nombreuses API imposent un nombre maximal d’appels par minute ou par jour. En Afrique, les fournisseurs de services mobiles (ex. : MTN, Orange) offrent souvent des plans avec des quotas limités.  

* **Stratégie de limitation côté client** : conservez le nombre d’appels effectués dans une variable en mémoire ou dans Redis et refusez les appels supplémentaires jusqu’au rafraîchissement du compteur.  
* **Back‑off exponentiel** : lorsqu’une réponse `429 Too Many Requests` est reçue, attendez `2^n` secondes (où `n` est le nombre de tentatives) avant de réessayer.  

```python
import time

def safe_get(url, **kwargs):
    for attempt in range(5):
        resp = requests.get(url, **kwargs)
        if resp.status_code == 429:
            wait = 2 ** attempt
            print(f"Quota dépassé, attente de {wait}s")
            time.sleep(wait)
            continue
        resp.raise_for_status()
        return resp.json()
    raise RuntimeError("Échec après plusieurs tentatives")
```

### Authentification sécurisée  

Les API utilisent souvent une **clé d’accès** (API key) ou **OAuth 2.0**. La meilleure pratique consiste à **ne jamais** hard‑coder ces secrets dans le code source.  

* **Variables d’environnement** : `export OPENWEATHER_KEY=xxxx` sur le serveur, puis `os.getenv`.  
* **Fichier `.env`** (non versionné) :  
  ```dotenv
  OPENWEATHER_KEY=xxxxxxxxxxxxxxxx
  ```  
* **Vault** : pour les projets plus avancés, HashiCorp Vault ou Azure Key Vault permettent de stocker, de versionner et de faire pivoter les secrets automatiquement.  

> **Voir chapitre 3** pour la mise en place d’un environnement virtuel et la gestion des dépendances.  

### Consommer une API GraphQL  

GraphQL offre une flexibilité supérieure à REST en permettant de spécifier exactement les champs souhaités. De nombreux projets africains (ex. : **Hasura** déployé sur des serveurs locaux) exposent une API GraphQL pour interroger des bases de données.  

```python
import requests

GRAPHQL_ENDPOINT = "https://api.africa-data.org/v1/graphql"
HEADERS = {"Authorization": f"Bearer {os.getenv('HASURA_TOKEN')}"}

def query_cities(country_code: str):
    query = """
    query ($code: String!) {
      cities(where: {country_code: {_eq: $code}}) {
        id
        name
        population
      }
    }
    """
    variables = {"code": country_code}
    payload = {"query": query, "variables": variables}
    resp = requests.post(GRAPHQL_ENDPOINT, json=payload, headers=HEADERS)
    resp.raise_for_status()
    return resp.json()["data"]["cities"]
```

Le **batching** (regroupement de plusieurs requêtes) et le **caching** des réponses peuvent réduire considérablement le nombre d’appels et les coûts associés.  

---

## Interaction avec des bases de données locales  

### Choix entre SQLite et PostgreSQL  

* **SQLite** : fichier unique, aucune configuration serveur, idéal pour des prototypes ou des déploiements légers (ex. : agents fonctionnant sur un Raspberry Pi dans une zone rurale).  
* **PostgreSQL** : serveur robuste, support des transactions, extensions géospatiales (PostGIS) utiles pour des cas d’usage comme la géolocalisation de points de vente.  

En Afrique, la disponibilité d’une connexion fiable influence le choix : privilégiez SQLite quand la connectivité réseau est intermittente, PostgreSQL quand vous avez besoin de scalabilité et de réplication.  

### Connexion avec `sqlite3` et `psycopg2`  

```python
# SQLite
import sqlite3

def init_sqlite(db_path="agent.db"):
    conn = sqlite3.connect(db_path)
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS conversation_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id TEXT,
            message TEXT,
            response TEXT,
            timestamp DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    return conn
```

```python
# PostgreSQL
import psycopg2
import os

def init_postgres():
    conn = psycopg2.connect(
        dbname=os.getenv("PG_DB"),
        user=os.getenv("PG_USER"),
        password=os.getenv("PG_PASSWORD"),
        host=os.getenv("PG_HOST"),
        port=os.getenv("PG_PORT", 5432)
    )
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS conversation_history (
            id SERIAL PRIMARY KEY,
            user_id VARCHAR(64),
            message TEXT,
            response TEXT,
            timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    """)
    conn.commit()
    return conn
```

### Modélisation simple pour l’historique des conversations  

Conserver le contexte d’une conversation permet à l’agent de répondre de façon cohérente. Chaque interaction est stockée avec un `user_id` (numéro de téléphone, identifiant Telegram, etc.).  

```python
def save_message(conn, user_id, message, response):
    cur = conn.cursor()
    cur.execute(
        "INSERT INTO conversation_history (user_id, message, response) VALUES (%s, %s, %s)",
        (user_id, message, response)
    )
    conn.commit()
```

### Utilisation d’un ORM léger (SQLModel)  

SQLModel, basé sur SQLAlchemy, simplifie le mapping objet‑relationnel tout en restant lisible.  

```python
from sqlmodel import Field, Session, SQLModel, create_engine

class Conversation(SQLModel, table=True):
    id: int = Field(default=None, primary_key=True)
    user_id: str
    message: str
    response: str

engine = create_engine("sqlite:///agent.db")
SQLModel.metadata.create_all(engine)

def log_conversation(user_id: str, msg: str, resp: str):
    with Session(engine) as session:
        conv = Conversation(user_id=user_id, message=msg, response=resp)
        session.add(conv)
        session.commit()
```

### Récupération du contexte récent  

```python
def get_last_n_messages(conn, user_id: str, n: int = 5):
    cur = conn.cursor()
    cur.execute(
        """
        SELECT message, response FROM conversation_history
        WHERE user_id = %s
        ORDER BY timestamp DESC
        LIMIT %s
        """,
        (user_id, n)
    )
    return cur.fetchall()
```

Ces cinq derniers messages peuvent être injectés dans le prompt LLM pour maintenir la continuité.  

---

## Intégration de services de messagerie  

### WhatsApp Business API : un canal incontournable  

WhatsApp détient une part de marché élevée en Afrique (plus de 70 % des smartphones). La **WhatsApp Business Cloud API** de Meta permet d’envoyer/recevoir des messages via HTTP.  

**Étapes d’implémentation**  

1. Créez une application Facebook Business et obtenez un **Access Token**.  
2. Configurez un **Webhook** (exemple avec FastAPI) pour recevoir les notifications.  
3. Utilisez l’endpoint `/messages` pour envoyer des réponses.  

```python
from fastapi import FastAPI, Request
import httpx
import os

app = FastAPI()
WHATSAPP_TOKEN = os.getenv("WHATSAPP_TOKEN")
WHATSAPP_NUMBER_ID = os.getenv("WHATSAPP_NUMBER_ID")
WHATSAPP_URL = f"https://graph.facebook.com/v16.0/{WHATSAPP_NUMBER_ID}/messages"

@app.post("/webhook")
async def webhook(request: Request):
    payload = await request.json()
    for entry in payload.get("entry", []):
        for change in entry.get("changes", []):
            value = change.get("value", {})
            messages = value.get("messages", [])
            for msg in messages:
                sender = msg["from"]
                text = msg["text"]["body"]
                # Appeler l’agent IA ici
                response = await generate_reply(sender, text)
                await send_whatsapp(sender, response)
    return {"status": "ok"}

async def send_whatsapp(to: str, text: str):
    headers = {"Authorization": f"Bearer {WHATSAPP_TOKEN}"}
    data = {
        "messaging_product": "whatsapp",
        "to": to,
        "type": "text",
        "text": {"body": text}
    }
    async with httpx.AsyncClient() as client:
        await client.post(WHATSAPP_URL, json=data, headers=headers)
```

Le webhook doit être accessible publiquement ; en Afrique, on utilise souvent **ngrok** en phase de test ou un serveur VPS chez **OVHcloud Africa** pour la production.  

### Bot Telegram avec `python-telegram-bot`  

Telegram est populaire grâce à son API ouverte et à l’absence de frais d’envoi.  

```python
from telegram import Update
from telegram.ext import ApplicationBuilder, ContextTypes, CommandHandler, MessageHandler, filters

TOKEN = os.getenv("TELEGRAM_BOT_TOKEN")

async def start(update: Update, context: ContextTypes.DEFAULT_TYPE):
    await update.message.reply_text("👋 Bienvenue ! Posez votre question.")

async def handle_message(update: Update, context: ContextTypes.DEFAULT_TYPE):
    user_id = str(update.message.from_user.id)
    text = update.message.text
    # Appel à l’agent IA
    answer = await generate_reply(user_id, text)
    await update.message.reply_text(answer)

app = ApplicationBuilder().token(TOKEN).build()
app.add_handler(CommandHandler("start", start))
app.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))

if __name__ == "__main__":
    app.run_polling()
```

Le modèle de polling est suffisant pour des projets modestes. Pour un trafic plus important, **webhook** (via `setWebhook`) minimise la latence et la consommation de bande passante.  

### Gestion des sessions utilisateurs  

Que ce soit WhatsApp ou Telegram, chaque message porte un identifiant unique (`from` ou `user_id`). Conservez cet identifiant dans la base de données afin de :

* **Récupérer le contexte** (voir section précédente).  
* **Appliquer des quotas personnalisés** (ex. : 10 requêtes LLM par jour pour chaque numéro).  

---

## Sécurisation des clés et secrets en production  

### Variables d’environnement & fichier `.env`  

```bash
# .env (à placer à la racine, jamais versionné)
POSTGRES_PASSWORD=SuperSecret123!
WHATSAPP_TOKEN=EAAJZ... (long token)
```

Chargez‑les avec `python-dotenv` : `load_dotenv()` au démarrage de l’application.  

### Utilisation d’un gestionnaire de secrets  

Pour des déploiements Docker ou Kubernetes, **Docker secrets**, **Kubernetes Secrets** ou **HashiCorp Vault** offrent une couche supplémentaire : les secrets ne sont jamais écrits sur le disque et sont injectés en mémoire au moment du lancement.  

```bash
# Exemple d’injection Vault dans Docker
docker run -e VAULT_ADDR=https://vault.africa \
           -e VAULT_TOKEN=$(cat /run/secrets/vault-token) \
           -e DB_PASSWORD=$(vault kv get -field=password secret/postgres) \
           my-agent-image
```

### Rotation et