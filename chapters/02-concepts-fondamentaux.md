## Intelligence artificielle : panorama et vocabulaire

L’intelligence artificielle (IA) désigne l’ensemble des techniques qui permettent à une machine d’accomplir des tâches qui, traditionnellement, requièrent l’intelligence humaine : raisonner, apprendre, percevoir, communiquer.  
Dans le domaine du développement logiciel, l’IA se décline en trois niveaux d’abstraction :

| Niveau | Description | Exemple concret |
|--------|-------------|-----------------|
| **Intelligence artificielle symbolique** | Règles logiques écrites à la main (ex. systèmes experts). | Un chatbot qui répond « Bonjour » uniquement si le texte exact « bonjour » est détecté. |
| **Machine learning (apprentissage automatique)** | Algorithmes qui apprennent des données sans être explicitement programmés. | Un classifieur qui prédit si un message texte provient d’un client satisfait ou mécontent. |
| **Deep learning (apprentissage profond)** | Sous‑ensemble du ML utilisant des réseaux de neurones multicouches, capables d’extraire automatiquement des représentations complexes. | Un modèle de traduction qui passe du français au swahili sans règles linguistiques pré‑codées. |

Ces trois niveaux s’enchaînent : le deep learning est une technique de machine learning, qui lui‑même fait partie du champ plus large de l’IA.

---

## Apprentissage automatique : du jeu de données au modèle

### 1. Le pipeline de base

1. **Collecte des données** – Textes, images, sons.  
2. **Pré‑traitement** – Nettoyage, normalisation, tokenisation (pour le texte).  
3. **Séparation** – Jeux d’entraînement (≈ 80 %), de validation (≈ 10 %) et de test (≈ 10 %).  
4. **Choix de l’algorithme** – Régression, arbres de décision, SVM, réseaux de neurones.  
5. **Entraînement** – Optimisation d’une fonction de perte sur le jeu d’entraînement.  
6. **Évaluation** – Mesure de la performance sur le jeu de validation / test.  

```python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score

# 1. Chargement d’un jeu de données de tickets support (français)
df = pd.read_csv('tickets.csv')
X = df['texte']                # texte du ticket
y = df['catégorie']            # catégorie à prédire

# 2. Tokenisation très simple (pour l’exemple)
X_tokenized = X.str.lower().str.replace(r'[^a-zàâçéèêëïîôûùüÿñæœ\s]', '', regex=True)

# 3. Séparation
X_train, X_test, y_train, y_test = train_test_split(
    X_tokenized, y, test_size=0.2, random_state=42)

# 4. Modèle (logistique) – on utilise un vecteur TF‑IDF simplifié
from sklearn.feature_extraction.text import TfidfVectorizer
vectorizer = TfidfVectorizer()
X_train_vec = vectorizer.fit_transform(X_train)
X_test_vec  = vectorizer.transform(X_test)

clf = LogisticRegression(max_iter=1000)
clf.fit(X_train_vec, y_train)

# 5. Évaluation
y_pred = clf.predict(X_test_vec)
print('Accuracy :', accuracy_score(y_test, y_pred))
```

Ce script illustre le flux complet d’un modèle de classification de texte simple. Dans les chapitres suivants, nous remplacerons la régression logistique par des **grands modèles de langage (LLM)** qui offrent des performances nettement supérieures.

### 2. Supervision vs non‑supervision

- **Supervisé** : chaque exemple d’entraînement possède une étiquette (ex. “spam / non‑spam”).  
- **Non‑supervisé** : le modèle découvre des structures sans étiquette (ex. clustering de documents).  

Dans le NLP, les modèles de langage pré‑entraînés sont généralement **non‑supervisés** : ils apprennent à prédire le mot suivant à partir de milliards de phrases, sans aucune annotation humaine.

---

## Deep learning : réseaux de neurones et représentations

### 1. Architecture de base d’un réseau de neurones

Un réseau est constitué de **couches** (layers). Chaque couche applique une transformation linéaire suivie d’une fonction d’activation non linéaire (ReLU, tanh, etc.).  

```
input → Dense → ReLU → Dense → Softmax → output
```

Dans le traitement du texte, les couches les plus courantes sont :

- **Embedding** : transforme chaque token (mot ou sous‑mot) en un vecteur dense de dimension fixe (ex. 768).  
- **Transformer** : architecture à base d’attention qui capture les dépendances à longue distance.  

### 2. Le transformer, moteur des LLM

Le transformer, introduit par Vaswani et al. (2017), repose sur le mécanisme **self‑attention** :

