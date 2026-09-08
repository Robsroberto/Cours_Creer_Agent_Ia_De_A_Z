## Architecture d’un agent conversationnel

Un agent conversationnel repose sur une **chaîne de traitement** qui transforme l’entrée utilisateur en une réponse pertinente. La chaîne se compose généralement de :

1. **Ingestion** – capture du texte (ou de la voix) et normalisation.
2. **Gestion du contexte** – mémorisation des échanges précédents.
3. **Raisonnement** – appel du LLM ou d’un module de logique.
4. **Actions externes** – recherche web, calculs, requêtes DB.
5. **Synthèse** – construction du texte final à renvoyer.

Chaque maillon doit être découpé en composant réutilisable afin de faciliter les tests, le versionnage et le passage à l’échelle. LangChain fournit déjà des abstractions (`PromptTemplate`, `LLMChain`, `AgentExecutor`) qui correspondent à ces étapes.

### Chaîne de traitement avec LangChain

```python
from langchain import PromptTemplate, LLMChain
from langchain.llms import OpenAI
from langchain.agents import AgentExecutor, Tool, initialize_agent

# 1. Prompt template
prompt = PromptTemplate(
    input_variables=["history", "question"],
    template="""
Historique de la conversation :
{history}

Question de l'utilisateur :
{question}

Répondez de façon concise et proposez une action si nécessaire.
"""
)

# 2. LLM
llm = OpenAI(model="gpt-4o-mini", temperature=0.2)

# 3. Chaîne principale
qa_chain = LLMChain(prompt=prompt, llm=llm)
```

Dans cet exemple, le `PromptTemplate` incorpore le **historique** (`history`) et la **question** actuelle. Le `LLMChain` exécute le prompt à chaque appel. La suite du chapitre montre comment enrichir ce pipeline avec de la mémoire et des outils.

## Gestion du contexte : mémoire à court et long terme

### Mémoire à court terme (short‑term memory)

Le **court terme** conserve les quelques derniers tours de dialogue afin que le modèle puisse s’appuyer sur le fil conducteur. LangChain propose `ConversationBufferMemory` :

```python
from langchain.memory import ConversationBufferMemory

short_memory = ConversationBufferMemory(k=5)  # garde les 5 derniers échanges
```

`k` définit la profondeur de la fenêtre. Pour un chatbot de service client, 5 tours suffisent généralement à garder la cohérence sans alourdir le prompt.

### Mémoire à long terme (long‑term memory)

Certaines informations doivent persister au-delà de la session : préférences utilisateur, historique d’achat, ou réponses à des questions fréquentes. Deux stratégies sont courantes :

| Stratégie | Stockage | Exemple d’usage |
|-----------|----------|-----------------|
| **Vector Store** | Base de vecteurs (FAISS, Chroma, Milvus) | Recherche de documents juridiques ou de législation locale |
| **Base relationnelle** | PostgreSQL, SQLite | Historique des tickets d’assistance, profil client |

#### Exemple : Mémoire vectorielle avec Chroma

```python
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings

embeddings = OpenAIEmbeddings()
vector_db = Chroma(collection_name="knowledge_base", embedding_function=embeddings)

def add_document(text: str, metadata: dict):
    vector_db.add_texts([text], metadatas=[metadata])

def retrieve(query: str, top_k: int = 3):
    return vector_db.similarity_search(query, k=top_k)
```

Lorsqu’une question porte sur une réglementation fiscale nigériane, on interroge le vecteur pour récupérer les passages pertinents, puis on les injecte dans le prompt.

### Fusion des mémoires

LangChain permet de combiner plusieurs sources de mémoire grâce à `CombinedMemory` :

```python
from langchain.memory import CombinedMemory, ConversationBufferMemory, VectorStoreRetrieverMemory

long_memory = VectorStoreRetrieverMemory(retriever=vector_db.as_retriever())
combined_memory = CombinedMemory(memories=[short_memory, long_memory])
```

Le `combined_memory` renvoie d’abord les échanges récents, puis les documents pertinents, offrant ainsi un contexte riche sans surcharge.

## Implémentation d’un agent LangChain

Un **agent** est un orchestrateur qui décide, à chaque tour, s’il faut :

* répondre directement,
* appeler une fonction externe (outil),
* demander une clarification.

### Définition des outils (actions)

#### 1. Recherche web

```python
import requests
from bs4 import BeautifulSoup
from langchain.tools import Tool

def web_search(query: str) -> str:
    url = f"https://duckduckgo.com/html/?q={query}"
    resp = requests.get(url, headers={"User-Agent": "Mozilla/5.0"})
    soup = BeautifulSoup(resp.text, "html.parser")
    results = soup.select(".result__a")[:3]
    return "\n".join([r.get_text() for r in results])

search_tool = Tool(
    name="WebSearch",
    func=web_search,
    description="Effectue une recherche sur le web et renvoie les titres des trois premiers résultats."
)
```

