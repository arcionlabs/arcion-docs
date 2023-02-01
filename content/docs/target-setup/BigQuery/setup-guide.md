---
pageTitle: Set up Google BigQuery as target
title: Setup guide
description: "Set up BigQuery as target, supporting realtime replication using load job method and Storage Write API."
weight: 1
bookHidden: false
---

# Setup guide for Google BigQuery target

The extracted `replicant-cli` will be referred to as the `$REPLICANT_HOME` directory in the following steps.

## I. Obtain the JDBC driver for Google BigQuery

Replicant requires the JDBC driver for Google BigQuery as a dependency. To obtain the appropriate driver, follow these steps: 

- Go to the [JDBC drivers for BigQuery page](https://cloud.google.com/bigquery/docs/reference/odbc-jdbc-drivers#current_jdbc_driver).
- From there, download the [latest JDBC 4.2-compatible JDBC driver ZIP](https://storage.googleapis.com/simba-bq-release/jdbc/SimbaJDBCDriverforGoogleBigQuery42_1.2.25.1029.zip).
- From the downloaded ZIP, locate and extract the `GoogleBigQueryJDBC42.jar` file.
- Put the `GoogleBigQueryJDBC42.jar` file inside `$REPLICANT_HOME/lib` directory.

## II. Required permissions
To load data into BigQuery, you need to make sure you have the necessary IAM permissions. These permissions are required to run a load job and load data into BigQuery tables and partitions. For more information about these permissions, see [Permissions to load data into BigQuery](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv#required_permissions).

## III. Set up connection configuration

1. From `$REPLICANT_HOME`, navigate to the sample connection configuration file:
    ```BASH
    vi conf/conn/bigquery.yaml
    ```

2. If you store your connection credentials in AWS Secrets Manager, you can tell Replicant to retrieve them. For more information, see [Retrieve credentials from AWS Secrets Manager](/docs/references/secrets-manager). 
    
    Otherwise, you can put your credentials like usernames and passwords in plain form like the sample below:
    ```YAML
    type: BIGQUERY

    host: https://www.googleapis.com/bigquery/v2
    port: 443
    project-id: bigquerytest-268904
    auth-type: 0
    o-auth-service-acc-email: bigquerytest@bigquerytest-268904.iam.gserviceaccount.com
    o-auth-pvt-key-path: <path_to_oauth_private_key>
    location: US
    timeout: 500

    username: "xxx"
    password: "xxxx"

    max-connections: 20

    max-retries: 10
    retry-wait-duration-ms: 1000
    ```

## IV. Set up Applier configuration
To configure replication according to your requirements, specify your configuration in the Applier configuration file. You can find a sample Applier configuration file `bigquery.yaml` in the `$REPLICANT_HOME/conf/dst` directory.

Arcion Replicant supports the following modes of replication for BigQuery target:

- `snapshot`
- `realtime`
- `full`

For more information about different Replicant modes, see [Running Replicant]({{< ref "running-replicant" >}}).

For a detailed explanation of configuration parameters in the Applier file, see [Applier Reference]({{< ref "/docs/references/applier-reference" >}}).

### Configure `snapshot` replication
The following is a sample configuration for operating in `snapshot` mode:

```YAML
snapshot:
  threads: 16

  batch-size-rows: 100_000_000
  txn-size-rows: 1_000_000_000

  bulk-load:
    enable: true
    type: FILE
    save-file-on-error: true
    serialize: true

  use-quoted-identifiers: false
```

For more information about the configuration parameters for snapshot mode, see [Snapshot Mode]({{< ref "/docs/references/applier-reference#snapshot-mode" >}}).

### Configure `realtime` replication
For [`realtime`]({{< ref "docs/running-replicant#replicant-realtime-mode" >}}) replication, Replicant supports the following two methods:

- [Loading data with conventional load job method](#load-data-with-load-job-method).
- [Streaming data using BigQuery Storage Write API](#load-data-using-the-storage-write-api).

{{< hint "info" >}}
**Tip:** When choosing between the load job method and the Storage Write API, consider the following: 

- Load job method is comparatively slower. In some cases, it can be two to three times slower. But it offers more reliability.
- Storage Write API offers faster data loading and streaming, but is less reliable.
{{< /hint >}}


#### Load data with load job method
This method uses [the conventional load job method](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv) to load data into BigQuery partitions and tables. This is the default method and configurable using different [Applier configuration parameters]({{ ref "docs/references/applier-reference" >}}).

The following is a sample configuration that uses load job method to load data in realtime:

```YAML
realtime:
  threads: 16
    _traceDBTasks: true
    skip-tables-on-failures : false
    replay-strategy: IN_MEMORY_MERGE
    per-table-config:
      - catalog: tpch_scale_0_01
        tables:
          nation:
            replay-strategy: MERGE
            enable-dependency-tracking: false
          region:
            replay-strategy: INSERT_MERGE
            enable-dependency-tracking: true
```

#### Load data using the Storage Write API
This method uses the [BigQuery Storage Write API](https://cloud.google.com/bigquery/docs/write-api-streaming) to stream data into BigQuery. You can choose this method by setting the `use-write-storage-api` parameter to `true` under the `realtime` sectin of the Applier configuration file.

The following is a sample configuration that uses the Storatge Write API to load data in realtime:

```YAML
realtime:
  threads: 16
  _traceDBTasks: true
  skip-tables-on-failures : false

  use-write-storage-api: true

  replay-strategy: IN_MEMORY_MERGE
  per-table-config:
    - catalog: tpch_scale_0_01
      tables:
        nation:
          replay-strategy: MERGE
          enable-dependency-tracking: false
        region:
          replay-strategy: INSERT_MERGE
          enable-dependency-tracking: true
```

#### Replay strategies
Replay strategies are how Arcion implements CDC changes and applies them in real-time to the target. For more information about replay strategies for BigQuery target, see [Replay strategies]({{< relref "replay-strategies" >}}).
