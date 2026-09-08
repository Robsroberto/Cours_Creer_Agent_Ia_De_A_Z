## Identifier les sources de données pertinentes

L’étape cruciale avant tout entraînement consiste à savoir **où puiser les informations** qui permettront à votre agent IA de répondre de façon pertinente.  
En contexte africain francophone, trois catégories d’accès sont généralement les plus utiles :

| Source | Exemple concret | Pourquoi c’est intéressant |
|--------|-----------------|-----------------------------|
| **API publiques** | API de la Banque Mondiale (`https://api.worldbank.org/v2/country/CM`) | Données fiables, structurées, régulièrement mises à jour ; idéales pour des réponses chiffrées (PIB, indicateurs sociaux). |
| **Bases de données locales** | Jeu de données « FAQ » d’un service client d’une PME de Douala stocké dans un fichier SQLite | Permet de capitaliser sur le savoir interne, d’assurer la pertinence sectorielle. |
| **Web scraping** | Extraction des articles du site « Le Monde Afrique » ou des forums de développeurs comme **Developpez.com/africa** | Accès à du texte brut riche, couvrant les expressions locales, les néologismes et les questions fréquentes. |

> **voir chapitre 3** pour la configuration de l’environnement Python et l’installation des bibliothèques nécessaires (requests, beautifulsoup4, pandas, etc.).

### Sélectionner les sources selon le cas d’usage

- **Assistant administratif** : privilégiez les API gouvernementales (open data) et les bases internes (agenda, contacts).  
- **Chatbot support technique** : combinez les tickets résolus (base locale) avec les articles de documentation (scraping).  
- **Agent conversationnel culturel** : ciblez les sites de presse locale, les blogs linguistiques et les réseaux sociaux (Twitter, Facebook) pour capter les tournures idiomatiques.

---

## Accéder aux API publiques

Les API offrent généralement un format JSON ou CSV, facile à ingérer. Voici un petit pipeline Python qui récupère les indicateurs de santé d’un pays africain et les transforme en DataFrame :

```python
import requests
import pandas as pd

def fetch_worldbank_indicator(country_code: str, indicator: str) -> pd.DataFrame:
    url = f"https://api.worldbank.org/v2/country/{country_code}/indicator/{indicator}"
    params = {"format": "json", "date": "2015:2023", "per_page": "1000"}
    response = requests.get(url, params=params)
    response.raise_for_status()
    data = response.json()[1]                     # la première entrée est le méta‑données
    df = pd.json_normalize(data)
    df = df[["date", "value"]].rename(columns={"date": "Année", "value": indicator})
    df["Année"] = pd.to_numeric(df["Année"])
    return df

# Exemple : taux de mortalité infantile au Niger (NER)
df_mortality = fetch_worldbank_indicator("NER", "SH.DTH.INF.ZS")
print(df_mortality.head())
```

**Bonnes pratiques** :

1. **Gestion du throttling** – ajoutez un `time.sleep(0.5)` entre les appels pour éviter le blocage.  
2. **Cache local** – stockez les réponses JSON dans un dossier `cache/` afin de ne pas re‑interroger l’API pendant le développement.  
3. **Gestion des erreurs** – utilisez `try/except` pour capturer les codes HTTP 429 (trop de requêtes) ou 500 (serveur indisponible).

---

## Scraper le Web de façon responsable

Le scraping doit respecter les **robots.txt** et les conditions d’utilisation du site. En Afrique, de nombreux portails d’information ne proposent pas d’API, ce qui rend le scraping indispensable.

```python
import requests
from bs4 import BeautifulSoup
import re

def scrape_african_news(url: str, max_pages: int = 3) -> list[str]:
    articles = []
    for page in range(1, max_pages + 1):
        page_url = f"{url}?page={page}"
        resp = requests.get(page_url, headers={"User-Agent": "EmpireDuWebBot/1.0"})
        soup = BeautifulSoup(resp.text, "html.parser")
        for a in soup.select("h2.title a"):
            article_url = a["href"]
            article_resp = requests.get(article_url, headers={"User-Agent": "EmpireDuWebBot/1.0"})
            article_soup = BeautifulSoup(article_resp.text, "html.parser")
            paragraphs = article_soup.select("div.article-body p")
            text = " ".join(p.get_text(strip=True) for p in paragraphs)
            # Nettoyage basique des espaces et des caractères non‑ASCII
            text = re.sub(r"\s+", " ", text)
            articles.append(text)
    return articles

news = scrape_african_news("https://www.lemonde.fr/afrique")
print(f"Nombre d'articles récupérés : {len(news)}")
```

