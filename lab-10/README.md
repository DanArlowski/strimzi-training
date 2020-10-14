# AMQ Streams / Strimzi Training - Lab 10

## Mirror Maker 2

* Create two namespaces `demo-europe` and `demo-na`
  * `oc new-project demo-na`
  * `oc new-project demo-europe`

## Deploy Kafka clusters

* We need to deploy the Kafka clusters in their respective namespaces
  * `oc apply -f 01-kafka-europe.yaml -n demo-europe`
  * `oc apply -f 02-kafka-na.yaml -n demo-na`

## Deploy applications using the clusters

* Once the clusters are running, we can deploy some applications using them:
  * `oc apply -f 03-application-europe.yaml -n demo-europe`
  * `oc apply -f 04-application-na.yaml -n demo-na`
* Notice that both applications are producing to the topic `my-topic`
* Check that the applications work and consume only the local messages

## Deploy the Mirror Makers

* Now we can deploy the Mirror Makers 2
  * `oc apply -f 05-mirror-maker-2-europe.yaml -n demo-europe`
  * `oc apply -f 06-mirror-maker-2-na.yaml -n demo-na`
* MM2s should sync the topics
  * The Europe consumer should now be consuming from a copy of the NA topic and vice versa
  * Notice the topic names on both sides
