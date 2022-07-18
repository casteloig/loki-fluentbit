# Commands
```
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add fluent https://fluent.github.io/helm-charts

helm upgrade --install grafana grafana/grafana -f manifests/grafana-values.yaml --version 6.32.1
```
## (Optional) Log generator
```
kubectl apply -f manifests/log-generator.yaml
```

## Install Loki
### Monolithic
```
helm upgrade --install loki grafana/loki -f manifests/loki-simple-values.yaml --version 2.12.2
```

### Scalable
You can't use scalable loki and use local filesystem as the block store (deployed with Helm because it does not allow to mofify the `accessMode: ReadWriteOnce`). https://github.com/grafana/helm-charts/issues/1111#issuecomment-1140933791
```
TBD
```

## Install Fluentbit
### Manifests
```
kubectl apply -f manifests/fluent-raw/
```

### Helm chart
```
helm upgrade --install fluent-bit fluent/fluent-bit -f manifests/fluentbit-values.yaml
```

# Configs
## Promtail
Promtail is by default configured to work in K8S. No further configuration needed.

## Fluentbit
The pipeline will go as follows:

1. `SERVICE` is the basic configuration of fluentbit.
2. `INPUT` tail plugin will search the logs that fluentbit is going to process. Important fields:
  * Tag: it adds a tag to the chunk that is being processed so the filters will know which chunks they have to filter.
  * Parser: indicates the parser that will be used in this pipeline.
  * Storage_type: if set to filesystem, it will store on filesystem the chunks that are not yet flushed (so they will not consume memory).
3. `PARSER` allows to convert from structured data to structured data. In this case, we will use it to extract labels from the log that the runtime (CRI-containerd or docker) is adding.
  * ?<XXXXX> are the name of the labels that will be parsed (the name that will have after parsing). 
4. `FILTER` allows to alter the data before delivering it to the output. In this case we are using a K8S environment so it is needed to add useful metadata information obtained from kubernetes.
  * Match: this field should match at least one Tag field in the `INPUT`.
5. `OUTPUT` these are the plugings that will deliver/forward the logs to another system.
  * Match: if all logs must be sent to X system, it must be set to `*`
  * Labels: indicates the labels that will be delivered to the output system (recognised specifically by the system). We will use these labels in loki to perform the queries. They can be set like follows:
    * `name=`is the name of the label that the output system will use once the logs are delivered to it.
    * `$var` is the name of the label contained in the logs.
    * `[val]` is the name of the value that must be used. If there are nested labels, the first one must be `$var` and the next ones `[var]`.


```
fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
        Health_Check  On
        storage.path  /var/log/fluent-bit/buffer
        storage.max_chunks_up 10
        storage.sync  full
        storage.backlog.mem_limit 5M

    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-loki.conf

  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Parser            docker
        Path              /var/log/containers/*.log
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     8MB
        Skip_Long_Lines   On
        Buffer_Chunk_Size 150KB
        Buffer_Max_Size   150KB
        Refresh_Interval  20
        Storage.type      filesystem

  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed



  output-loki.conf: |
    [OUTPUT]
        name                   loki
        match                  *
        labels                 namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],container=$kubernetes['container_name'],app=$kubernetes['labels']['k8s-app'] 
        Host                   ${FLUENT_LOKI_HOST}
        Port                   ${FLUENT_LOKI_PORT}
        auto_kubernetes_labels Off
        net.connect_timeout    10
        net.keepalive          on
        net.keepalive_idle_timeout 300
        net.keepalive_max_recycle 10000
```

You should use one parser or another depending on which runtime you are using.

```
[PARSER]
    Name        docker
    Format      json
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L
    Time_Keep   On
```

```
[PARSER]
    # http://rubular.com/r/tjUt3Awgg4
    Name cri
    Format regex 
    Regex ^(?<time>[^ ]+) (?<stream>stdout|stderr) (?<logtag>[^ ]*) (?<message>.*)$
    Time_Key    time
    Time_Format %Y-%m-%dT%H:%M:%S.%L%z
```


## Index and Chunk storing
Index should be stored using boltdb-shipper. 

Chunks can be stored in filesystem, Cassandra or a cloud system store (S3, GCS).

### Filesystem based chunk store
You can specify several different schemas based on the time logs are stored. For example you can use scheme v11 with filesystem object store from 01-05-2020 and use another scheme from 01-05-2021.

* `store` -> type of storage for indexes.
* `object_store` -> type of storage for chunks

`period` must be set to a multiple of 24h.
```
schema_config:
    configs:
    - from: 2020-05-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: loki_index_
        period: 24h
```

```
storage_config:
    boltdb_shipper:
      active_index_directory: /data/loki/boltdb-shipper-active
      cache_location: /data/loki/boltdb-shipper-cache
      cache_ttl: 24h         # Can be increased for faster performance over longer query periods, uses more disk space
      shared_store: filesystem
    filesystem:
      directory: /data/loki/chunks
```

