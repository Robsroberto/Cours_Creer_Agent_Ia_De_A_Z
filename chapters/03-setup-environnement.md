## Installation de Python 3.11

Python 3.11 est la version recommandée pour les projets d’IA modernes : elle offre un meilleur rendement (≈ 10 % plus rapide que 3.10) et un support à long terme.  

### 1. Choisir la bonne distribution

| Distribution | Avantages pour l’Afrique | Comment l’obtenir |
|--------------|--------------------------|-------------------|
| **Python officiel (python.org)** | Binaires légers, aucune dépendance supplémentaire | Télécharger le fichier **Windows‑x86‑64‑installer.exe** ou le **tar.gz** Linux depuis <https://www.python.org/downloads/release/python-3110/> |
| **Miniconda** | Gestion simplifiée des environnements, accès à des miroirs conda‑forge plus proches (ex. `https://mirrors.tuna.tsinghua.edu.cn/anaconda/`) | `wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh` puis `bash Miniconda3-latest-Linux-x86_64.sh` |
| **Pyenv** (Linux/macOS) | Permet d’installer plusieurs versions côte à côte | `curl https://pyenv.run | bash` puis `pyenv install 3.11.9` |

> **Astuce** : sur les réseaux mobiles très limités, privilégiez le **Mini‑installer** Windows qui ne nécessite qu’un seul fichier d’environ 30 Mo.

### 2. Vérifier l’installation

```bash
python3 --version   # doit renvoyer Python 3.11.x
pip --version       # pip doit pointer vers la même version
```

Si la commande `python3` n’est pas reconnue sous Windows, utilisez simplement `python`.

---

## Création d’un environnement virtuel

Un *environnement virtuel* (venv) garantit que chaque projet possède ses propres dépendances, évitant les conflits entre versions de bibliothèques.

### 1. Initialiser le projet

```bash
mkdir mon_agent_ia
cd mon_agent_ia
```

### 2. Créer le venv

```bash
python3 -m venv .venv          # crée le répertoire .venv
```

### 3. L’activer

| OS | Commande |
|----|----------|
| **Windows** | `.\.venv\Scripts\activate` |
| **Linux / macOS** | `source .venv/bin/activate` |

Une fois activé, le prompt affichera `(.venv)` : toutes les installations suivantes seront locales à ce projet.

### 4. Mettre à jour pip et installer les outils de base

```bash
pip install --upgrade pip setuptools wheel
```

---

## Configuration de VS Code pour un workflow fluide

Visual Studio Code (VS Code) est gratuit, léger et fonctionne même sur des machines modestes (ex. : un portable avec 4 Go de RAM).  

### 1. Installation

- **Windows** : télécharger le fichier `UserSetup-x64.exe` depuis <https://code.visualstudio.com/> et lancer l’assistant.  
- **Linux** : `sudo snap install --classic code` ou `sudo apt install code` (si le dépôt Microsoft a été ajouté).  
- **macOS** : `brew install --cask visual-studio-code`.

### 2. Extensions indispensables

| Extension | Pourquoi |
|-----------|----------|
| **Python** (Microsoft) | Linting, IntelliSense, gestion du venv. |
| **Pylance** | Analyse statique ultra‑rapide, suggestions de type. |
| **GitLens** | Historique détaillé des commits, très utile avec une connexion intermittente. |
| **Docker** (si vous prévoyez le déploiement) | Visualisation des conteneurs, génération de Dockerfile. |
| **Remote‑SSH** (facultatif) | Travailler directement sur un serveur distant (ex. : un VPS africain). |

### 3. Lier le venv à VS Code

Ouvrez le dossier du projet (`File > Open Folder…`). VS Code détecte automatiquement le venv ; sinon :

1. Ouvrez la palette (`Ctrl+Shift+P`).  
2. Tapez **Python: Select Interpreter**.  
3. Choisissez l’interpréteur situé dans `.venv/bin/python` (Linux/macOS) ou `.venv\Scripts\python.exe` (Windows).

### 4. Paramètres de sauvegarde adaptés à une connexion lente

Ajoutez le bloc suivant dans le fichier `.vscode/settings.json` :

