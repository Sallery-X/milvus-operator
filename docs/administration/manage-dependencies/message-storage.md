# Configure Message Storage with Milvus Operator
Milvus uses `RocksMQ`, `Pulsar` or `Kafka` for managing logs of recent changes, outputting stream logs, and providing log subscriptions. This topic introduces how to configure message storage dependencies when you install Milvus with Milvus Operator.

This topic assumes that you have deployed Milvus Operator.

> See [Deploy Milvus Operator](../../installation/installation.md) for more information.

You need to specify a configuration file for using Milvus Operator to start a Milvus.

```shell
kubectl apply -f https://raw.githubusercontent.com/zilliztech/milvus-operator/main/config/samples/demo.yaml
```

You only need to edit the code template in `demo.yaml` to configure third-party dependencies. The following sections introduce how to configure etcd.

# Before you begin
The table below shows whether RocksMQ, Pulsar, and Kafka are supported in Milvus standalone and cluster mode.

|                 | RocksMQ | Pulsar | Kafka |
|:---------------:|:-------:|:------:|:-----:|
| Standalone mode |    ✔️    |    ✔️   |   ✔️   |
|   Cluster mode  |    ✖️    |    ✔️   |   ✔️   |

There are also other limitations for specifying the message storage:
- Only one message storage for one Milvus instance is supported. However we still have backward compatibility with multiple message storages set for one instance. The priority is as follows:
  - standalone mode:  RocksMQ (default) > Pulsar > Kafka
  - cluster mode: Pulsar (default) > Kafka
- The message storage cannot be changed while the Milvus system is running. 
-  Only Kafka 2.x or 3.x verison is supported.

## Configure RocksMQ
RocksMQ is the default message storage in Milvus standalone. 

> Currently, you can only configure RocksMQ as the message storage for Milvus in standalone mode.

#### Example 

The following example configures a RocksMQ service. 

```YAML
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: milvus
spec:
  # Omit other fields ...
  dependencies:
    # Omit other fields ...
    msgStreamType: rocksmq
    rocksmq:
      persistence:
        enabled: true
        pvcDeletion: true
```

The fields under `rocksmq.persistence` adds an extra PVC to persist the RocksMQ data. `pvcDeletion` determines whether the persisted data will be deleted when the Milvus instance is deleted.

## Configure Pulsar

Pulsar manages logs of recent changes, outputs stream logs, and provides log subscriptions. Configuring Pulsar for message storage is supported in both Milvus standalone and Milvus cluster. However, with Milvus Operator, you can only configure Pulsar as message storage for Milvus cluster. Add required fields under `spec.dependencies.pulsar` to configure Pulsar.

`pulsar` supports `external` and `inCluster`.

### External Pulsar

`external` indicates using an external Pulsar service. 
Fields used to configure an external Pulsar service include:

- `external`:  A `true` value indicates that Milvus uses an external Pulsar service.
- `endpoints`: The endpoints of Pulsar.

####  Example

The following example configures an external Pulsar service.

```YAML
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: my-release
  labels:
    app: milvus
spec:
  dependencies: # Optional
    pulsar: # Optional

      # Whether (=true) to use an existed external pulsar as specified in the field endpoints or 
      # (=false) create a new pulsar inside the same kubernetes cluster for milvus.
      external: true # Optional default=false
      # The external pulsar endpoints if external=true
      endpoints:
      - 192.168.1.1:6650
  components: {}
  config: {}           
```

### Internal Pulsar

`inCluster` indicates when a Milvus cluster starts, a Pulsar service starts automatically in the cluster.

#### Example 

The following example configures an internal Pulsar service in the minimum cost of resources.