## Retention
It depends on the loki version: 
1. Loki scalable: The object storages - like Amazon S3 and Google Cloud Storage - supported by Loki to store chunks, are not managed by the Table Manager. Chunks are stored in S3 (so retention is enabled by default). There must be a bucket policy to delete old chunks in S3.
2. Loki simple (monolithic): it can be archieved with two different components:
  * Table manager.
  * Compactor.

https://grafana.com/docs/loki/latest/operations/storage/table-manager/#retention
https://grafana.com/docs/loki/latest/operations/storage/retention/

In K8S environment, it must be created a PV and PVC.

### Retention with Compactor
Loki 2.4.0: the single binary no longer runs a table-manager. Retention can be archieved by including `-target=all,table-manager` (which is not recommended). The second and recommended solution is to use deletes via the compactor.

If you use the compactor to archieve retention, the table manager can't be used (it may lead to data corruption).

Steps to reproduce log retention with Compactor:

```
limits_config:
  retention_period: 744h  # Duration of retention (always multiple of 24h)
```

```
schema_config:
  configs:
  - from: YYYY-MM-DD
    index:
      period: 24h
```

```
compactor:
  working_directory: /data/loki/boltdb-shipper-compactor
  shared_store: filesystem
  compaction_interval: 10m    # Interval at which to re-run the compactor operation (or retention if enabled)
  retention_enabled: true
  retention_delete_delay: 2h  # Delay after which chunks will be fully deleted during retention
  retention_delete_worker_count: 150
```
```
persistence:
  enabled: true
  accessModes:
  - ReadWriteOnce
  size: 10Gi
  labels: {}
  annotations: {}
  # selector:
  #   matchLabels:
  #     app.kubernetes.io/name: loki
  # subPath: ""
  existingClaim: loki-pvc
  storageClassName: loki-pvc
```

Retention period is configured within the `limits_config`

There are two ways of setting retention policies (and per tenant):
* `retention_period` which is applied globally.
* `retention_stream` which is only applied to chunks matching the selector.

Example of "three" policies
```
limits_config:
  retention_period: 744h
  retention_stream:
  - selector: '{namespace="dev"}'
    priority: 1
    period: 24h
  per_tenant_override_config: /etc/overrides.yaml
```

overrides.yaml:
```
overrides:
    "29":
        retention_period: 168h
        retention_stream:
        - selector: '{namespace="prod"}'
          priority: 2
          period: 336h
        - selector: '{container="loki"}'
          priority: 1
          period: 72h
    "30":
        retention_stream:
        - selector: '{container="nginx"}'
          priority: 1
          period: 24h
```

### Persistence
We need to create a pv which above 10Gi and has same namespace with loki.

If using monolythic application, it can use ReadWriteOnce. If not, use ReadWriteMany.

```
persistence:
  enabled: true
  accessModes:
  - ReadWriteOnce
  size: 10Gi
```

### Scalable

* Replication_factor = number of ingesters to write to and read from
```
commonConfig:
  path_prefix: /var/loki
  replication_factor: 2
```





## Loki configuration

```yaml
auth_enabled: false
    common:
      path_prefix: /var/loki
      replication_factor: 2
      storage:
        s3:
          bucketnames: optare-loki-test
          insecure: false
          region: eu-west-1
          s3: s3://optare-loki-test
          s3forcepathstyle: true
          http_config:
            response_header_timeout: 120s
    ingester:
      max_chunk_age: 1hm
      chunk_idle_period: 30m
      chunk_block_size: 262144
      chunk_retain_period: 1hm
    limits_config:
      enforce_metric_name: false
      max_cache_freshness_per_query: 1m
      reject_old_samples: true
      reject_old_samples_max_age: 168h
      split_queries_by_interval: 10m
    memberlist:
      join_members:
      - loki-memberlist
    querier:
      query_timeout: 2m
      engine:
        max_look_back_period: 3m
    query_range:
      align_queries_with_step: true
    ruler:
      storage:
        s3:
          bucketnames: optare-loki-test
    schema_config:
      configs:
      - from: "2022-01-01"
        index:
          period: 24h
          prefix: loki_index_
        object_store: s3
        schema: v12
        store: boltdb-shipper
    server:
      grpc_listen_port: 9095
      http_listen_port: 3100
      http_server_read_timeout: 600s
      http_server_write_timeout: 600s
    storage_config:
      boltdb_shipper:
        active_index_directory: /var/loki/data/loki/boltdb-shipper-active
        cache_location: /var/loki/data/loki/boltdb-shipper-cache
        cache_ttl: 24h
        shared_store: s3
```



## Situation
1. Fluent-bit: standby caused by malfunction with Loki. It cannot reconnect to Loki if it loses his connection.
2. Loki: 