**Points d’attention** :

- **Respect du taux de requêtes** : limitez à 1 requête / seconde pour les sites à faible capacité serveur.  
- **Gestion de la pagination** : adaptez le sélecteur CSS (`h2.title a`) à la structure du site ciblé.  
- **Encodage** : forcez `response.encoding = "utf-8"` si le site ne le spécifie pas, afin de conserver les caractères accentués et les caractères spéciaux des langues locales.

---

## Exploiter les bases de données locales

Souvent, votre client possède déjà des bases de connaissances (FAQ, tickets, manuels). Voici comment charger rapidement un fichier SQLite contenant des dialogues précédemment résolus :

```python
import sqlite3
import pandas as pd

def load_faq_from_sqlite(db_path: str) -> pd.DataFrame:
    conn = sqlite3.connect(db_path)
    query = """
        SELECT question, answer, langue
        FROM faq
        WHERE langue IN ('fr', 'wolof', 'swahili')
    """
    df = pd.read_sql_query(query, conn)
    conn.close()
    return df

faq_df = load_faq_from_sqlite("data/client_faq.db")
print(faq_df.head())
```

**Astuce** : ajoutez un index sur la colonne `langue` pour accélérer les requêtes multilingues.

---

## Nettoyer les textes bruts

Les données collectées sont rarement prêtes à être utilisées « tel quel ». Le nettoyage doit :

1. **Normaliser les caractères** (Unicode NFC) pour éviter les variantes de même lettre.  
2. **Supprimer les balises HTML** résiduelles, les URLs et les mentions inutiles (`@username`).  
3. **Gérer les espaces** (double espaces, tabulations).  
4. **Filtrer les lignes vides** et les réponses trop courtes (< 5 mots).

```python
import unicodedata
import re

def clean_text(text: str) -> str:
    # 1. Normalisation Unicode
    text = unicodedata.normalize("NFC", text)

    # 2. Suppression des balises HTML
    text = re.sub(r"<[^>]+>", " ", text)

    # 3. Suppression des URLs
    text = re.sub(r"https?://\S+", " ", text)

    # 4. Suppression des mentions et hashtags
    text = re.sub(r"[@#]\w+", " ", text)

    # 5. Nettoyage des espaces
    text = re.sub(r"\s+", " ", text).strip()
    return text

faq_df["question_clean"] = faq_df["question"].apply(clean_text)
faq_df["answer_clean"]   = faq_df["answer"].apply(clean_text)
```

### Gestion des caractères spéciaux africains

Certaines langues (p.ex. le **fon**, le **wolof**) utilisent des caractères diacritiques rares. Il faut :

- **Conserver les diacritiques** : ne pas les supprimer dans le nettoyage, sinon le sens change.  
- **Vérifier la présence dans le vocabulaire du modèle** : si le LLM choisi ne les connaît pas, envisagez un **tokenizer custom** (voir chapitre 5).  

---

## Normalisation et tokenisation multilingue

Après le nettoyage, le texte doit être découpé en unités compréhensibles par le modèle. Deux approches courantes :

| Méthode | Quand l’utiliser | Exemple d’outil |
|---------|------------------|-----------------|
| **Byte‑Pair Encoding (BPE)** | Langues à alphabet latin avec peu de variantes | `tokenizers` de Hugging Face (`ByteLevelBPETokenizer`). |
| **SentencePiece (unigram)** | Langues à faible ressource, incluant des caractères non‑latins | `sentencepiece` (modèle `spm_train`). |

```python
from tokenizers import ByteLevelBPETokenizer

def train_bpe_tokenizer(texts: list[str], vocab_size: int = 30_000):
    tokenizer = ByteLevelBPETokenizer()
    tokenizer.train_from_iterator(texts, vocab_size=vocab_size, min_frequency=2)
    tokenizer.save_model("tokenizer", "bpe")
    return tokenizer

# Entraînement sur les questions et réponses nettoyées
tokenizer = train_bpe_tokenizer(
    list(faq_df["question_clean"]) + list(faq_df["answer_clean"])
)
```

**Note** : gardez une copie du tokenizer dans le répertoire du projet (`tokenizer/`) et versionnez‑le avec Git. Cela garantit que le même vocabulaire sera utilisé lors du fine‑tuning (voir chapitre 5).

---

## Gestion des langues locales

### Français

