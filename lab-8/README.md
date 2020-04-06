# AMQ Streams / Strimzi Training - Lab 8

## Customizing deployments

* Check the cluster deployment in `templating/kafka.yaml`
  * Notice the `template` fields with their definitions of labels and annotations
  * For pods we also define the termination grace period which defines how much time do we give to the pods to terminate
  * Missing in this examples are image pull secrets and security context
* Apply the Kafka resource
  * `oc apply -f templating/kafka.yaml`
* Check the labels and annotations assigned to the different resources such as services, pods, etc.
  * `oc get services -o='custom-columns=NAME:.metadata.name,LABELS:.metadata.labels,ANNOTATIONS:.metadata.annotations'`
* On your own
  * Try to use some other customization such as Security Context or Image Pull Secrets
* Delete the Kafka cluster
  * `oc delete kafka my-cluster`

## JBOD Storage

* Check the cluster deployment in `jbod/kafka.yaml`
  * Notice the `storage` section in `.spec.kafka`
  * It defines two persistent disks
* Apply the Kafka resource
  * `oc apply -f jbod/kafka.yaml`
* Check the volumes which were created
  * You should see 6 Persistent Volume Claims (+3 for Zookeeper): `oc get pvc`
  * You should see 6 Persistent Volume (+3 for Zookeeper): `oc get pv | grep Bound`
  * Exec into the pod and check how the volumes are mounted and configured
    * Open a terminal inside the pod `oc exec -ti my-cluster-kafka-0 -- bash`
    * Check the `/var/lib/kafka` directory and the configuration file in `/tmp/strimzi.properties` (look for the `log.dirs` property)
* Delete the Kafka cluster
  * `oc delete kafka my-cluster`