Cette fonction fonctionne même avec des requêtes en français ou en langues locales (ex. swahili), ce qui est utile pour des projets panafricains.

#### 2. Calcul mathématique

```python
import math
from langchain.tools import Tool

def calculator(expression: str) -> str:
    try:
        # utilisation sécurisée de eval via math module
        allowed_names = {k: getattr(math, k) for k in dir(math) if not k.startswith("_")}
        allowed_names["abs"] = abs
        result = eval(expression, {"__builtins__": {}}, allowed_names)
        return str(result)
    except Exception as e:
        return f"Erreur de calcul : {e}"

calc_tool = Tool(
    name="Calculator",
    func=calculator,
    description="Évalue une expression mathématique simple, par ex. 'sqrt(16) + 5'."
)
```

Le sandboxing de `eval` empêche l’exécution de code arbitraire tout en conservant la souplesse nécessaire aux calculs financiers ou scientifiques.

#### 3. Accès à une base de données PostgreSQL

```python
import psycopg2
from langchain.tools import Tool

def query_db(sql: str) -> str:
    conn = psycopg2.connect(
        dbname="support",
        user="agent",
        password="********",
        host="db.empiredesweb.africa"
    )
    cur = conn.cursor()
    cur.execute(sql)
    rows = cur.fetchall()
    cur.close()
    conn.close()
    return "\n".join([", ".join(map(str, r)) for r in rows])

db_tool = Tool(
    name="DatabaseQuery",
    func=query_db,
    description="Exécute une requête SQL en lecture seule sur la base de tickets d’assistance."
)
```

En production, on ajoutera une couche d’autorisation et de limitation du nombre de lignes retournées pour éviter les abus.

### Construction de l’agent

```python
tools = [search_tool, calc_tool, db_tool]

agent = initialize_agent(
    tools=tools,
    llm=llm,
    agent_type="zero-shot-react-description",  # agent qui décide d’appeler un outil ou de répondre
    memory=combined_memory,
    verbose=True
)
```

`zero-shot-react-description` utilise le raisonnement **ReAct** : le LLM génère un plan (penser → agir → observer) et le moteur exécute les actions requises. Le paramètre `verbose` affiche chaque étape, pratique pour le débogage.

## Gestionnaire de dialogue (Dialogue Manager)

Le gestionnaire orchestre le flux complet : réception du message, mise à jour du contexte, appel de l’agent, post‑traitement de la réponse. Une implémentation simple en FastAPI :

```python
from fastapi import FastAPI, Request
from pydantic import BaseModel

app = FastAPI()

class Message(BaseModel):
    user_id: str
    content: str

@app.post("/chat")
async def chat(msg: Message):
    # 1. Récupérer l'historique depuis la mémoire courte
    history = short_memory.load_memory_variables({})["history"]
    
    # 2. Exécuter l'agent
    response = agent.run({"question": msg.content, "history": history})
    
    # 3. Mettre à jour la mémoire courte
    short_memory.save_context({"input": msg.content}, {"output": response})
    
    # 4. Retourner la réponse
    return {"answer": response}
```

Ce point d’entrée peut être consommé par un front‑end React, un bot WhatsApp (via Twilio) ou un service USSD, ce qui répond aux besoins variés des développeurs africains.

### Gestion des interruptions et des clarifications

Lorsqu’un LLM n’est pas sûr, il génère souvent une **demande de clarification**. Le gestionnaire doit détecter les phrases du type « Pouvez‑vous préciser ? » et les renvoyer à l’utilisateur sans passer par les outils :

```python
def is_clarification(text: str) -> bool:
    cues = ["précisez", "plus de détails", "clarifiez", "expliquez"]
    return any(cue in text.lower() for cue in cues)

# Dans le endpoint /chat
if is_clarification(response):
    # On ne met pas à jour la mémoire courte, on attend la réponse de l'utilisateur
    return {"answer": response, "awaiting_user": True}
```

Cette logique évite de polluer la mémoire avec des tours incomplets et améliore la fluidité de la conversation.

## Enrichir l’agent avec des compétences locales

### Utilisation de modèles de langue africains

Les modèles comme **Mistral‑7B‑Instruct** ou **LLaMA‑2‑7B** peuvent être fine‑tuned sur des corpus francophones africains (articles de presse, législation locale). L’appel se fait via `HuggingFaceHub` :

