## Qu’est‑ce qu’un agent IA ?  

Un **agent IA** est un logiciel capable d’interpréter des entrées humaines (texte, voix, images) et de produire des réponses ou des actions de façon autonome.  
Contrairement à un simple script qui exécute une tâche pré‑définie, l’agent possède :

* **Un modèle de langage** qui comprend le sens du texte et génère du contenu naturel.  
* **Une mémoire contextuelle** qui garde en mémoire les échanges précédents pour assurer la cohérence.  
* **Des capacités d’action** (appel d’API, mise à jour de bases de données, envoi de messages) qui lui permettent de passer de la parole à l’acte.  

### Définition et caractéristiques  

| Caractéristique | Description | Exemple concret |
|-----------------|-------------|-----------------|
| **Compréhension** | Analyse sémantique du texte, détection d’intentions. | Un agriculteur écrit « Comment protéger mes maïs contre la rouille ? » et l’agent identifie l’intention « conseil phytosanitaire ». |
| **Génération** | Production d’une réponse naturelle, adaptée au ton demandé. | Réponse sous forme de texte court, emoji ou même audio. |
| **Mémoire** | Stockage temporaire (session) ou permanente (profil utilisateur). | Historique des commandes d’un client e‑commerce. |
| **Action** | Interaction avec des services externes (API, bases de données). | Envoi d’une requête à l’API du service météo pour afficher la prévision locale. |
| **Adaptabilité** | Possibilité de ré‑entraîner ou de fine‑tuner le modèle sur des données spécifiques. | Enrichir le vocabulaire d’un bot de santé avec des termes médicaux locaux. |

### Différence avec un chatbot classique  

Un *chatbot* se contente souvent de suivre un arbre de décision ou des réponses pré‑enregistrées. Un agent IA, quant à lui, exploite un **grand modèle de langage (LLM)** qui lui donne une compréhension et une créativité bien supérieures, tout en pouvant **exécuter des actions réelles** (paiement, mise à jour de planning, etc.). Cette capacité à **agir** le distingue clairement d’un simple chatbot.

---

## Cas d’usage pertinents en Afrique francophone  

### Assistant personnel multilingue  

De nombreux utilisateurs en Afrique utilisent plusieurs langues (français, anglais, langues locales comme le wolof, le swahili, le bambara). Un agent IA multilingue peut :

* Gérer les agendas, envoyer des rappels SMS, proposer des itinéraires en fonction du trafic de Lagos ou de Dakar.  
* Traduire en temps réel des messages entre langues, facilitant les échanges entre agriculteurs et experts.

```python
# Exemple d’appel à l’API OpenAI pour traduire du français vers le wolof
import openai, os
openai.api_key = os.getenv("OPENAI_API_KEY")

def traduire(fr_text):
    response = openai.ChatCompletion.create(
        model="gpt-4o-mini",
        messages=[{"role":"system","content":"Tu traduis du français vers le wolof."},
                  {"role":"user","content":fr_text}]
    )
    return response.choices[0].message.content.strip()

print(traduire("Quel est le prix du maïs aujourd'hui ?"))
```

### Service client pour e‑commerce local  

Les plateformes comme **Jumia**, **Konga** ou les boutiques Shopify africaines ont besoin d’un service client disponible 24 h/24, capable de :

* Répondre aux questions fréquentes (livraison, retours).  
* Vérifier le statut d’une commande en interrogeant l’API du magasin.  
* Proposer des produits complémentaires en fonction du panier.

### Automatisation de processus métiers  

Dans le secteur agricole, les coopératives collectent chaque jour des données sur les récoltes, les intrants et les ventes. Un agent IA peut :

* Recevoir les rapports SMS des agriculteurs, les normaliser et les stocker dans une base de données.  
* Générer automatiquement des alertes météo ou des recommandations d’irrigation.  
* Produire des rapports hebdomadaires synthétisés pour les responsables.

### Bot d’information santé ou éducation  

Dans les zones rurales où l’accès aux professionnels de santé est limité, un agent IA peut :