```json
{
    "files.autoSave": "afterDelay",
    "files.autoSaveDelay": 2000,
    "git.autofetch": false,
    "editor.formatOnSave": true,
    "python.linting.enabled": true,
    "python.linting.pylintEnabled": true,
    "python.analysis.typeCheckingMode": "basic"
}
```

- `git.autofetch` désactivé évite des requêtes réseau inutiles.  
- `autoSave` limite les pertes de travail en cas de coupure d’alimentation.

---

## Git : versionner votre code même avec un accès internet limité

Git est la pierre angulaire du travail collaboratif. Même si votre connexion est intermittente, vous pouvez travailler hors‑ligne et pousser les changements dès que le réseau redevient disponible.

### 1. Installation

- **Windows** : téléchargez l’installateur depuis <https://git-scm.com/download/win>.  
- **Linux** : `sudo apt install git` ou `sudo dnf install git`.  
- **macOS** : `brew install git`.

### 2. Configuration initiale

```bash
git config --global user.name "Votre Nom"
git config --global user.email "vous@example.com"
git config --global core.autocrlf input   # évite les problèmes de fin de ligne
```

### 3. Créer un dépôt local

```bash
git init
git add .
git commit -m "Initialisation du projet d'agent IA"
```

### 4. Utiliser un miroir Git local

Dans plusieurs pays d’Afrique (ex. : Kenya, Nigeria), des organisations communautaires hébergent des miroirs de GitHub (`https://github.com.cnpmjs.org/`). Vous pouvez les déclarer comme *remote* :

```bash
git remote add origin https://github.com.cnpmjs.org/username/mon_agent_ia.git
```

Pousser les changements :

```bash
git push -u origin master
```

> **Conseil** : si le réseau est très instable, créez des *commits* fréquents (toutes les 10‑15 minutes). Ainsi, même en cas de perte de connexion, votre travail reste sauvegardé localement.

---

## Installation des bibliothèques essentielles

Les trois bibliothèques suivantes constituent le socle de notre agent : `transformers` (modèles de langage), `langchain` (chaînes de traitement) et `fastapi` (API web).  

### 1. Utiliser un miroir PyPI

Le serveur officiel `https://pypi.org/simple` peut être lent depuis l’Afrique. Configurez pip pour passer par un miroir plus proche :

```bash
mkdir -p ~/.pip
cat > ~/.pip/pip.conf <<EOF
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
trusted-host = pypi.tuna.tsinghua.edu.cn
EOF
```

> Vous pouvez remplacer l’URL par le miroir de votre région (ex. : `https://mirrors.aliyun.com/pypi/simple/`).

### 2. Installer les paquets

```bash
pip install transformers==4.41.0
pip install langchain==0.2.5
pip install "fastapi[all]"   # installe fastapi + uvicorn
```

#### Vérification rapide

```python
import transformers, langchain, fastapi

print("transformers version :", transformers.__version__)
print("langchain version :", langchain.__version__)
print("fastapi version :", fastapi.__version__)
```

Si le script s’exécute sans erreur, votre stack est prête.

### 3. Gestion de la taille des modèles

Les modèles LLM sont volumineux (plusieurs gigaoctets). Sur des connexions limitées, utilisez :

- **Modèles quantifiés** (`bitsandbytes`) : réduction de la taille de 4× à 8×.  
- **Miroirs de modèles** : par exemple, le dépôt `huggingface.co` possède un miroir africain (`https://huggingface.co/` via CDN local).  

Installation de `bitsandbytes` (compatible avec CUDA 11.8 ou 12) :

```bash
pip install bitsandbytes
```

---

## Exemple complet : créer un mini‑projet d’agent IA

### 1. Structure du répertoire

```
mon_agent_ia/
│
├─ .venv/               # environnement virtuel
├─ app/
│   ├─ main.py          # point d’entrée FastAPI
│   └─ agent.py         # logique LangChain + Transformers
├─ tests/
│   └─ test_agent.py
├─ .gitignore
└─ requirements.txt
```

### 2. `requirements.txt`

```text
transformers==4.41.0
langchain==0.2.5
fastapi[all]==0.115.0
bitsandbytes==0.43.1
```

### 3. `app/agent.py` – un agent très simple

