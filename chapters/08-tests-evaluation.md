## Tests unitaires : garantir le bon fonctionnement de chaque composant

### Pourquoi les tests unitaires sont indispensables
Un agent IA est composé de plusieurs briques : le pré‑traitement du texte, l’appel au LLM, la gestion du contexte, les fonctions d’action (requêtes API, accès base de données…).  
Un défaut dans l’une de ces briques peut entraîner des réponses incohérentes ou même bloquer tout le pipeline. Les tests unitaires permettent de :

* détecter rapidement les régressions après une modification,
* documenter le comportement attendu de chaque fonction,
* faciliter l’intégration continue (CI) et le déploiement automatisé.

### Installation et configuration de pytest
```bash
# dans votre environnement virtuel (voir chapitre 3)
pip install pytest pytest-mock
```

Créez un répertoire `tests/` à la racine du projet et ajoutez un fichier `conftest.py` pour partager des fixtures :

```python
# tests/conftest.py
import pytest
from my_agent.pipeline import AgentPipeline

@pytest.fixture
def pipeline():
    # Initialise l'agent avec un modèle de petite taille pour les tests
    return AgentPipeline(model_name="distilbert-base-uncased")
```

### Exemple de test unitaire sur le pré‑traitement
```python
# tests/test_preprocess.py
def test_clean_text(pipeline):
    raw = "Bonjour!!!   Comment ça va ?   😊"
    clean = pipeline.preprocess(raw)
    assert clean == "bonjour comment ça va"
```

### Mocking des appels externes
Les fonctions d’action (ex. appel à une API météo) ne doivent pas réellement toucher le réseau pendant les tests. `pytest-mock` permet de remplacer ces appels :

```python
# tests/test_actions.py
def test_get_weather(mocker, pipeline):
    # Simule la réponse de l'API météo
    mock_response = {"temp": 28, "condition": "ensoleillé"}
    mocker.patch("my_agent.actions.fetch_weather", return_value=mock_response)

    result = pipeline.run("Quel temps fait‑il à Dakar ?")
    assert "28°C" in result
    assert "ensoleillé" in result
```

### Couverture de code
Intégrez `pytest-cov` pour mesurer la proportion de code testé :

```bash
pip install pytest-cov
pytest --cov=my_agent tests/
```

Une couverture supérieure à 80 % est un bon indicateur de robustesse, mais ne remplace pas des tests fonctionnels plus poussés.

---

## Simulation de conversations : tests d’intégration

### Scénarios de conversation typiques
Pour un agent destiné à un service client d’une fintech africaine, on peut imaginer :

1. **Vérification de solde** – l’utilisateur demande son solde, l’agent interroge la base de données et répond.
2. **Signalement de fraude** – l’utilisateur signale une transaction suspecte, l’agent déclenche une alerte interne.
3. **FAQ locale** – l’utilisateur pose une question sur les frais de transfert vers le Kenya.

### Script de simulation avec `pytest`
```python
# tests/test_conversation_flow.py
SCENARIOS = [
    {
        "input": "Quel est mon solde ?",
        "expected_substr": "Votre solde est de"
    },
    {
        "input": "J’ai vu un paiement de 150 000 FCFA que je ne reconnais pas",
        "expected_substr": "Nous avons bien enregistré votre signalement"
    },
    {
        "input": "Combien coûte un transfert vers Nairobi ?",
        "expected_substr": "Le frais est de 500 FCFA"
    },
]

def test_conversation_flow(pipeline):
    for s in SCENARIOS:
        reply = pipeline.run(s["input"])
        assert s["expected_substr"] in reply
```

### Utilisation de `pytest-bdd` pour des scénarios plus complexes
```bash
pip install pytest-bdd
```

```gherkin
# tests/features/transfer_fee.feature
Feature: Calcul des frais de transfert

  Scenario: Frais pour un transfert vers le Kenya
    Given l'utilisateur veut transférer 100 000 FCFA
    When il demande le coût du transfert vers Nairobi
    Then la réponse doit contenir "500 FCFA"
```