* Répondre à des questions courantes (symptômes, prévention du paludisme).  
* Orienter l’utilisateur vers le centre de santé le plus proche grâce à la géolocalisation.  
* Diffuser des contenus éducatifs (cours de français, de mathématiques) en format texte ou audio.

---

## Composants d’un agent IA  

### Modèle de langage (LLM)  

Le cœur de l’agent est le **LLM** qui transforme le texte d’entrée en représentation sémantique et génère la réponse. Deux approches sont possibles :

* **Open‑source** : LLaMA 2, Mistral, Falcon – hébergés sur vos serveurs pour garder la souveraineté des données.  
* **SaaS** : OpenAI GPT‑4, Anthropic Claude – faciles à intégrer via API, mais nécessitent une connexion internet stable.

### Gestion du dialogue et du contexte  

Un gestionnaire de dialogue orchestre les tours de conversation :

1. **Analyse d’intention** – classification du besoin (question, commande, feedback).  
2. **Mise à jour du contexte** – stockage des variables (nom, localisation, préférences).  
3. **Sélection de la réponse** – appel au LLM avec le contexte enrichi.

```python
# Schéma simplifié d’un gestionnaire de dialogue
class DialogueManager:
    def __init__(self):
        self.session = {}

    def update(self, user_id, key, value):
        self.session.setdefault(user_id, {})[key] = value

    def build_prompt(self, user_id, user_message):
        ctx = self.session.get(user_id, {})
        return f"Contexte: {ctx}\nUtilisateur: {user_message}\nRéponse:"
```

### Mémoire et persistance  

* **Mémoire courte** (session) : conserve les échanges pendant la conversation en cours.  
* **Mémoire longue** (profil) : stocke les informations persistantes dans une base (ex. PostgreSQL) pour personnaliser les interactions futures.

### Interfaces d’entrée / sortie  

| Canal | Exemple d’intégration | Usage typique |
|-------|----------------------|---------------|
| **API REST** | FastAPI, Flask | Appels depuis une application mobile ou un site web. |
| **Messagerie instantanée** | WhatsApp Business API, Telegram Bot API | Interaction via smartphone, très répandu en Afrique. |
| **Voice** | Twilio Voice, Google Speech‑to‑Text | Assistance vocale pour les agriculteurs en terrain. |
| **Web UI** | Streamlit, Gradio | Prototype rapide pour démonstrations internes. |

---

## Panorama technologique actuel  

### Modèles open‑source  

| Modèle | Taille | Licence | Points forts pour l’Afrique |
|--------|--------|---------|-----------------------------|
| **LLaMA 2** | 7 B – 70 B | Meta (non‑commercial) | Performances élevées, possibilité d’hébergement local. |
| **Mistral‑7B** | 7 B | Apache 2.0 | Optimisé pour les GPU modestes, bon rapport coût/performances. |
| **Falcon‑180B** | 180 B | Apache 2.0 | État‑de‑l’art, mais nécessite des serveurs puissants (cloud). |

### Services SaaS  

* **OpenAI** – large éventail de modèles, facturation à la token, disponibilité globale.  
* **Anthropic** – modèle orienté sécurité, idéal pour les applications sensibles (santé).  
* **Google Gemini** – intégration aisée avec les services GCP, bonnes performances multilingues.  

### Frameworks de construction  

| Framework | Langage | Fonctionnalité phare |
|-----------|---------|----------------------|
| **LangChain** | Python | Chaînes de prompts, intégration d’outils externes, mémoire. |
| **LlamaIndex** | Python | Indexation de documents locaux, recherche contextuelle. |
| **Botpress** | JavaScript/TypeScript | Plateforme low‑code pour créer des bots conversationnels. |
| **Rasa** | Python | Gestion du dialogue avec NLU personnalisable, open‑source complet. |

### Outils de déploiement  