```python
from transformers import AutoModelForCausalLM, AutoTokenizer
from langchain.schema import LLMResult
from typing import List

# Chargement d’un petit modèle quantifié (ex. : "mistralai/Mistral-7B-Instruct-v0.1")
MODEL_NAME = "mistralai/Mistral-7B-Instruct-v0.1"

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME, trust_remote_code=True)
model = AutoModelForCausalLM.from_pretrained(
    MODEL_NAME,
    device_map="auto",
    load_in_8bit=True,          # quantification 8‑bits, moins de RAM/VRAM
    trust_remote_code=True,
)

def generer_reponse(prompt: str, max_new_tokens: int = 128) -> str:
    """Retourne la réponse du LLM à partir d’un prompt."""
    inputs = tokenizer(prompt, return_tensors="pt")
    output = model.generate(**inputs, max_new_tokens=max_new_tokens)
    return tokenizer.decode(output[0], skip_special_tokens=True)

# Exemple d’utilisation avec LangChain (facultatif)
def chain_simple(messages: List[str]) -> LLMResult:
    prompt = "\n".join(messages)
    réponse = generer_reponse(prompt)
    return LLMResult(generations=[[{"text": réponse}]])
```

### 4. `app/main.py` – API FastAPI

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from .agent import generer_reponse

app = FastAPI(title="Agent IA - Demo")

class Requete(BaseModel):
    prompt: str

@app.post("/chat")
async def chat(req: Requete):
    try:
        texte = generer_reponse(req.prompt)
        return {"response": texte}
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

### 5. Lancer le serveur localement

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Accédez à `http://localhost:8000/docs` pour tester l’endpoint `/chat` via l’interface Swagger générée automatiquement.

### 6. Tests unitaires simples (`tests/test_agent.py`)

```python
import pytest
from app.agent import generer_reponse

def test_generer_reponse():
    prompt = "Quel est le capital du Sénégal ?"
    réponse = generer_reponse(prompt, max_new_tokens=32)
    assert "Dakar" in réponse
```

Exécutez : `pytest -q`.

---

## Astuces pour travailler avec une connexion internet intermittente

| Problème | Solution pratique |
|----------|-------------------|
| **Téléchargement de gros modèles** | Utilisez `git lfs` pour récupérer les fichiers depuis un serveur local (ex. : un NAS dans le bureau). |
| **Installation de paquets** | Privilégiez le mode *offline* de pip : `pip download -r requirements.txt -d ./cache` puis `pip install --no-index --find-links=./cache -r requirements.txt`. |
| **Mises à jour Git** | Planifiez les `git pull` pendant les plages horaires où le réseau est plus stable (souvent tôt le matin). |
| **Synchronisation de notebooks** | Si vous utilisez Jupyter, stockez les notebooks dans un dépôt Git et poussez‑les uniquement lorsque la bande passante le permet. |
| **Sauvegarde du venv** | Archivez le répertoire `.venv` (`tar -czf venv.tar.gz .venv`) et conservez‑le sur un disque externe ; ainsi vous pouvez restaurer l’environnement sans télécharger à nouveau les paquets. |

---

## Points clés

- **Python 3.11** offre le meilleur compromis entre performance et compatibilité ; choisissez la distribution qui minimise le volume de téléchargement (Miniconda ou le mini‑installateur Windows).  
- Un **environnement virtuel** (`venv`) isole les dépendances ; activez‑le à chaque session de travail.  
- **VS Code** devient votre IDE principal dès que vous installez les extensions Python, Pylance et GitLens ; configurez‑le pour éviter les requêtes réseau inutiles.  
- **Git** fonctionne hors‑ligne ; créez des commits fréquents et utilisez des miroirs Git locaux pour réduire le temps de push/pull.  
- **pip** peut être redirigé vers des miroirs PyPI plus proches ; cela accélère l’installation de `transformers`, `langchain` et `fastapi`.  
- Les **modèles quantifiés** (8‑bits) permettent de charger des LLM lourds même sur des machines modestes et avec une bande passante limitée.  
- Un petit projet de démonstration (FastAPI + LangChain + Transformers) montre comment les pièces s’assemblent : le serveur d’API expose une fonction `generer_reponse` qui exploite le modèle quantifié.  
- En appliquant les bonnes pratiques de sauvegarde et de gestion du réseau, vous pouvez développer un agent IA complet même dans des environnements où l’accès internet est sporadique.