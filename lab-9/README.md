# AMQ Streams / Strimzi Training - Lab 9

## HTTP Bridge

* Go to the `http-bridge` directory
  * `cd http-bridge`
* Create a topic `my-topic`
  * `oc apply -f topic.yaml`
* Deploy the Kafka cluster
  * `oc apply -f kafka-ephemeral.yaml`
* Deploy the HTTP Bridge and wait for it to be ready
  * `oc apply -f http-bridge.yaml`
* Expose the bridge using OpenShift Route to allow it being used from our local computers
  * `oc apply -f http-route.yaml`
  * Get the address of the route `export BRIDGE="$(oc get routes my-bridge -o jsonpath='{.status.ingress[0].host}')"`

### Sending messages

* Send messages using a simple `POST` call
  * Notice the `content-type` header which is important!
```sh
curl -X POST $BRIDGE/topics/my-topic \
  -H 'content-type: application/vnd.kafka.json.v2+json' \
  -d '{"records":[{"key":"message-key","value":"message-value"}]}'
```

### Receiving messages

* First, we need to create a consumer
```sh
curl -X POST $BRIDGE/consumers/my-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "consumer1",
    "format": "json",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": false,
    "fetch.min.bytes": 512,
    "consumer.request.timeout.ms": 30000
  }'
```

* Then we need to subscribe the consumer to the topics it should receive from
```sh
curl -X POST $BRIDGE/consumers/my-group/instances/consumer1/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{"topics": ["my-topic"]}'
```

* And then we can consume the messages (can be called repeatedly - e.g. in a loop)
```sh
curl -X GET $BRIDGE/consumers/my-group/instances/consumer1/records \
  -H 'accept: application/vnd.kafka.json.v2+json'
```

* At the end we should close the consumer
```sh
curl -X DELETE $BRIDGE/consumers/my-group/instances/consumer1
```

* Delete the Kafka cluster
  * `oc delete kafka my-cluster`

## Debezium

* Go to the `debezium` directory
  * `cd debezium`
* Deploy the Address Book application
  * `oc apply -f address-book.yaml`
  * This YAML deploys a MySQL database, and a simple application using it as a address book
  * Check it in OpenShift, make sure it works and you are able to add / edit / remove addresses
* Deploy the Kafka cluster
  * `oc apply -f kafka-ephemeral.yaml`
* Deploy Kafka Connect
  * `oc apply -f kafka-connect.yaml`
* Get the address of the Kafka Connect REST interface
  * `export CONNECT="$(oc get routes my-connect-cluster -o jsonpath='{.status.ingress[0].host}')"`

### Create the connector

* The connector can be created using the `debezium-connector.json` file
  * POST it to the Kafka Connect REST interface
```sh
curl -X POST $CONNECT/connectors \
  -H "Content-Type: application/json" \
  --data "@debezium-connector.json"
```
* Check that status of the connector
```sh
curl $CONNECT/connectors/debezium-connector/status | jq
```

### Debezium messages

* We will use the HTTP Bridge deployed in previous section to get the Debezium messages
* Create a consumer
```sh
curl -X POST $BRIDGE/consumers/debezium-group \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{
    "name": "debezium-consumer",
    "format": "json",
    "auto.offset.reset": "earliest",
    "enable.auto.commit": "false",
    "fetch.min.bytes": "512",
    "consumer.request.timeout.ms": "30000"
  }'
```
* Subscribe to the Kafka topic where we get the messages for our MySQL Address book table
```sh
curl -X POST $BRIDGE/consumers/debezium-group/instances/debezium-consumer/subscription \
  -H 'content-type: application/vnd.kafka.v2+json' \
  -d '{"topics": ["dbserver1.inventory.customers"]}'
```
* Read the messages created by the initial state of the database (you might need to run it multiple time before you get all the messages)
```sh
curl -X GET $BRIDGE/consumers/debezium-group/instances/debezium-consumer/records \
  -H 'accept: application/vnd.kafka.json.v2+json' | jq
```
* Go to the address book UI and make some changes
* Observe the new Debezium messages by calling the `GET` again
```sh
curl -X GET $BRIDGE/consumers/debezium-group/instances/debezium-consumer/records \
  -H 'accept: application/vnd.kafka.json.v2+json' | jq
```
* You can also consume the messages using a regular Kafka client:
```sh
oc run kafka-consumer -ti --image=strimzi/kafka:0.12.1-kafka-2.2.1 --rm=true --restart=Never -- bin/kafka-console-consumer.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --topic dbserver1.inventory.customers --from-beginning --property print.key=true --property key.separator=" - "
```

* Delete the deployments
  * `oc delete kafkaconnect my-connect-cluster`
  * `oc delete kafka my-cluster`
  * `oc delete all -l app=mysql`
  * `oc delete kafkabridges my-bridge`
