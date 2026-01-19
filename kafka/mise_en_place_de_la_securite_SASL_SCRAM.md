# Guide Ultime : Kafka KRaft avec Sécurité SASL/SCRAM et ACLs

Ce guide détaille le déploiement d'un broker Kafka moderne (**v4.x**) en mode **KRaft** (sans Zookeeper). Nous allons sécuriser l'infrastructure via une authentification **SASL SCRAM-SHA-256** et un contrôle d'accès granulaire par **ACLs**.

## 1. Architecture et Objectifs

L'objectif est de sécuriser les échanges entre deux services distincts : un donneur d'ordre (`client1`) et un exécutant (`client2`).

- **Mode KRaft** : Gestion simplifiée des métadonnées sans Zookeeper.
- **SASL SCRAM** : Authentification par sel et hachage (Salted Challenge Response Authentication Mechanism).
- **ACLs (Access Control Lists)** : Application du principe du moindre privilège pour restreindre les actions sur les topics.

---

## 2. Configuration de la Sécurité

### Authentification : `config/kafka_server_jaas.conf`

Ce fichier permet au broker de s'identifier lui-même pour les opérations internes.

```conf
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="admin-pw"
    user_admin="admin-pw";
};

```

### Paramètres du Broker : `config/server.properties`

Voici les réglages clés pour activer la couche de sécurité :

```ini
# =============================================================================
#  CONFIGURATION KAFKA KRAFT AVEC SASL SCRAM-SHA-256
# =============================================================================

############################# Server Basics #############################
process.roles=broker,controller
node.id=1
controller.quorum.voters=1@localhost:9093

############################# Socket Server Settings #############################
# Liste des listeners
# 9092 : Communication interne entre brokers (Non sécurisée pour simplifier)
# 9093 : Quorum des contrôleurs (Non sécurisé)
# 9094 : Accès client (Sécurisé via SASL SCRAM)
listeners=PLAINTEXT://:9092,CONTROLLER://:9093,SASL_PLAINTEXT://:9094

# Mapping des protocoles
# Important : PLAINTEXT doit pointer vers PLAINTEXT pour la com interne
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SASL_PLAINTEXT:SASL_PLAINTEXT

# Listener utilisé pour la communication entre brokers
inter.broker.listener.name=PLAINTEXT

# Listener utilisé par le contrôleur
controller.listener.names=CONTROLLER

# Adresse annoncée aux clients
advertised.listeners=PLAINTEXT://localhost:9092,SASL_PLAINTEXT://localhost:9094

############################# SASL Configuration #############################
# Activer le mécanisme SCRAM
sasl.enabled.mechanisms=SCRAM-SHA-256

# Configuration de l'authentification pour le listener SASL_PLAINTEXT
# Ce bloc définit l'utilisateur "admin" qui permet au broker de s'identifier
LISTENER.NAME.SASL_PLAINTEXT.SCRAM-SHA-256.SASL.JAAS.CONFIG=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-pw";

# Utilisation de l'Authorizer standard pour KRaft
authorizer.class.name=org.apache.kafka.metadata.authorizer.StandardAuthorizer

# Définir les super-utilisateurs (ceux qui ont tous les droits)
super.users=User:admin

allow.everyone.if.no.acl.found=true

############################# Log Basics #############################
log.dirs=/tmp/kraft-combined-logs
num.partitions=1
num.recovery.threads.per.data.dir=1

############################# Internal Topic Settings #############################
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1

############################# Log Retention Policy #############################
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

############################# Network Settings #############################
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
```

Les paramètres critiques pour activer la couche de sécurité sont :

- **Listeners** : Le port `9094` est dédié au trafic sécurisé (`SASL_PLAINTEXT`).
- **Authorizer** : Utilisation de `StandardAuthorizer` pour KRaft.
- **Super Users** : `User:admin` possède les pleins pouvoirs pour configurer les ACLs.

---

## 3. Déploiement avec Docker : `docker-compose.yml`

Le passage à Kafka 4.x nécessite le formatage du stockage KRaft avec un **Cluster ID** unique avant le premier lancement.

