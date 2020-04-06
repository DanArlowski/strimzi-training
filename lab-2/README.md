# AMQ Streams / Strimzi Training - Lab 2

## Replicas

* Open `replicas/kafka.yaml` and have a look at all the different replicas configuration
* Deploy the Kafka cluster and wait until it is deployed
  * `oc apply -f replicas/kafka.yaml`
* Open `replicas/kafka-connect.yaml` and have a look at all the replicas configuration
* Deploy the Kafka Connect cluster and wait until it is deployed
  * `oc apply -f replicas/kafka-connect.yaml`
* Wait for the deployments to be ready and check the pods
  * `oc get pods`
* Edit the Kafka cluster and scale-up the Kafka brokers
  * `oc edit kafka my-cluster`

```yaml
  kafka:
    replicas: 5
```

* Observe how the scaling works
  * New pods `my-cluster-kafka-3` and `my-cluster-kafka-4` will be create
* Now try to scale down to 3 nodes
* Observe how the scaling works
  * Pods `my-cluster-kafka-3` and `my-cluster-kafka-4` will be deleted
  * Notice that AMQ Streams removes the pod one by one, not both at the same time
* Try to scale Kafka connect up and down using `oc edit kafkaconnect my-connect-cluster` and see that this was it works
* Observe which pods are created and deleted
* Delete the deployments
  * `oc delete kafkaconnect my-connect-cluster`
  * `oc delete kafka my-cluster`

## Resources

* Open `resources/kafka-with-resources.yaml` and have a look at all the different resource configuration
* Deploy the Kafka cluster and wait until it is deployed
  * `oc apply -f resources/kafka-with-resources.yaml`
* Try the OpenShift web console or the UI to check that the resources were correctly applied
* Try to change the resources:
  * `oc edit kafka my-cluster`
  * Change the resources for Kafka broker to very large number. Foŕ example change the memory request and limit for Kafka broker to 20GB.

```yaml
  kafka:
    replicas: 3
    storage:
      type: ephemeral
    resources:
      requests:
        memory: 20Gi
        cpu: 200m
      limits:
        memory: 20Gi
        cpu: 1000m
    listeners:
      plain: {}
      tls: {}
```

* Observe what happens when the pod cannot be scheduled because of insuficient memory:
  * Check that the pod will stay in the `Pending` state
  * Check the events related to the `Pending` pod
    * `oc get events | grep  _<pod_name>_`
    * You should see `FailedScheduling` event with the reason similar to _0/1 nodes are available: 1 Insufficient memory._
* Use `oc edit kafka my-cluster` again and revert the memory request and limit
  * This might take a lot of time, because the Cluster Operator will be waiting for the pod to start
  * Only after it timeouts, it will see the new changes and revert the memeory back again
  * This timeout is configured in Cluster Operator using the environment variable `STRIMZI_OPERATION_TIMEOUT_MS`. Default is 5 minutes.
* Delete the deployment
  * `oc delete kafka my-cluster`

## Configuration

* Open `configuration/kafka.yaml` and have a look at the configuration option used there for Apache Kafka
* Deploy the Kafka cluster and wait until it is deployed
  * `oc apply -f configuration/kafka.yaml`
* Check the Kafka pod logs using `oc logs my-cluster-kafka-0 -c kafka | more` or in the OpenShift webconsole
  * On the beginning of the log you will see the broker configuration printed
  * Check that it contains the values form the `Kafka` resource
  * Notice also the other options configured by AMQ Streams

```properties
# Provided configuration
transaction.state.log.replication.factor=3
default.replication.factor=3
offsets.topic.replication.factor=3
transaction.state.log.min.isr=1
```

* Check the Zookeeper pod logs using `oc logs my-cluster-zookeeper-0 -c zookeeper | more` or in the OpenShift webconsole
  * On the beginning of the log you will see the node configuration printed
  * Check that although you didn't specify any values in the `Kafka` resource, the default values are there
  * Notice also the other options configured by AMQ Streams

```properties
# Provided configuration
timeTick=2000
autopurge.purgeInterval=1
syncLimit=2
initLimit=5
```

* Edit the Kafka resource and change the Kafka configuration to:

```yaml
    config:
      broker.id: 0
      num.partitions: 1
      num.recovery.threads.per.data.dir: 1
      default.replication.factor: 3
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 1
      log.retention.hours: 168
      log.segment.bytes: 1073741824
      log.retention.check.interval.ms: 300000
      num.network.threads: 3
      num.io.threads: 8
      socket.send.buffer.bytes: 102400
      socket.receive.buffer.bytes: 102400
      socket.request.max.bytes: 104857600
      group.initial.rebalance.delay.ms: 0
```

* Wait for the rolling update to complete
* Check the Kafka pod logs using `oc logs my-cluster-kafka-0 -c kafka | more` or in the OpenShift webconsole
  * On the beginning of the log you will see the broker configuration printed
  * Check that the values were updated
* Delete the deployments
  * `oc delete kafka my-cluster`

## Storage

* Open `storage/kafka.yaml` and have a look at the storage configuration
* Deploy the Kafka cluster and wait until it is deployed
  * `oc apply -f storage/kafka.yaml`
* Check the Persistent storage used by Kafka and Zookeeper
  * Do `oc get pv` and look for the Persistent Volumes bound to the Kafka and Zookeeper pods
  * Do `oc get pvc` and look for the Persistent Volume Claims bound to the Kafka and Zookeeper pods.The PVCs are linking Kafka and Zookeeper to the volumes.
* Edit the Kafka resource and change the storage to ephemeral:

```yaml
apiVersion: kafka.strimzi.io/v1alpha1
kind: Kafka
metadata:
  name: my-cluster
spec:
  kafka:
    replicas: 3
    storage:
      type: ephemeral
    listeners:
      plain: {}
      tls: {}
  zookeeper:
    replicas: 3
    storage:
      type: ephemeral
  entityOperator:
    topicOperator: {}
    userOperator: {}
```

* Check Cluster Operator logs to see the warning about forbidden option
  * `oc logs $(oc get pod -l name=amq-streams-cluster-operator -o=jsonpath='{.items[0].metadata.name}') | grep WARN`
  * The warning will repeat until you revert the storage change
* Check that no rolling update and no storage change happens!
* Delete the deployments
  * `oc delete kafka my-cluster`
* Check that the PVs and PVCs used by the pods are still there after we deleted the deployment
  * `oc get pv`
  * `oc get pvc`
* Delete pvc
  * `oc delete pvc -l strimzi.io/cluster=my-cluster`
