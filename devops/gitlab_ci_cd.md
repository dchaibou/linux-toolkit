# Mise en place d’un CI/CD COMPLET avec GitLab et Docker

Ce guide vous permet de déployer une infrastructure **GitLab CI/CD autonome** et de configurer des pipelines modernes et sécurisés.

| Service                    | Container name    | IP Address      | Description                                   |
| :------------------------- | :---------------- | :-------------- | :-------------------------------------------- |
| GitLab Server              | `gitlab`          | **172.25.0.10** | Serveur principal GitLab (Web, Git, Registry) |
| GitLab Runner              | `gitlab-runner`   | **172.25.0.11** | Exécuteur de jobs de pipeline                 |
| Registry Privé (Optionnel) | `gitlab-registry` | **172.25.0.12** | Stockage local pour les images Docker         |

## Partie 1 : Infrastructure Docker et Services

### 1. Créer un Réseau Docker Personnalisé

Créer un réseau de type `bridge` pour isoler les services et garantir une communication fluide par IP/Nom.

```bash
docker network create \
 --driver bridge \
 --subnet 172.25.0.0/16 \
 --gateway 172.25.0.1 \
 gitlab-network
```

### 2. Créer et Configurer le Serveur GitLab

On expose les ports standards et on s'assure de la **persistance des données** grâce aux volumes.

```bash
sudo mkdir -p /srv/gitlab/{config,logs,data}

docker run -d \
 --name gitlab \
 --network gitlab-network \
 --ip 172.25.0.10 \
 --publish 443:443 \
 --publish 80:80 \
 --publish 22:22 \
 --volume /srv/gitlab/config:/etc/gitlab \
 --volume /srv/gitlab/logs:/var/log/gitlab \
 --volume /srv/gitlab/data:/var/opt/gitlab \
 --restart always \
 gitlab/gitlab-ce:latest
```

> **Accès :** GitLab sera accessible via l'adresse IP de votre hôte sur les ports 80 et 443. Pour l'accès interne au Runner, l'URL est **[http://172.25.0.10](http://172.25.0.10)**.

### 3. Créer le Conteneur GitLab Runner

Le Runner a besoin d'accéder au **socket Docker** (`/var/run/docker.sock`) de l'hôte pour pouvoir lancer des conteneurs pour les jobs (Executor `docker`).

```bash
sudo mkdir -p /srv/gitlab-runner/config

docker run -d \
 --name gitlab-runner \
 --network gitlab-network \
 --ip 172.25.0.11 \
 -v /srv/gitlab-runner/config:/etc/gitlab-runner \
 -v /var/run/docker.sock:/var/run/docker.sock \
 --restart always \
 gitlab/gitlab-runner:latest
```

---

## Partie 2 : Configuration et Sécurité

### 4. Enregistrer le GitLab Runner

Une fois les conteneurs démarrés (laissez 5 minutes à GitLab pour initialiser), vous devez lier le Runner à l'instance GitLab.

1. **Récupérer le Token :** Dans l'interface Web GitLab (**<http://172.25.0.10>**), allez dans **Admin Area \> Overview \> Runners** (ou **Settings \> CI/CD \> Runners** de votre projet). Récupérez l'**URL** et le **Registration Token**.

2. **Exécuter l'enregistrement :**

    ```bash
    docker exec -it gitlab-runner gitlab-runner register
    ```

    - **GitLab instance URL:** `http://172.25.0.10`
    - **Registration token:** _(Coller le token)_
    - **Description:** `Mon Runner Docker Local`
    - **Tags:** `docker, build, deploy` (**TRES IMPORTANT**)
    - **Executor:** `docker`
    - **Default Docker Image:** `alpine:latest`

### 5. Sécurité du Déploiement SSH (Best Practice)

Pour le déploiement sur un serveur distant, il faut utiliser des **clés SSH sécurisées** via les variables CI/CD de GitLab.