```text
Attention(Q, K, V) = softmax(Q·Kᵀ / √d_k) · V
```

- **Q** (queries), **K** (keys) et **V** (values) sont des projections linéaires des mêmes entrées.  
- Le facteur √d_k assure une stabilité numérique.  

Cette architecture permet de traiter simultanément tous les tokens d’une phrase, contrairement aux réseaux récurrents (RNN) qui les parcourent séquentiellement. Le résultat : des modèles capables de comprendre le contexte global d’un texte en quelques millisecondes.

---

## Traitement du langage naturel (NLP) : définitions clés

### 1. Tokenisation et sous‑mots

La tokenisation consiste à découper le texte en unités manipulables par le modèle. Pour les langues africaines (ex. le wolof, le lingala) où les frontières de mots peuvent être floues, les **tokenizers à sous‑mots** (Byte‑Pair Encoding – BPE, WordPiece) sont très utiles. Ils créent un vocabulaire partagé entre les langues et réduisent le nombre de tokens inconnus (`[UNK]`).

```python
from transformers import AutoTokenizer

tokenizer = AutoTokenizer.from_pretrained("facebook/mbart-large-50")
sentence = "Bonjour, comment ça va ?"
tokens = tokenizer(sentence, return_tensors="pt")
print(tokens)
```

### 2. Embeddings : vecteurs qui portent le sens

Un **embedding** est un vecteur dense qui représente un token dans un espace sémantique. Deux mots proches sémantiquement (ex. « marché » et « bazar ») auront des vecteurs proches (distance euclidienne faible). Les embeddings sont la base des modèles de langage modernes.

### 3. Fine‑tuning vs prompts

| Méthode | Principe | Quand l’utiliser |
|--------|----------|-------------------|
| **Fine‑tuning** | On ré‑entraîne (ou on continue l’entraînement) le modèle sur une tâche spécifique avec un jeu de données annoté. | Besoin de performances élevées sur un domaine très ciblé (ex. assistance juridique en français). |
| **Prompting** | On fournit une instruction texte (prompt) au modèle pré‑entraîné pour qu’il génère la réponse souhaitée, sans modifier ses poids. | Cas où les ressources de calcul sont limitées ou où le jeu de données d’entraînement est inexistant (ex. génération de réponses rapides pour un chatbot de FAQ). |

#### Exemple de prompting simple

```python
from transformers import pipeline

generator = pipeline("text-generation", model="bigscience/bloom-560m")
prompt = "En Afrique de l'Ouest, quel est le meilleur moyen de réduire le gaspillage alimentaire ?"
result = generator(prompt, max_new_tokens=80, do_sample=True, temperature=0.7)
print(result[0]["generated_text"])
```

#### Exemple de fine‑tuning minimal (avec 🤗 Trainer)

```python
from datasets import load_dataset
from transformers import AutoModelForCausalLM, Trainer, TrainingArguments

model = AutoModelForCausalLM.from_pretrained("bigscience/bloom-560m")
dataset = load_dataset("csv", data_files={"train": "faq_fr.csv"})["train"]

def tokenize(batch):
    return tokenizer(batch["question"], truncation=True, padding="max_length", max_length=128)

tokenized = dataset.map(tokenize, batched=True)

training_args = TrainingArguments(
    output_dir="./bloom-faq-fr",
    per_device_train_batch_size=4,
    num_train_epochs=2,
    learning_rate=5e-5,
    weight_decay=0.01,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized,
)

trainer.train()
```

Le fine‑tuning ajuste les poids du modèle pour qu’il réponde spécifiquement aux questions de la FAQ, alors que le prompting aurait simplement utilisé le modèle générique.

---

## Modèles pré‑entraînés : panorama et accès

### 1. Modèles open‑source

| Modèle | Taille (paramètres) | Langues supportées | Licence |
|--------|--------------------|--------------------|---------|
| **LLaMA 2** (Meta) | 7 B – 70 B | Multilingue (incl. français) | Meta Research License |
| **Mistral 7B** | 7 B | Multilingue | Apache 2.0 |
| **BLOOM** (BigScience) | 560 M – 176 B | 46 langues, dont le français et plusieurs langues africaines | RAIL‑License |

Ces modèles peuvent être téléchargés via le hub 🤗 Hub et exécutés localement, ce qui est idéal pour les développeurs africains qui souhaitent éviter les coûts de l’API cloud.