```python
# tests/test_transfer_fee.py
from pytest_bdd import scenarios, given, when, then

scenarios("features/transfer_fee.feature")

@given("l'utilisateur veut transférer 100 000 FCFA")
def amount():
    return 100_000

@when("il demande le coût du transfert vers Nairobi")
def ask_fee(pipeline, amount):
    return pipeline.run(f"Combien coûte un transfert de {amount} FCFA vers Nairobi ?")

@then('la réponse doit contenir "500 FCFA"')
def check_fee(response):
    assert "500 FCFA" in response
```

Ces tests d’intégration valident l’interaction entre les différentes briques (pré‑traitement, appel LLM, fonctions d’action) dans des contextes réalistes.

---

## Métriques d’évaluation : quantifier la qualité des réponses

### BLEU – comparaison n‑grammes
BLEU mesure la similarité entre la réponse générée et une ou plusieurs références humaines. Il est surtout utile pour les tâches de génération de texte où il existe une « bonne » réponse attendue (ex. FAQ).

```python
# utils/metrics.py
from nltk.translate.bleu_score import sentence_bleu, SmoothingFunction

def bleu_score(reference, hypothesis):
    # reference : liste de phrases de référence
    # hypothesis : phrase générée
    smoothie = SmoothingFunction().method4
    return sentence_bleu([reference.split()], hypothesis.split(), smoothing_function=smoothie)
```

**Exemple d’utilisation** :

```python
ref = "Le frais de transfert vers Nairobi est de 500 FCFA."
hyp = "Le frais de transfert vers Nairobi est 500 FCFA."
print(f"BLEU : {bleu_score(ref, hyp):.2f}")
```

### ROUGE – rappel sur les n‑grammes
ROUGE est plus sensible au rappel que BLEU, ce qui le rend adapté aux résumés ou réponses longues.

```python
# utils/metrics.py (suite)
from rouge_score import rouge_scorer

def rouge_score(reference, hypothesis):
    scorer = rouge_scorer.RougeScorer(['rouge1', 'rougeL'], use_stemmer=True)
    scores = scorer.score(reference, hypothesis)
    return scores['rougeL'].fmeasure
```

### Précision, rappel et F1 sur les intents
Pour un agent qui classifie les intentions (solde, fraude, FAQ…), on calcule précision, rappel et F1 à partir d’une matrice de confusion.

```python
# utils/metrics.py (suite)
from sklearn.metrics import precision_recall_fscore_support

def intent_metrics(y_true, y_pred):
    precision, recall, f1, _ = precision_recall_fscore_support(
        y_true, y_pred, average='weighted')
    return {"precision": precision, "recall": recall, "f1": f1}
```

### Tableau de bord d’évaluation
Regroupez les métriques dans un fichier `evaluation_report.json` :

```json
{
  "BLEU": 0.68,
  "ROUGE-L": 0.74,
  "Intent": {
    "precision": 0.92,
    "recall": 0.89,
    "f1": 0.90
  }
}
```

Un script `scripts/evaluate.py` charge les jeux de test, exécute le pipeline et génère le rapport. Intégrez ce script dans votre pipeline CI pour obtenir un feedback à chaque commit.

---

## Collecte de retours utilisateurs : boucle d’amélioration continue

### Méthodes de collecte low‑cost
* **Formulaires Web** – après chaque échange, proposez un petit questionnaire « Cette réponse vous a‑t‑elle été utile ? » avec un score de 1 à 5.
* **Log d’interaction** – stockez le texte de la requête, la réponse générée et le score d’utilité dans une base SQLite ou PostgreSQL (voir chapitre 7).
* **Analyse des conversations abandonnées** – si l’utilisateur envoie un nouveau message sans attendre la réponse, cela peut indiquer une incompréhension.

### Traitement des retours
1. **Normalisation** – convertissez les scores en catégories (positif ≥ 4, neutre = 3, négatif ≤ 2).
2. **Échantillonnage** – sélectionnez aléatoirement 10 % des conversations négatives pour une annotation manuelle (voir chapitre 4).
3. **Enrichissement du jeu de données** – ajoutez les paires (question, réponse correcte) à votre corpus d’entraînement.

```python
# scripts/feedback_ingest.py
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://user:pwd@localhost/agent_feedback")
df = pd.read_sql("SELECT * FROM feedback WHERE rating <= 2", engine)

# Exemple d’ajout à un fichier JSON d’entraînement
with open("data/augmented.jsonl", "a") as f:
    for _, row in df.iterrows():
        f.write(json.dumps({
            "prompt": row["user_message"],
            "completion": row["expected_answer"]
        }) + "\n")
```

