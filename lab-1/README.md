# Strimzi Training - Lab 1

* Open the file `examples/kafka/kafka-persistent.yaml` and get familiar with it
* Deploy the Kafka cluster in one of the namespaces which the Cluster Operator is watching
  * `oc apply -f examples/kafka/kafka-persistent.yaml -n amqstreams-demo-<userXX>`
* Watch as the Kafka cluster is deployed (3 zookeeper,3 kafka broker and my-cluster-entity-operator)
  * Using the OpenShift webconsole
  * Using the command line
    * `oc -n amqstreams-demo-<userXX> get pods -l strimzi.io/cluster=my-cluster -w`
* Edit the Kafka cluster:
  * From the command line do `oc edit kafka my-cluster` and change the following section to enable TLS client authentication and authorization for kafka. Change it from:

```
    listeners:
      plain: {}
      tls: {}
```

to:

```
    listeners:
      tls:
        authentication:
          type: tls
    authorization:
      type: simple
```

* Watch as the Cluster Operator does a rolling update to reconfigure Kafka
  * `oc -n amqstreams-demo-<userXX> get pods -l strimzi.io/cluster=my-cluster -w`
* Open the file `examples/topic/kafka-topic.yaml`
  * Edit the topic to set 3 partitions and 2 replicas
* Create the topic
  * `oc apply -f examples/topic/kafka-topic.yaml`
* List the topics:
  * `oc get kafkatopics`
* Open the file `examples/user/kafka-user.yaml`
  * Review the access rights configured in the file
* Create the user
  * `oc apply -f examples/user/kafka-user.yaml`
* Check the secret with the certificate of the newly created user
  * `oc get secret my-user -o yaml`
* List the users:
  * `oc get kafkausers`
* Check the _Hello World_ consumer and producer in `examples/hello-world/deployment.yaml`
  * Notice how it is using the secret created by the User Operator to load the TLS certificates
* Deploy the _Hello World_ producer and consumer
  * `oc apply -f examples/hello-world/deployment.yaml`
* Check the producer and consumer logs to verify that they are working (1000 messages)
  * `oc logs $(oc get pod -l app=hello-world-producer -o=jsonpath='{.items[0].metadata.name}') -f`
  * `oc logs $(oc get pod -l app=hello-world-consumer -o=jsonpath='{.items[0].metadata.name}') -f`
* Change the deployment configuration of the producer:
  * `oc edit deployment hello-world-producer`
  * And set the environment variable `TOPIC` to some other topic
  * Wait for the pod to restart and check it gets an _authorization error_ using `oc logs $(oc get pod -l app=hello-world-producer -o=jsonpath='{.items[0].metadata.name}') -f`
  * Edit the KafkaUser resource with `oc edit kafkauser my-user` and update the access rights to allow it to use the new topic
  * Check the logs again to see how it is now authorized to produce to the new topic