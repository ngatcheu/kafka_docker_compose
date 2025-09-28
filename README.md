# kafka_docker_compose ce repo permet de deployer apache kafka avec docker compose afin de faire du streaming distribuée et de messagerie publish/subscribe.


## Configuration Kafka en mode KRaft

Ce tableau décrit les principales variables de configuration :

| Variable                                | Exemple                                       | Description |
|-----------------------------------------|-----------------------------------------------|-------------|
| **KAFKA_KRAFT_MODE**                    | `"true"`                                      | Active le mode **KRaft** (Kafka sans ZooKeeper). |
| **KAFKA_PROCESS_ROLES**                 | `controller,broker`                           | Rôles de l’instance : **contrôleur** et **broker**. |
| **KAFKA_NODE_ID**                       | `1`                                           | Identifiant unique de ce nœud Kafka. |
| **KAFKA_CONTROLLER_QUORUM_VOTERS**      | `"1@localhost:9093"`                          | Définit le quorum des contrôleurs (ID@host:port). Ici, un seul contrôleur. |
| **KAFKA_LISTENERS**                     | `PLAINTEXT://localhost:9092,CONTROLLER://localhost:9093` | Liste des adresses et ports où Kafka écoute. |
| **KAFKA_LISTENER_SECURITY_PROTOCOL_MAP**| `PLAINTEXT:PLAINTEXT,CONTROLLER:PLAINTEXT`    | Mappe chaque listener à un protocole de sécurité. |
| **KAFKA_INTER_BROKER_LISTENER_NAME**    | `PLAINTEXT`                                   | Listener utilisé pour la communication entre brokers. |
| **KAFKA_CONTROLLER_LISTENER_NAMES**     | `CONTROLLER`                                  | Listener utilisé par les contrôleurs pour communiquer entre eux. |
| **KAFKA_ADVERTISED_LISTENERS**          | `PLAINTEXT://localhost:9092`                  | Adresse/port annoncés aux clients pour se connecter au broker. |
| **KAFKA_LOG_DIRS**                      | `/var/lib/kafka/data`                         | Répertoire où sont stockés les journaux Kafka. |
| **KAFKA_AUTO_CREATE_TOPICS_ENABLE**     | `"true"`                                      | Active la création automatique des topics. |
| **KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR** | `1`                                         | Facteur de réplication du topic interne `__consumer_offsets`. |
| **KAFKA_LOG_RETENTION_HOURS**           | `168` (7 jours)                               | Durée de rétention des messages dans les logs. |
| **KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS** | `0`                                         | Temps d’attente avant le premier rééquilibrage d’un consumer group. |
| **CLUSTER_ID**                          | `"Mk3OEYBSD34fcwNTJENDM2Qk"`                  | Identifiant unique du cluster Kafka (utilisé en mode KRaft). |


## Déploiement simple
  ```
  docker compose -f compose-simple-apache-kafka.yaml up -d
  ```

## Déploiement haute disponibilité
  ```
  docker compose -f compose-cluster-apache-kafka.yaml up -d
  ```

## Se connecter aux différents containers
  ```
  docker exec --workdir /opt/kafka/bin/ -it broker-1 sh
  docker exec --workdir /opt/kafka/bin/ -it broker-2 sh
  docker exec --workdir /opt/kafka/bin/ -it broker-3 sh
  docker exec --workdir /opt/kafka/bin/ -it controller-1 sh
  docker exec --workdir /opt/kafka/bin/ -it controller-2 sh
  docker exec --workdir /opt/kafka/bin/ -it controller-3 sh
  ```

🔹 Scénario 1 : Réplication des données
  Créer 3 topics a partir le broker 1 répliqué (3 brokers)
  ```
  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic userlogin --partitions 1 --replication-factor 3

  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic usersignup --partitions 1 --replication-factor 3

  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic usersignout --partitions 1 --replication-factor 3
  ```
 
 Lister les topics sur le broker-1 et broker-2 et voir qu'on a le meme resultat
 ```
  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --list
  ```
 Vérifier la répartition des leaders et ISR(réplicas synchronisés et fiables)
 ```
  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --describe --topic userlogin.
  ```

🔹 Scénario 2 : Résilience des Brokers
   se mettre sur le broker 1 et Produire des messages via un nouveau topic

 ```
  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic userlogin2 --partitions 1 --replication-factor 3
  ```
  Produire des messages
 ```
  ./kafka-console-producer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic userlogin2
  ```
  (tape message 1, message 2, etc.)
  
 ```
  ./kafka-console-producer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic userlogin2
  ```
  Consommer depuis un autre broker
 ```
   ./kafka-console-consumer.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --topic userlogin2 --from-beginning
   ```
  Stopper un broker 1
 ```
   docker stop broker-1
   ```
  👉 Le consommateur continue de lire.
     se placer sur le bocker 3  produire un messages sur un nouvel topic

```
  ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --create --topic userlogin3 --partitions 1 --replication-factor 3
  ```    
  Produire un message
 ```
   ./kafka-console-producer.sh --bootstrap-server broker-2:19092,broker-3:19092 --topic userlogin3
   ```
  Consommer le message
 ```
   ./kafka-console-producer.sh --bootstrap-server broker-2:19092,broker-3:19092 --topic userlogin3
   ```

  Lister les topics 
 ```
   ./kafka-topics.sh --bootstrap-server broker-2:19092,broker-3:19092 --list
   ```

  Relancer le broker et vérifier la synchro
 ```
   docker start broker-1
   ```
  Se connecter au brocker 1
  Lister les topics 
 ```
   ./kafka-topics.sh --bootstrap-server broker-1:19092,broker-2:19092,broker-3:19092 --list
   ```
  👉 Tu devrais voir broker-1 de nouveau dans la liste des ISR (in-sync replicas).


🔹 Scénario 3 : Tolérance aux pannes des Controllers
   Vérifier quel controller est leader

 ```
   docker exec --workdir /opt/kafka/bin/ -it controller-1 sh docker start broker-1
   ```

 ```
   ./kafka-metadata-quorum.sh --bootstrap-server broker-1:19092 describe --status
   ```

 👉 Regarde la ligne Leader Id.

    Stopper le controller leader

 ```
   docker stop controller-2
   ```

    Vérifier la nouvelle élection

 ```
   docker exec --workdir /opt/kafka/bin/ -it controller-1 sh
   ```
 ```
   ./kafka-metadata-quorum.sh --bootstrap-server broker-1:19092 describe --status
   ```
   
  👉 Tu verras que controller-1 est maintenant leader.

    Redémarrer le controller
 ```
   docker start controller-2
   ```
    Puis revérifier avec la commande describe --status.

 ```
   ./kafka-metadata-quorum.sh --bootstrap-server broker-1:19092 describe --status
   ```
    Deux situations se presentes soit 
    * il est follower 
    * ou alors leader apres une reelection

# Liens Utiles
- [Apache kafka Documentation](https://kafka.apache.org/documentation)
- [Docker compose Apache kafka](https://hub.docker.com/r/apache/kafka)
- [Kafkio ui](https://kafkio.com)