```YAML
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: my-release
  labels:
    app: milvus
spec:
  # Omit other fields ...
  dependencies:
    # Omit other fields ...
    pulsar:
      inCluster:
        chartVersion: pulsar-v3
        deletionPolicy: Delete
        pvcDeletion: true
        values:
          volumes:
            persistence: false
          components:
            autorecovery: false
          zookeeper:
            replicaCount: 1
            podMonitor:
              enabled: false
            resources:
              requests:
                memory: 64Mi
                cpu: 0.001
            volumes:
              # use a persistent volume or emptyDir
              persistence: false
            configData:
              PULSAR_MEM: >
                -Xms64m -Xmx256m
          bookkeeper:
            component: bookie
            replicaCount: 1
            podMonitor:
              enabled: false
            resources:
              requests:
                memory: 64Mi
                cpu: 0.001
            volumes:
              # use a persistent volume or emptyDir
              persistence: false
            configData:
              PULSAR_MEM: >
                -Xms64m
                -Xmx4096m
                -XX:MaxDirectMemorySize=8192m
          autorecovery:
            replicaCount: 0
            podMonitor:
              enabled: false
          broker:
            replicaCount: 1
            podMonitor:
              enabled: false
            resources:
              requests:
                memory: 64Mi
                cpu: 0.001
            configData:
              PULSAR_MEM: >
                -Xms64m -Xmx4096m -XX:MaxDirectMemorySize=8192m
              managedLedgerDefaultEnsembleSize: '1'
              managedLedgerDefaultWriteQuorum: '1'
              managedLedgerDefaultAckQuorum: '1'
          proxy:
            replicaCount: 1
            autoscaling:
              enabled: false
            podMonitor:
              enabled: false
              interval: 30s
              scrapeTimeout: 10s
            resources:
              requests:
                memory: 64Mi
                cpu: 0.001
            configData:
              PULSAR_MEM: >
                -Xms64m -Xmx512m -XX:MaxDirectMemorySize=2048m  
```

> Find the complete configuration items to configure an internal Pulsar service in <a href="https://artifacthub.io/packages/helm/apache/pulsar/3.3.0?modal=values"> values.yaml</a>. Add configuration items as needed under `pulsar.inCluster.values` as shown in the preceding example.


## Configure Kafka

Pulsar is the default message storage in a Milvus cluster. If you want to use Kafka, add the optional field `msgStreamType` to configure Kafka.

`kafka` supports `external` and `inCluster`.

### External Kafka

`external` indicates using an external Kafka service. 

Fields used to configure an external Kafka service include:

- `external`: A `true` value indicates that Milvus uses an external Kafka service.
- `brokerList`: The list of brokers to send the messages to.

#### Example

The following example configures an external Kafka service.

```yaml
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: my-release
  labels:
    app: milvus
spec:
  config:
    kafka:
      # securityProtocol supports: PLAINTEXT, SSL, SASL_PLAINTEXT, SASL_SSL 
      securityProtocol: PLAINTEXT
      # saslMechanisms supports: PLAIN, SCRAM-SHA-256, SCRAM-SHA-512
      saslMechanisms: PLAIN
      saslUsername: ""
      saslPassword: ""
  # Omit other fields ...
  dependencies:
    # Omit other fields ...
    msgStreamType: "kafka"
    kafka:
      external: true
      brokerList: 
        - "kafkaBrokerAddr1:9092"
        - "kafkaBrokerAddr2:9092"
        # ...
```

### Internal Kafka

`inCluster` indicates when a Milvus cluster starts, a Kafka service starts automatically in the cluster.

#### Example

The following example configures an internal Kafka service.

```yaml
apiVersion: milvus.io/v1beta1
kind: Milvus
metadata:
  name: my-release
  labels:
    app: milvus
spec: 
  dependencies:
    msgStreamType: "kafka"
    kafka:
      inCluster: 
        values: {} # values can be found in https://github.com/bitnami/charts/blob/1fdd2283f0e5a8772e4a763b455733c77e01b119/bitnami/kafka/values.yaml
```

Find the complete configuration items to configure an internal Kafka service [here](https://github.com/bitnami/charts/blob/1fdd2283f0e5a8772e4a763b455733c77e01b119/bitnami/kafka/values.yaml). Add configuration items as needed under `kafka.inCluster.values`.