1. **Générez une paire de clés SSH** dédiée pour le CI/CD (sans passphrase).
2. **Ajoutez la clé publique** au fichier `~/.ssh/authorized_keys` de l'utilisateur sur le serveur distant.
3. **Configurez une Variable Secrète dans GitLab :**
    - Allez dans votre projet : **Settings \> CI/CD \> Variables**.
    - Cliquez sur **Add variable**.
    - **Key (Clé) :** `SSH_PRIVATE_KEY`
    - **Value (Valeur) :** Collez le **contenu complet** de la clé privée SSH (commençant par `-----BEGIN...`).
    - **Protect variable :** (Cochez pour n'être disponible que sur les branches protégées, ex: `main`).
    - **Mask variable :** (Cochez pour masquer la valeur dans les logs du pipeline).

---

## Partie 3 : Pipelines CI/CD Avancés (`.gitlab-ci.yml`)

Utilisez les **tags** définis lors de l'enregistrement du Runner pour forcer l'exécution sur votre Runner local.

### Exemple 1 : Projet Node.js (avec Docker Build)

Ce pipeline montre la compilation, le test et la **construction d'une image Docker** puis le déploiement sécurisé via SSH.

```yaml
stages:
  - build
  - test
  - deploy

variables:
  # Nom de l'image Docker
  IMAGE_NAME: mon-app-node
  # Adresse du serveur de déploiement (Mettre en variable CI/CD si sensible)
  DEPLOY_SERVER: user@192.168.1.50

# Job de compilation et installation des dépendances
build_job:
  stage: build
  image: node:20
  tags: [docker, build]
  script:
    - npm ci # Installation propre à partir du package-lock.json
    - npm run build # Commande de compilation (ex: pour React/Vue/Angular)
  artifacts:
    paths:
      - build/ # Dossier de l'application compilée
    expire_in: 1 day

# Job de test unitaire
test_job:
  stage: test
  image: node:20
  tags: [docker]
  script:
    - npm ci
    - npm test
  dependencies: [] # N'a pas besoin du build, seulement du code

# Déploiement via SSH sécurisé
deploy_job:
  stage: deploy
  image: alpine/helm:3.14.0 # Image légère avec SSH et rsync
  tags: [docker, deploy]
  only:
    - main # Déploiement uniquement sur la branche main
  before_script:
    - apk add --no-cache openssh-client rsync bash # S'assurer que les outils sont là
    # Configuration de la clé SSH à partir de la variable sécurisée
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - # Ajout de la clé privée
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - ssh-keyscan ${DEPLOY_SERVER#*@} >> ~/.ssh/known_hosts # Ajout de l'hôte connu
    - chmod 600 ~/.ssh/known_hosts
  script:
    - echo "Déploiement du build sur le serveur distant: $DEPLOY_SERVER"
    # Transfert des fichiers compilés
    - rsync -avz --delete build/ $DEPLOY_SERVER:/var/www/mon-app/
    # Redémarrage du service (ex: PM2 ou Nginx)
    - ssh $DEPLOY_SERVER "pm2 restart mon-app || true"
  dependencies:
    - build_job # Dépendance au job de compilation
```

### Exemple 2 : Projet Java/Spring Boot (avec Maven)

Adaptation pour un projet Java utilisant Maven.

```yaml
stages:
  - build
  - test
  - package
  - deploy

# Image Docker avec Maven et JDK
image: maven:3.9.5-jdk-17
tags: [docker, build]

variables:
  MAVEN_CLI_OPTS: "-s .m2/settings.xml --batch-mode"
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

# Configuration du cache pour les dépendances Maven
cache:
  paths:
    - .m2/repository/

build_job:
  stage: build
  script:
    - mvn $MAVEN_CLI_OPTS compile

test_job:
  stage: test
  script:
    - mvn $MAVEN_CLI_OPTS test

package_job:
  stage: package
  script:
    - mvn $MAVEN_CLI_OPTS package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 week

# Utiliser le même modèle de déploiement SSH sécurisé que l'exemple Node.js
deploy_job:
  stage: deploy
  image: alpine:latest
  tags: [docker, deploy]
  only:
    - main
  # --- Scripts before_script et script à reprendre du déploiement Node.js ---
  before_script:
    - apk add --no-cache openssh-client rsync bash
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan user@server.com >> ~/.ssh/known_hosts
    # ... (le reste de la configuration SSH) ...
  script:
    - rsync -avz target/*.jar user@server:/opt/app/
    - ssh user@server "systemctl restart my-java-app"
  dependencies:
    - package_job
```

---
