# IoT, Kafka & k8s: Midiendo la temperatura a lo grande

## Setup

### Create weavescope

```bash
kubectl create clusterrolebinding "cluster-admin-$(whoami)" --clusterrole=cluster-admin --user="$(gcloud config get-value core/account)"
kubectl apply -f "https://cloud.weave.works/k8s/scope.yaml?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

```bash
WEAVE_POD=$(kubectl get pod -n weave | grep app | awk '{printf("%s", $1)}')
kubectl port-forward -n weave $WEAVE_POD 4040
```

### Create zookeeper, kafka y kafka-rest

```bash
kubectl apply -f zookeeper.yml
kubectl apply -f kafka.yml
kubectl apply -f kafka-http.yml
```

### Configure data topics

```bash
kubectl exec -it kafka-0 bash
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic agg-metrics
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic data
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic rules
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic alerts
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic control
/opt/kafka/bin/kafka-topics.sh --create --partitions 60 --zookeeper zk-cs:2181 --replication-factor 3 --topic sensor-metadata
```

### Create Kafka Streams engine

```bash
kubectl apply -f iot-engine.yml
```

## Working

## Configure IP


```bash
export HTTP_KAFKA_REST_IP=<IP>
export HTTP_IOT_ENGINE=<IP>
```

### Sending data

```bash
docker run -it -e HTTP_SERVER=$HTTP_KAFKA_REST_IP:8082 -e HTTP_TOPIC=data -e HTTP_INTERVAL_MS=10 andresgomezfrr/data-simulator:3.0
```

## Query timeseries data

```bash
ID=1111
START=2021-04-06T00:00:00.00Z
END=2021-04-10T00:00:00.00Z
curl http://$HTTP_IOT_ENGINE:5574/iot-engine/query/metrics/$ID/$START/$END
```

### Create rule to sensor '1111'

* humidity < 50
* temperature > 22

```bash
curl http://$HTTP_IOT_ENGINE:5574/iot-engine/query/rules -d '{"id":"1111","rules":[{"ruleName":"max_temperature","metricName":"temperature","metricValue":22,"condition":">"},{"ruleName":"min_humidity","metricName":"humidity","metricValue":50,"condition":"<"}]}' -H "Content-type: application/json"
```

### Create alert streams

```bash
curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" -H "Accept: application/vnd.kafka.v2+json" \
    --data '{"name": "alert_consumer_instance", "format": "json", "auto.offset.reset": "latest"}' \
    http://$HTTP_KAFKA_REST_IP:8082/consumers/alert_stream

curl -X POST -H "Content-Type: application/vnd.kafka.v2+json" --data '{"topics":["alerts"]}' \
    http://$HTTP_KAFKA_REST_IP:8082/consumers/alert_stream/instances/alert_consumer_instance/subscription

curl -X GET -H "Accept: application/vnd.kafka.json.v2+json" \
    http://$HTTP_KAFKA_REST_IP:8082/consumers/alert_stream/instances/alert_consumer_instance/records

curl -X DELETE -H "Accept: application/vnd.kafka.v2+json" \
          http://$HTTP_KAFKA_REST_IP:8082/consumers/alert_stream/instances/alert_consumer_instance

```

## Links

* [Google Cloud Kubernetes](https://cloud.google.com/kubernetes-engine)
* [Weavescope](https://www.weave.works/oss/scope/)
* [Kafka](https://kafka.apache.org/)
* [Zookeeper](https://zookeeper.apache.org/)
* [Kafka-Rest](https://github.com/confluentinc/kafka-rest)
* [IoT-Engine](https://github.com/andresgomezfrr/iot-engine)
* [Data Simulator](https://hub.docker.com/r/andresgomezfrr/data-simulator)