```bash
pip install transformers huggingface_hub
```

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_name = "mistralai/Mistral-7B-Instruct-v0.1"
tokenizer = AutoTokenizer.from_pretrained(model_name)
model = AutoModelForCausalLM.from_pretrained(model_name, device_map="auto")
```

### 2. Services SaaS (API)

- **OpenAI (GPT‑4, GPT‑3.5)** – accès via clé API, facturation à la requête.  
- **Anthropic (Claude)** – API sécurisée, orientation « safe‑first ».  

Ces services offrent une **latence** très faible et une **mise à jour continue** du modèle, mais le coût peut rapidement devenir un frein pour des projets à petit budget. Le choix entre open‑source et SaaS dépend donc du **budget**, de la **souveraineté des données** (ex. données sensibles d’une ONG locale) et du **niveau de performance** requis.

---

## Limites et bonnes pratiques à garder en tête

### 1. Biais et représentativité des données

Les modèles apprennent les biais présents dans leurs corpus d’entraînement. Un LLM entraîné majoritairement sur des textes anglophones peut produire des réponses moins précises en français ou dans les langues locales (ex. bambara). Il est donc crucial de :

- **Évaluer** le modèle sur des jeux de données représentatifs de votre audience.  
- **Compléter** le modèle avec du fine‑tuning sur des textes locaux (ex. articles de presse sénégalais).  

### 2. Coût computationnel

Les modèles de plusieurs dizaines de milliards de paramètres nécessitent des GPU de classe A100 ou équivalents. Pour un développeur en Afrique, les options réalistes sont :

- **Utiliser des modèles ≤ 7 B** (Mistral, LLaMA‑2‑7B) qui tournent sur un GPU RTX 3060 ou même sur un CPU avec quantisation.  
- **Appliquer la quantisation 8‑bits** ou la **pruning** pour réduire la taille du modèle sans perdre trop de précision.

```python
from transformers import BitsAndBytesConfig

quant_config = BitsAndBytesConfig(load_in_8bit=True)
model = AutoModelForCausalLM.from_pretrained(
    "mistralai/Mistral-7B-Instruct-v0.1",
    quantization_config=quant_config,
    device_map="auto"
)
```

### 3. Confidentialité et conformité

Lorsque vous manipulez des données personnelles (ex. dossiers médicaux ou informations financières), assurez‑vous de :

- **Anonymiser** les textes avant de les envoyer à une API tierce.  
- **Respecter** les réglementations locales (ex. la loi nigérienne sur la protection des données).  

### 4. Hallucinations et fiabilité

Les LLM peuvent « halluciner », c’est‑à‑dire générer des informations factuellement incorrectes. Les stratégies pour limiter ce phénomène comprennent :

- **Récupération augmentée** (RAG) : interroger une base de connaissances avant de générer la réponse.  
- **Post‑validation** : vérifier la sortie avec des règles simples (ex. format de date, présence d’un numéro de téléphone).  

---

## Synthèse des concepts clés

### 1. IA, ML, DL – une hiérarchie
- IA = champ global.  
- ML = méthodes d’apprentissage à partir de données.  
- DL = réseaux de neurones profonds, base des LLM modernes.

### 2. NLP repose sur la tokenisation, les embeddings et les transformeurs.
- Les sous‑mots permettent de couvrir les langues africaines à faible ressource.  
- Les embeddings capturent le sens et sont réutilisables dans de multiples tâches.

### 3. Modèles pré‑entraînés vs fine‑tuning vs prompting
- **Prompting** : rapide, aucune modification du modèle.  
- **Fine‑tuning** : meilleur résultat sur un domaine ciblé, nécessite des données annotées et du calcul.  
- **Modèles pré‑entraînés** : point de départ commun, disponibles en open‑source (Mistral, LLaMA) ou via API (OpenAI).

### 4. Limites à anticiper
- Biais linguistiques et culturels.  
- Coût matériel et énergétique.  
- Risque d’hallucinations, besoin de validation.

---

## À retenir

- **Comprendre la chaîne** : données → tokenisation → embeddings → transformer → sortie.  
- **Choisir le bon compromis** entre open‑source (contrôle, coût) et SaaS (simplicité, performance).  
- **Adapter les modèles** aux réalités locales : fine‑tune avec des textes africains, quantise pour réduire les exigences matérielles.  
- **Intégrer des garde‑fous** (validation, RAG) pour éviter les hallucinations et garantir la conformité aux normes de confidentialité.  

Ces bases vous permettront d’aborder les chapitres suivants avec une vision claire des leviers techniques et des contraintes à maîtriser lors de la création d’un agent IA complet.