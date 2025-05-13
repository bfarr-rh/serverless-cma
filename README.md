
The following is a demo on an OpenShift 4.18 custom metrics autoscaler.

## Prerequisites

1. **Login as admin**
2. **Install the following operators**:
   - Red Hat OpenShift Serverless - `1.35.1` (as of time of writing)
   - Streams for Apache Kafka - `2.9.01` (as of time of writing)
   - Custom Metrics Autoscaler - `2.15.1-6`

## Setup Steps

```bash
# Apply config to allow Prometheus
oc apply -f 0-cluster-monitoring-config.yml

# Create the demo project
oc new-project demo

# Create the KEDA controller
oc apply -f 1-cma.yml

# Create the Kafka deployment
oc apply -f 2-kafka.yml

# Enable Knative Serving and Eventing
oc apply -f 3-knative.yml

# Create the Kafka topic
oc apply -f 4-kafkatopic.yml

## Deploy the Listener Application
oc new-app https://github.com/bfarr-rh/kafka-openshift-python-listener.git \
  -e KAFKA_BROKERS=my-cluster-kafka-bootstrap.demo.svc:9092 \
  -e KAFKA_TOPIC=my-topic \
  -e CONSUMER_GROUP=my-kafka-consumer-group \
  -e DELAY=10 \
  --name=listener

## Create the Scaled Object
oc apply -f 5-scaled-object-kafka-listener.yml

## Configure Prometheus Access
# Create service account and apply roles
oc create serviceaccount thanos 
oc describe serviceaccount thanos 
oc apply -f 6-prometheus-roles.yml

# Retrieve the token
oc get secret my-sa-token -o jsonpath='{.data.token}' | base64 -d


## Demonstration: Send Messages to Kafka
# Access the Kafka broker pod
oc rsh my-cluster-broker-0 

# Send messages
bin/kafka-console-producer.sh --topic my-topic --bootstrap-server my-cluster-kafka-bootstrap.demo.svc:9092 --property key.separator=:

# Describe consumer group
bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-kafka-consumer-group

## Demonstrate Knative Integration
# Update a Knative service
kn service update openshift-fastapi --env verson=v2

## Notes
#You can pause the autoscaling of a scaled object by adding the autoscaling.keda.sh/paused-replicas annotation. This scales the replicas to the specified value and pauses autoscaling until the annotation is removed.

## References

# https://github.com/zroubalik/keda-openshift-examples/blob/main/prometheus/ocp-monitoring/triggerauthentication.yaml
# https://developers.redhat.com/articles/2025/05/05/practical-example-custom-metrics-autoscaler?source=sso#overview_of_installation_and_configuration

## Demonstration: Prometheus

oc apply -f 7-deploymentforprom.yml
oc apply -f 8-sc-test-app.yml