### Priorisation des améliorations
* **Bugs critiques** – réponses erronées qui peuvent entraîner une perte financière (ex. mauvais montant de transfert). Corrigez immédiatement.
* **Faibles scores récurrents** – si un intent particulier obtient régulièrement < 3, envisagez d’ajouter plus d’exemples ou de ré‑entraîner le classifieur.
* **Nouvelles fonctionnalités** – les retours peuvent révéler des besoins non couverts (ex. support du swahili). Planifiez un itération dédiée.

---

## Intégration continue et automatisation des tests

### Pipeline CI avec GitHub Actions
```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
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
          pip install pytest pytest-cov
      - name: Run unit tests
        run: |
          source venv/bin/activate
          pytest --cov=my_agent tests/
      - name: Run evaluation script
        run: |
          source venv/bin/activate
          python scripts/evaluate.py > evaluation_report.json
      - name: Upload report as artifact
        uses: actions/upload-artifact@v3
        with:
          name: evaluation-report
          path: evaluation_report.json
```

Ce workflow :

1. Installe les dépendances,
2. Exécute les tests unitaires et d’intégration,
3. Lance l’évaluation des métriques,
4. Archive le rapport pour consultation.

### Gestion des secrets
Les clés d’API (OpenAI, services météo, etc.) sont stockées dans les **secrets** du dépôt GitHub et injectées via `${{ secrets.OPENAI_API_KEY }}`. Ainsi, les tests qui utilisent des appels réels restent sécurisés.

### Tests de performance (optionnel)
Pour s’assurer que le temps de réponse reste acceptable (< 2 s) même sous charge, ajoutez un test de performance avec `locust` ou `pytest-benchmark`.

```python
# tests/test_performance.py
def test_latency(pipeline, benchmark):
    result = benchmark(pipeline.run, "Quel est le taux de change du dollar aujourd'hui ?")
    assert result < 2.0  # secondes
```

---

## Bonnes pratiques d’écriture de tests pour les agents IA

| Pratique | Description |
|----------|-------------|
| **Isoler les dépendances externes** | Utilisez le mocking pour les API, bases de données et appels LLM afin que les tests restent rapides et déterministes. |
| **Utiliser des données de test réalistes** | Créez des jeux de test qui reproduisent les variations linguistiques locales (français, langues locales comme le wolof ou le swahili). |
| **Vérifier le contenu et le format** | Au-delà du texte brut, testez la présence de balises JSON, de liens ou de formats numériques attendus. |
| **Automatiser la génération de rapports** | Intégrez `pytest-cov` et votre script d’évaluation dans le CI pour disposer d’un tableau de bord à jour. |
| **Faire évoluer les scénarios** | Ajoutez régulièrement de nouveaux scénarios issus des retours utilisateurs pour couvrir les cas d’usage émergents. |
| **Documenter les fixtures** | Chaque fixture doit clairement indiquer le contexte (modèle léger, jeu de données factice) pour faciliter la maintenance. |

---

## Points clés

* **Tests unitaires** : chaque fonction (pré‑traitement, appel LLM, action) possède son propre test, avec mock des dépendances externes.  
* **Scénarios de conversation** : les tests d’intégration valident le flux complet en reproduisant des dialogues typiques du contexte africain (fintech, service client, FAQ locales).  
* **Métriques** : combinez BLEU/ROUGE pour la qualité du texte, et précision/recall/F1 pour la classification d’intents. Un tableau de bord automatisé fournit une visibilité continue.  
* **Feedback utilisateur** : récoltez les scores d’utilité, enrichissez le jeu de données et priorisez les corrections en fonction de l’impact métier.  
* **CI/CD** : intégrez les suites de tests et le script d’évaluation dans un pipeline GitHub Actions (ou GitLab CI) pour garantir que chaque modification reste fiable et performante.  
* **Itération** : la boucle « test → évaluation → feedback → re‑entraînement » est le cœur de l’amélioration continue d’un agent IA, surtout dans des environnements où les langues et les besoins évoluent rapidement.