```yaml
version: "3.8"

services:
  kafka-auth:
    image: apache/kafka:4.1.0
    container_name: kafka-auth
    environment:
      KAFKA_OPTS: "-Djava.security.auth.login.config=/opt/kafka/config/kafka_server_jaas.conf"
    volumes:
      - ./config/kafka_server_jaas.conf:/opt/kafka/config/kafka_server_jaas.conf
      - ./config/server.properties:/opt/kafka/config/server.properties
      - ./secrets:/opt/kafka/secrets
    ports:
      - "9092:9092"
      - "9094:9094"
    command:
      - /bin/bash
      - -c
      - |
        if [ ! -f /tmp/kraft-combined-logs/meta.properties ]; then
          echo "Formatting storage..."
          KAFKA_CLUSTER_ID=$$(/opt/kafka/bin/kafka-storage.sh random-uuid)
          /opt/kafka/bin/kafka-storage.sh format -t $$KAFKA_CLUSTER_ID -c /opt/kafka/config/server.properties
        fi
        /opt/kafka/bin/kafka-server-start.sh /opt/kafka/config/server.properties
    networks:
      my-network:
        ipv4_address: 172.20.0.61

networks:
  my-network:
    external: true
```

---

## 4. Gestion des Identités et Droits (ACLs)

### Tableau des permissions

| Utilisateur | Topic : `request-topic`  | Topic : `response-topic` |
| ----------- | ------------------------ | ------------------------ |
| **admin**   | Full Access (Super User) | Full Access (Super User) |
| **client1** | **Write** (Producteur)   | **Read** (Consommateur)  |
| **client2** | **Read** (Consommateur)  | **Write** (Producteur)   |

### Configuration du Cluster (Scripting)

Exécutez ces commandes pour initialiser les topics et les permissions via le port interne `9092` :

```bash
# 1. Création des topics
docker exec kafka-auth /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic request-topic
docker exec kafka-auth /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic response-topic

# 2. Création de l'utilisateur Admin
docker exec kafka-auth /opt/kafka/bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=admin-pw]' --entity-type users --entity-name admin

# 3. Configuration Client 1
docker exec kafka-auth /opt/kafka/bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=client1-pw]' --entity-type users --entity-name client1
docker exec kafka-auth /opt/kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal User:client1 --operation Write --topic request-topic
docker exec kafka-auth /opt/kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal User:client1 --operation Read --topic response-topic

# 4. Configuration Client 2
docker exec kafka-auth /opt/kafka/bin/kafka-configs.sh --bootstrap-server localhost:9092 --alter --add-config 'SCRAM-SHA-256=[password=client2-pw]' --entity-type users --entity-name client2
docker exec kafka-auth /opt/kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal User:client2 --operation Write --topic response-topic
docker exec kafka-auth /opt/kafka/bin/kafka-acls.sh --bootstrap-server localhost:9092 --add --allow-principal User:client2 --operation Read --topic request-topic

```

---

## 5. Fichiers de Propriétés Clients

Créez ces fichiers dans votre dossier `./secrets` pour permettre aux clients de s'authentifier sur le port **9094** lors de l'utilisaion du CLI.

**`admin.properties`**

```ini
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="admin" password="admin-pw";

```

**`client1.properties`**

```ini
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="client1" password="client1-pw";

```

**`client2.properties`**

```ini
security.protocol=SASL_PLAINTEXT
sasl.mechanism=SCRAM-SHA-256
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="client2" password="client2-pw";

```

---

## 6. Tests et Validation

### Cas n°1 : Succès Nominal

Testez la production et consommation croisée sur le port **9094**.

**Production (Client 1) :**

```bash
docker exec -it kafka-auth /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9094 \
  --topic request-topic \
  --producer.config /opt/kafka/secrets/client1.properties

```

**Consommation (Client 2) :**

```bash
docker exec -it kafka-auth /opt/kafka/bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9094 \
  --topic request-topic \
  --from-beginning \
  --consumer.config /opt/kafka/secrets/client2.properties

```

### Cas n°2 : Échec d'Autorisation (Test ACL)

Tentez de produire avec `client1` sur un topic où il n'a qu'un droit de lecture :

```bash
docker exec -it kafka-auth /opt/kafka/bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9094 \
  --topic response-topic \
  --producer.config /opt/kafka/secrets/client1.properties

```

_Résultat : Une erreur `TopicAuthorizationException` doit s'afficher._

---

## Conclusion

Cette architecture KRaft combinée à SASL/SCRAM et aux ACLs offre un niveau de sécurité robuste pour vos environnements de production. Vous garantissez ainsi l'intégrité de vos flux et l'isolation stricte de vos services.