- **Accentuation** : le français utilise des accents (é, è, à) qui doivent être conservés.  
- **Contractions** : « c’est », « j’ai » – le tokenizer BPE les gère généralement bien.

### Langues africaines (ex. Wolof, Swahili, Bambara)

- **Variantes orthographiques** : le wolof peut s’écrire avec ou sans apostrophe (`ndey` vs `ndéy`). Normalisez via une table de correspondance.  
- **Mots composés** : certains termes sont formés par concaténation (`sunu`+`bokk` = `sunubokk`). Découpez‑les avec un dictionnaire de racines si le modèle montre des difficultés.  

```python
# Exemple de normalisation spécifique au wolof
wolof_map = {"ndéy": "ndey", "jàmm": "jamm", "ñuul": "nuul"}

def normalize_wolof(text: str) -> str:
    for k, v in wolof_map.items():
        text = text.replace(k, v)
    return text

faq_df["question_clean"] = faq_df["question_clean"].apply(normalize_wolof)
```

---

## Annotation pour le fine‑tuning

Le **fine‑tuning** d’un LLM nécessite des paires `prompt → réponse` clairement délimitées. Deux formats populaires :

| Format | Exemple de ligne |
|--------|------------------|
| **JSONL** (un objet par ligne) | `{"prompt":"Comment obtenir un prêt agricole au Mali ?","completion":"Vous devez vous rendre à la DGI…","metadata":{"lang":"fr"}}` |
| **CSV** (colonnes séparées) | `prompt,completion,lang`<br>`"Quel est le taux de change du franc CFA?","1 FCFA = 0,0015 USD","fr"` |

### Outils d’annotation collaboratifs

- **Label Studio** (open‑source) : interface web où chaque contributeur peut saisir la réponse attendue.  
- **doccano** : idéal pour les projets multilingues grâce à la prise en charge de plusieurs jeux de données simultanément.  

**Flux d’annotation recommandé** :

1. **Pré‑sélection** : le script ci‑dessus extrait les Q/R brutes.  
2. **Filtrage automatique** : éliminez les paires où la réponse fait moins de 10 mots ou contient plus de 50 % de chiffres (souvent des tables).  
3. **Annotation manuelle** : les annotateurs vérifient la pertinence, corrigent les fautes et ajoutent le champ `lang`.  
4. **Export** en JSONL, puis **validation** avec un petit script Python qui compte les lignes, les longueurs moyennes et les langues présentes.

```python
import json

def validate_dataset(path: str):
    langs = set()
    lengths = []
    with open(path, "r", encoding="utf-8") as f:
        for i, line in enumerate(f, 1):
            obj = json.loads(line)
            langs.add(obj.get("metadata", {}).get("lang", "unknown"))
            lengths.append(len(obj["completion"].split()))
    print(f"Lignes : {i}")
    print(f"Langues détectées : {langs}")
    print(f"Longueur moyenne de la réponse : {sum(lengths)/len(lengths):.1f} mots")

validate_dataset("data/faq_finetune.jsonl")
```

---

## Automatiser le pipeline de préparation

Pour éviter de refaire les mêmes étapes à chaque itération, encapsulez le flux dans une **pipeline** (ex. avec `prefect` ou `luigi`). Exemple minimal avec **prefect** :

```python
from prefect import flow, task
import pandas as pd

@task
def load_data():
    return pd.read_csv("raw/faq_raw.csv")

@task
def clean(df):
    df["question"] = df["question"].apply(clean_text)
    df["answer"]   = df["answer"].apply(clean_text)
    return df

@task
def split_and_save(df):
    df.to_json("processed/faq_finetune.jsonl", orient="records", lines=True)

@flow
def prepare_dataset():
    df = load_data()
    df_clean = clean(df)
    split_and_save(df_clean)

if __name__ == "__main__":
    prepare_dataset()
```

Cette approche vous donne :

- **Reproductibilité** : chaque exécution part du même point d’entrée.  
- **Traçabilité** : les logs de `prefect` indiquent quelles tâches ont échoué.  
- **Scalabilité** : vous pouvez paralléliser le scraping ou le téléchargement d’API en augmentant le nombre de workers.

---

## Sécurité, confidentialité et conformité

- **Masquage des données sensibles** : avant le nettoyage, identifiez les champs contenant des numéros de téléphone, des identifiants ou des adresses e‑mail. Utilisez `re.sub(r"\b\d{2,3}[-.\s]?\d{2,3}[-.\s]?\d{2,3}\b", "[NUMERO]", text)`.  
- **Consentement** : assurez‑vous que les