```python
from langchain.llms import HuggingFaceHub

african_llm = HuggingFaceHub(repo_id="mistralai/Mistral-7B-Instruct-v0.2", model_kwargs={"temperature":0.1})
```

En remplaçant `llm` par `african_llm` dans l’agent, on obtient des réponses plus adaptées aux références culturelles et aux termes juridiques locaux.

### Intégration de services de paiement mobile

Dans de nombreux pays d’Afrique de l’Ouest, les paiements se font via **M-Pesa** ou **Orange Money**. Un outil dédié peut être ajouté :

```python
def initiate_payment(user_id: str, amount: float) -> str:
    # Appel fictif à l'API de M-Pesa
    payload = {"user": user_id, "amount": amount}
    resp = requests.post("https://api.mpesa.africa/pay", json=payload)
    return resp.json().get("status", "Échec")

payment_tool = Tool(
    name="MobilePayment",
    func=initiate_payment,
    description="Déclenche un paiement mobile via M-Pesa pour le client identifié."
)
```

Après l’ajout à la liste `tools`, l’agent pourra proposer de régler automatiquement une facture ou un abonnement.

## Tests unitaires et simulation de conversations

Un bon agent doit être **testable**. LangChain expose les chaînes sous forme de fonctions, ce qui facilite l’écriture de tests avec `pytest`.

```python
import pytest

def test_web_search():
    result = web_search("capital du Sénégal")
    assert "Dakar" in result

def test_agent_simple_question():
    # Simuler un court historique vide
    short_memory.clear()
    answer = agent.run({"question": "Quel est le taux de TVA au Cameroun ?", "history": ""})
    assert "19%" in answer  # valeur fictive à ajuster
```

Pour des scénarios plus longs, on utilise des **scripts de dialogue** :

```python
dialogue = [
    ("Quel est le taux de change du franc CFA ?", "Le taux actuel est..."),
    ("Peux‑tu convertir 1000 XOF en EUR ?", "1000 XOF ≈ 1,52 EUR."),
]

for user_msg, expected in dialogue:
    resp = agent.run({"question": user_msg, "history": short_memory.load_memory_variables({})["history"]})
    assert expected in resp
    short_memory.save_context({"input": user_msg}, {"output": resp})
```

Ces tests garantissent que les outils sont correctement invoqués et que le contexte est bien préservé.

## Optimisation des performances

### Batching des appels LLM

Lorsque plusieurs utilisateurs envoient des requêtes simultanément, il est plus efficace de **batcher** les prompts :

```python
from langchain.llms import OpenAI
from langchain.prompts import PromptTemplate

batch_llm = OpenAI(model="gpt-4o-mini", temperature=0.2, batch_size=8)

def batch_respond(prompts: list[dict]):
    return batch_llm.generate(prompts)
```

Le `batch_size` réduit le nombre d’appels réseau et diminue les coûts d’inférence, crucial pour les projets à budget limité.

### Caching des résultats d’outils

Les recherches web ou les requêtes DB peuvent être **mise en cache** avec `functools.lru_cache` ou un store Redis :

```python
from functools import lru_cache

@lru_cache(maxsize=256)
def cached_web_search(query: str) -> str:
    return web_search(query)
```

Le cache évite les appels redondants, améliore la latence et limite les quotas d’API externes.

## Sécurité et conformité

- **Sanitisation des entrées** : toujours nettoyer les prompts avant de les injecter dans des requêtes SQL ou `eval`.
- **Masquage des données sensibles** : lors du logging, masquer les numéros de téléphone, les identifiants bancaires (`***`).
- **RGPD et législation locale** : stocker les historiques uniquement avec le consentement explicite, prévoir une fonction de suppression (`delete_user_data(user_id)`).

Ces bonnes pratiques sont indispensables pour déployer un agent fiable dans les marchés africains où la protection des données devient de plus en plus réglementée.

## Points clés

- Une chaîne de traitement claire (ingestion → contexte → raisonnement → actions → synthèse) garantit la maintenabilité.
- Combinez **mémoire courte** (fenêtre de conversation) et **mémoire longue** (vecteurs ou bases relationnelles) pour un contexte riche et persistant.
- LangChain simplifie la création d’**outils** (recherche web, calcul, DB) et la décision d’appel via le pattern **ReAct**.
- Un **gestionnaire de dialogue** centralise la logique de réception, mise à jour du contexte et réponse, tout en gérant les clarifications.
- Enrichissez l’agent avec des modèles locaux et des services régionaux (paiement mobile, législation) pour une pertinence culturelle.
- Testez chaque composant (outils, chaînes, agent) avec `pytest` et des scripts de conversation afin d’assurer la robustesse.
- Optimisez le débit grâce au **batching**, au **caching** et