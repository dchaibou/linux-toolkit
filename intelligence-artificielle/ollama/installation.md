# Guide Complet : Installer son IA locale avec Docker (Ollama + Open WebUI)

Ce guide vous permet de déployer une solution d'Intelligence Artificielle 100% privée, sans abonnement, fonctionnant entièrement sur votre propre matériel. Nous allons coupler **Ollama** (le moteur) avec **Open WebUI** (l'interface graphique).

## 📋 Prérequis

* **Docker** installés.
* **Système :** Linux, Windows (via WSL2) ou macOS.
* **Matériel :** 8 Go de RAM minimum (16 Go recommandés). Un GPU NVIDIA est un plus majeur pour la rapidité.

---

## 🛠 Étape 1 : Préparation du réseau et du stockage

Pour que les deux services communiquent de manière stable avec des adresses IP fixes, nous créons un réseau Docker dédié.

```bash
# Création du réseau avec un sous-réseau spécifique
docker network create --subnet=172.20.0.0/16 ai-network

# Création du dossier pour stocker les modèles (persistance des données)
sudo mkdir -p /opt/ollama
sudo chmod 777 /opt/ollama

```

---

## 🧠 Étape 2 : Installation du moteur Ollama

Ollama est le backend qui télécharge et exécute les modèles (Llama 3, Mistral, Phi-3, etc.).

### Option A : Installation standard (CPU)

```bash
docker run -d \
  --network ai-network \
  --ip 172.20.0.30 \
  -v /opt/ollama:/root/.ollama \
  --name ollama \
  --restart always \
  ollama/ollama

```

### Option B : Installation optimisée (GPU NVIDIA)

*Nécessite le [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html).*

```bash
docker run -d \
  --network ai-network \
  --ip 172.20.0.30 \
  --gpus all \
  -v /opt/ollama:/root/.ollama \
  --name ollama \
  --restart always \
  ollama/ollama

```

---

## 💻 Étape 3 : Installation de l'interface Open WebUI

Open WebUI offre une expérience utilisateur fluide identique à ChatGPT, avec gestion des documents (RAG), historique et multi-utilisateurs.

```bash
docker run -d \
  --network ai-network \
  --ip 172.20.1.100 \
  -p 3000:8080 \
  -e OLLAMA_BASE_URL=http://172.20.0.30:11434 \
  -v open-webui:/app/backend/data \
  --name open-webui \
  --restart always \
  ghcr.io/open-webui/open-webui:main

```

---

## 🚀 Étape 4 : Premier démarrage et configuration

1. **Accès :** Ouvrez votre navigateur sur `http://localhost:3000`.
2. **Compte Admin :** Créez votre compte. Le premier utilisateur devient l'administrateur système.
3. **Importation d'un modèle :**
   * Allez dans **Paramètres** > **Modèles**.
   * Dans le champ "Pull a model from Ollama.com", entrez `llama3` ou `mistral`.
   * Cliquez sur l'icône de téléchargement.
4. **Discutez :** Une fois le téléchargement fini, sélectionnez le modèle en haut de la page d'accueil et commencez à discuter.

---

## 🔍 Pourquoi cette configuration ?

| Choix technique          | Avantage                                                                          |
| ------------------------ | --------------------------------------------------------------------------------- |
| **IP Statique**          | Évite la perte de connexion entre l'interface et le moteur lors des redémarrages. |
| **Volume `/opt/ollama`** | Vos modèles ne sont pas supprimés si vous supprimez le conteneur.                 |
| **Réseau isolé**         | Sécurise les flux de données entre vos conteneurs.                                |

---

## 💡 Astuces de dépannage

* **Vérifier les logs :** `docker logs -f ollama` pour voir si le moteur tourne correctement.
* **Tester la connexion :** Depuis la machine hôte, essayez d'accéder à `http://172.20.0.30:11434`. Vous devriez voir le message *"Ollama is running"*.
* **Mise à jour :** Pour mettre à jour l'interface, faites simplement un `docker pull ghcr.io/open-webui/open-webui:main` et relancez le conteneur.

---

> **Note de sécurité :** Par défaut, cette installation est accessible sur votre réseau local via l'IP de votre machine sur le port 3000. Pensez à configurer un firewall si vous souhaitez limiter l'accès.
