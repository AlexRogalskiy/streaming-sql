# Streaming SQL

This repo has example Kubernetes deployment files and example applications for various streaming SQL options.

## Kafka

All the examples in this repo assume you have an Apache Kafka cluster running and accessible on you Kubernetes cluster. [Strimzi](https://strimzi.io/) is a great way to get Kafka up and running on Kubernetes.

Install the latest version of the Strimzi Kafka Cluster Operator on your cluster with the following command (see [the docs](https://strimzi.io/docs/latest/#downloads-str) for other deployment options):

```bash
curl -L http://strimzi.io/install/latest \
    | sed 's/namespace: .*/namespace: <namespace>/' \
    | kubectl apply -f - -n <namespace>
```

Then you can spin up a 3 node Kafka cluster with the following command (see the [Strimzi project repository](https://github.com/strimzi/strimzi-kafka-operator/tree/master/examples/kafka) for other example Kafka deployments):

```bash
kubectl apply -f \
    https://strimzi.io/examples/latest/kafka/kafka-persistent.yaml \
    -n <namespace>
```

## Stream Generator

In order to provide an example streaming source, for the stream SQL implementations in this repo, an example stream generator deployment is provided. This will stream the [Wikipedia](https://wikipedia.org) changes log into a Kafka topic (removing any non-change messages and errors). This stream source can be deployed using the following command:

```bash
kubectl apply -f stream-generator/generator-deployment.yaml
```

By default this will stream messages to the broker bootstrap address for the Strimzi cluster described in the section above. If you are using a different set up the change the `KAFKA_BOOTSTRAP_SERVERS` environment variable in the deployment file.
The generator will stream changes into the `wiki-changes` topic on the configured Kafka broker. If you do not have topic auto-creation enabled, you should create that topic first. If you are using the Strimzi deployment above, which has the Topic Operator enabled, the topic can be created using the command below:

```bash
kubectl apply -f stream-generator/kafka-topic.yaml
```

## ksqlDB

[ksqlDB](https://ksqldb.io/) is a streaming SQL implementation based on [Apache Kafka](https://kafka.apache.org/) and the [Kafka Stream](https://kafka.apache.org/documentation/streams/) library.

It is not a fully open source solution as it is licensed under the Confluent Community License Agreement v1.0.

To test out ksqlDB, you need to change the `KSQL_BOOTSTRAP_SERVERS` environment variable in the `ksqlDB-deployment.yaml` to match the boostrap address of you Kafka cluster. If you are using Strimzi with the example above then this should be set as below (if you changed the `metadata.name` field of the `kafka` custom resource then change `my-cluster` to match the new name):

```yaml
- name: KSQL_BOOTSTRAP_SERVERS
  value: PLAINTEXT://my-cluster-kafka-bootstrap:9092
```

Then you can deploy your ksqlDB instance:

```bash
$ kubectl apply -f ksqlDB/ksqlDB-deployment.yaml
```

Once deployed you can interact with the server using the `ksql-cli` command line client running in another pod using the `ksql-cli.sh` script:

```bash
$ ./ksqlDB/ksql-cli.sh <namespace> 1
```

Where the first argument is the namespace the ksqlDB instance is deployed in and the second argument is a number used to label different invocations of the CLI if you want to run more than one instance for data entry and analysis.

Now you can play with ksqlDB and follow the [project quickstart guide](https://ksqldb.io/quickstart.html). 

Alternatively, you can write queries against the Wikipedia changes stream. For example, if you want to create a stream which contains the user IDs and the title of the Wikipedia article they are editing you can use the command below:

```sql
CREATE STREAM userTitles (user VARCHAR, title VARCHAR) WITH (kafka_topic='wiki-changes', key='user', value_format='json');
```

You can then see the contents of that stream by using the query below:

```sql
SELECT * FROM userTitles EMIT CHANGES;
```

You create tables from this stream and others and then query these like tables in a database.

## materialize

[Materialize](https://materialize.io/) uses a streaming engine based on timely dataflow to provide a PostgreSQL compatible SQL interface. 

This is not a fully open source solution as it is licensed under the Business Source License v1.1. Which limits you to running the code on a singe instance unless you purchase a licence.

You can deploy the materialize server using the following command:

```bash
$ kubectl apply -f materialize/materialize-deployment.yaml
```

Once deployed you can interact with the server using the `psql` command line client running in another pod using the `psql-cli.sh` script:

```bash
$ ./materialize/psql-cli.sh <namespace> 1
```

Where the first argument is the namespace the ksqlDB instance is deployed in and the second argument is a number used to label different invocations of the CLI if you want to run more than one instance for data entry and analysis.

Now you can play with materialize and follow the [project quickstart guide](https://materialize.io/docs/get-started/).
