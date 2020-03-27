# AMQ Streams / Strimzi Training - Lab 5

* Install the Kafka cluster
  * `oc apply -f kafka.yaml`

## Cluster Operator Logging

* Edit the Cluster Operator deployment
  * `oc edit deployment amq-streams-cluster-operator-v1.4.0`
  * And change the log level:

```yaml
        env:
        - name: STRIMZI_LOG_LEVEL
          value: DEBUG
```

* Check the new information which it contains with the DEBUG log level

## Logging in other components

* Deploy Kafka Connect `oc apply -f connect.yaml`
  * Watch Kafka Connect log to see it fail because of missing ACL rights
* Change log levels for Kafka brokers
* Edit the Kafka resource using `oc edit kafka my-cluster`
  * Change the Kafka log levels. For example

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: Kafka
spec:
  kafka:
    # ...
    logging:
      type: inline
      loggers:
        log4j.logger.kafka.authorizer.logger: INFO
    # ...
```

* Wait for the rolling update of Kafka brokers to finish
* Check Kafka logs to see authorization errors
  * Try to find out which access rights are missing and add them
* _On your own: Try to configure log levels also in other componenets_

## Configure logging using config map

* Use the file `log4j.properties` to create a new config map
  * `oc create configmap kafka-broker-logging --from-file log4j.properties`
* Edit the Kafka resource using `oc edit kafka my-cluster`
  * Change the Kafka log levels. For example

```yaml
apiVersion: kafka.strimzi.io/v1beta1
kind: Kafka
spec:
  kafka:
    # ...
    logging:
      type: external
      name: kafka-broker-logging
    # ...
```

* Wait for the rolling update of Kafka brokers to finish
* Check that the logging configuration has been updated
* _On your own: Try to configure log levels using config map also in other componenets_
* Delete the deployments
  * `oc delete kafkaconnect my-connect-cluster`
  * `oc delete kafka my-cluster`