* **Docker** – conteneurisation pour garantir la même exécution sur un serveur à Nairobi ou à Abidjan.  
* **Serverless** (AWS Lambda, Azure Functions, Cloudflare Workers) – facturation à l’usage, adaptée aux projets à budget limité.  
* **Plateformes locales** – OVHcloud (France, mais avec data‑center à Paris), Hetzner (Allemagne) offrent des latences raisonnables pour l’Afrique de l’Ouest.

---

## Positionnement de l’agent IA dans l’écosystème IA  

### Niveau de la pile : données → modèle → application  

1. **Collecte de données** – textes, logs, enregistrements vocaux provenant de sources locales.  
2. **Pré‑traitement** – nettoyage, tokenisation, anonymisation (respect de la vie privée).  
3. **Entraînement / fine‑tuning** – adaptation du LLM aux spécificités linguistiques (ex. expressions sénégalaises).  
4. **Intégration** – mise en place du gestionnaire de dialogue, des API et de la persistance.  
5. **Déploiement** – conteneurisation, monitoring, scalabilité.

### Interaction avec d’autres IA  

Un agent IA peut être **complémenté** par :

* **Vision** – reconnaissance d’images de parasites sur les cultures (via un modèle de classification).  
* **Recommandation** – suggestion de produits agricoles basés sur l’historique d’achat.  
* **Analyse prédictive** – prévision de rendements grâce à des modèles de séries temporelles.

### Sécurité, éthique et souveraineté des données  

* **Chiffrement** des communications (HTTPS, TLS).  
* **Conformité** au RGPD et aux lois locales sur la protection des données (ex. loi nigériane sur la protection des données).  
* **Souveraineté** – privilégier le déploiement on‑premise ou sur des data‑centers situés en Europe ou en Afrique pour éviter le transfert non contrôlé des données sensibles.

---

## Objectifs du cours « Créer votre propre Agent IA de A à Z »  

### Maîtriser le cycle complet  

* **Conception** – identifier le besoin, définir le périmètre fonctionnel et le flux de dialogue.  
* **Développement** – implémenter le pipeline de traitement (prompt, appel LLM, action).  
* **Production** – containeriser, monitorer, assurer la haute disponibilité.  

### Acquérir des compétences pratiques avec Python 3.11  

* Utilisation des bibliothèques modernes (`openai`, `langchain`, `fastapi`).  
* Gestion d’environnements virtuels (`venv`, `poetry`).  
* Écriture de tests unitaires (`pytest`) pour garantir la robustesse du code.

### Déployer sur des environnements accessibles en Afrique  

* **Docker** sur un serveur dédié à Accra ou à Nairobi.  
* **Cloud public** (AWS Africa (Cape Town), Azure Africa (Johannesburg)) pour profiter de la proximité géographique et de la facturation à l’usage.  
* **Solution hybride** – modèle open‑source hébergé localement, appels SaaS uniquement pour les fonctions non critiques (ex. traduction).

---

## À retenir  

- Un **agent IA** combine compréhension, génération, mémoire et capacité d’action, le distinguant d’un simple chatbot.  
- Les **cas d’usage** africains (assistant multilingue, service client e‑commerce, automatisation agricole, bot santé/éducation) montrent le potentiel économique et social de ces systèmes.  
- Les **composants clés** sont le modèle de langage, le gestionnaire de dialogue, la mémoire (court et long terme) et les interfaces d’entrée/sortie.  
- Le **panorama technologique** offre un choix entre modèles open‑source (LLaMA, Mistral) pour la souveraineté des données et services SaaS (OpenAI, Anthropic) pour la rapidité d’intégration.  
- Le **positionnement** de l’agent IA s’inscrit dans une chaîne de valeur allant de la collecte de données à la mise en production, avec la possibilité d’interagir avec d’autres IA (vision, recommandation).  
- Le cours vise à **maîtriser tout le cycle** – de la conception à la production – en utilisant Python 3.11, Docker et les plateformes cloud accessibles aux développeurs francophones d’Afrique.  

Vous avez désormais les bases conceptuelles nécessaires pour envisager la création d’un agent IA adapté à votre contexte local. Le prochain chapitre vous plongera dans les notions fondamentales de l’IA et du traitement du langage naturel.