---
title: Db2
weight: 6
bookHidden: false
---

# Source Db2

The extracted `replicant-cli` will be referred to as the `$REPLICANT_HOME` directory. The user which `$REPLICANT_HOME` is in will be referred to as the `replicant-user`.

## I. Grant Db2 Permissions

1. Grant `replicant-user` read access to all the databases, schemas, and table you are going to replicate

2. Grant `replicant-user` read access to the following system tables/views:
    ```
    SYSIBM.SYSTABLES
    SYSIBM.SQLTABLETYPES
    SYSIBM.SYSCOLUMNS
    SYSIBM.SYSTABCONST
    SYSIBM.SQLCOLUMNS
    SYSCAT.COLUMNS (needed for fetch-schemas mode)
    ```

3. Grant `replicant-user` execute permissions on the following system procs:
    ```
    SYSIBM.SQLTABLES
    SYSIBM.SQLCOLUMNS
    SYSIBM.SQLPRIMARYKEYS d. SYSIBM.SQLSTATISTICS
    ```
**Note**: These permissions only need to be granted once at the start of fresh a replication job.

Steps II, III, IV, and V are for enabling CDC replication

## II. Install MQ

1. Download the latest MQ server package (IBM_MQ_9.1.3_LINUX_X86-64.tar.gz) from IBM passport advantage

2. Untar the the package:
    ```BASH
    $tar -xvf IBM_MQ_9.0.0.0_LINUX_X86-64.tar.gz
    ```
3. Navigate to the extracted MQServer directory:
    ```BASH
    $cd MQServer
    ```
4. Accept the MQ license:
    ```BASH
    $sudo ./mqlicense.sh
    ```
5. Install all the rpm files:
    ```BASH
    $sudo alien -i MQSeriesRuntime-*.rpm MQSeriesServer-*.rpm --scripts
    ```

## III. Apply the Q Replication License

1. Download the activation kit (IS_DataRep_11_4_Activtn_ProdDoc.zip) from IBM passport advantage

2. Unzip the archive and navigate into the extracted directory:
    ```BASH
    unzip IS_DataRep_11_4_Activtn_ProdDoc.zip
    cd CC4MTML
    ```

3. Untar the file from archive and navigate to the file
    ```BASH
    $tar xvf IS_DataRep_11.4_Activtn_ProdDoc.tar.gz
    cd IS_DataRep_11.4_Activtn
    ```
4. Navigate to the appropriate directory file containing license depending on the db2 version installed:
    ```BASH
    $cd License/Db2_11.5
    ```

5. Apply the license file:
    ```BASH
    $db2licm -a iidr.lic
    ```

## IV. Setup Queue Replication

In the following steps, the user in which your DB2 instance unix exists will be referred to as dbuser and will be used to connect to Db2. The mqm user will be used to create/configure the Queue Manager.

1. Enable log retention on your source DB2 instance:
    ```
    update db cfg for sourcedb using LOGARCHMETH1 LOGRETAIN
    ```

2. Backup you source database (SOURCEDB)
    ```
    backup db sourcedb
    ```

3. Grant dbadm access to the mqm user:
    ```
    grant dbadm on database to user mqm
    ```
**The steps below are to create/configure the MQ queue manager (Run as mqm user).**

4. Run the following commands in the order presented:
    ```BASH
    cd/opt/mqm/bin
    ./crtmqmCDC_QM
    ./strmqmCDC_QM
    ./runmqscCDC_QM

    ALTER QMGR MAXMSGL(104857600) //Alter the queue manager to have the recommended starting value for maxmsgl
    ALTER QMGR MAXUMSGS(999999999)
    DEFINE QLOCAL('CDC_RESTART_Q') put(enabled) get(enabled)
    DEFINE QLOCAL('CDC_ADMIN_Q') PUT(ENABLED)
    GET(ENABLED)
    DEFINE QLOCAL('CDC_LOG_Q') USAGE(NORMAL)
    MAXDEPTH(999999999) MAXMSGL(104857600)
    DEFINE CHANNEL('CDC_Q_CHANNEL') CHLTYPE(SVRCONN)
    TRPTYPE(TCP)
    START CHL('CDC_Q_CHANNEL')
    DEFINE LISTENER('REPL_LSTR') TRPTYPE(TCP) PORT(1450)
    CONTROL(QMGR)
    START LISTENER('REPL_LSTR')
    ALTER QMGR CHLAUTH(DISABLED)
    ```
    The following three commands are to allow disabling channel authentication. They are optional.

    ```BASH
    ALTER AUTHINFO(SYSTEM.DEFAULT.AUTHINFO.IDPWOS)
    AUTHTYPE(IDPWOS) CHCKCLNT(OPTIONAL)
    REFRESH SECURITY TYPE(CONNAUTH)
    ```
5. Run following commands on the asnclp prompt to configure Queuereplication:
    ```BASH
    ASNCLP SESSION SET TO Q REPLICATION
    SET SERVER CAPTURE TO DB SOURCEDB
    SET CAPTURE SCHEMA SOURCE ASN
    SET QMANAGER CDC_QM FOR CAPTURE SCHEMA  
    SET RUN SCRIPT NOW STOP ON SQL ERROR ON
    CREATE CONTROL TABLES FOR CAPTURE SERVER USING RESTARTQ "CDC_RESTART_Q" ADMINQ "CDC_ADMIN_Q"
    CREATE PUBQMAP MyPubQMap USING SENDQ "CDC_LOG_Q" MESSAGE FORMAT XML MESSAGE CONTENT TYPE R HEARTBEAT INTERVAL 0 MAX MESSAGE SIZE 101376
    #Note : The MESSAGE CONTENT TYPE can be set to either R (ROW TYPE) or T (TRANSACTION TYPE).
    CREATE PUB USING PUBQMAP MyPubQMap( PUBNAME CUSTOMER-UNI DBUSER.CUSTOMER ALL CHANGED ROWS Y BEFORE VALUES Y CHANGED COLS ONLY N HAS LOAD PHASE N SUPPRESS DELETES N)
    START XML PUB PUBNAME CUSTOMER-UNI
    ```
6. Set the following environment variables to Start Q Capture program:
    ```BASH
    export LD_LIBRARY_PATH=/opt/mqm/lib64:$LD_LIBRARY_PATH export PATH=/home/dbuser/sqllib/bin:$PATH
    export DB2INSTANCE="dbuser"
    export DB2LIB="/home/dbuser/sqllib/lib"
    export DB2_HOME="/home/dbuser/sqllib"
    export IBM_DB_DIR="/home/dbuser/sqllib"
    export IBM_DB_HOME="/home/dbuser/sqllib"
    export IBM_DB_INCLUDE="/home/dbuser/sqllib/include" export IBM_DB_LIB="/home/dbuser/sqllib/lib"
    ```
7. Run the capture program:
    ```BASH
    asnqcap capture_server=sourcedb capture_schema="ASN" logstdout=y
    ```

**Note: Make sure that the asnqcap program is running throughout the replication process.**

## V. Setup Connection Configuration

1. From ```$REPLICANT_HOME```, navigate to the connection configuration file:
    ```BASH
    vi conf/conn/db2.yaml
    ```
2. Make the necessary changes as follows:
   ```YAML
   type: DB2

   database: tpch
   host: localhost #Replace localhost with your db2 server host
   port: 50002 #Replace 50002 with the port number to connect to your db2 server

   username: replicant #Replace replicant with your username that connects to your db2 server
   password: "Replicant#123" #Replace Replicant#123 with your user's password

   max-connections: 30 #Maximum number of connections replicant can open in db2

   cdc-log-config: MQ #cdc-log-storage: MQ #Allowed values are MQ, KAFKA_STRING, KAFKA_AVRO.
     mqueue-connections:
       queue-conn1:
         host: localhost
         port: 1450
         queue-manager: CDC_QM
         queue-channel: CDC_Q_CHANNEL
         username: queue_manager
         password: queue_manager
         queues:
         - name: CDC_LOG_Q
         #message-format: XML #Allowed values are XML, DELIMITED.
         #message-type: ROW #Allowed values are ROW, TRANSACTION. Valid only for message-format XML.
         #lob-send-option: INLINE #Allowed values are INLINE, SEPARATE.
         #  backup-mqueue-connection:
         #    host: localhost
         #    port: 1460
         #    queue-manager: CDC_QM_BACKUP
         #    queue-channel: CDC_Q_BACKUP_CHANNEL
         #    username: backup_queue_manager
         #    password: backup_queue_manager
         #    queues:
         #    - name: CDC_LOG_BACKUP_Q
         #    ssl:
         #      trust-store:
         #        path: "/path/to/trust/store"
         #        password: 'changeit'
         #      key-store:  #Provide keystore details only in case of two-way SSL authentication i.e client authentication
         #        path: "/path/to/key/store"
         #        password: 'changeit'
         #      ssl-cipher-suite: 'TLS_RSA_WITH_AES_128_CBC_SHA256' #Provide your encryption algorithm based what is configured on MQ Manager

         #
         #- name: CDC_LOG_Q_DELIMITED
         #  message-format: DELIMITED #Allowed values are XML, DELIMITED.
         #ssl:
         #  trust-store:
         #    path: "/path/to/trust/store"
         #    password: 'changeit'
         #  key-store:  #Provide keystore details only in case of two-way SSL authentication i.e client authentication
         #    path: "/path/to/key/store"
         #    password: 'changeit'
         #  ssl-cipher-suite: 'TLS_RSA_WITH_AES_128_CBC_SHA256' #Provide your encryption algorithm based what is configured on MQ Manager

     #cdc-log-storage: KAFKA_STRING #Allowed values are MQ, KAFKA_STRING, KAFKA_AVRO.
     #kafka-connection:
     #  cdc-log-topic: cdc_log_topic
     #  message-format: XML
     #  message-type: ROW #Allowed values are ROW, TRANSACTION. Valid only for message-format XML.
     #  lob-send-option: INLINE
     #  connection:
     #    brokers:
     #      broker1:
     #        host: localhost
     #        port: 19092
     #
     #
     #cdc-log-storage: KAFKA_AVRO #Allowed values are MQ, KAFKA_STRING, KAFKA_AVRO.
     #kafka-connection:
     #  cdc-log-topic-prefix: "" #Common prefix for all Kafka IIDR topics.
     #  cdc-log-topic-prefix-list: #List of mapping from common prefixes to source tables.
     #  - cdc-log-topic-prefix: "" #Common prefix for all Kafka IIDR topics.
     #    tables: [table1, table2]
     #  - cdc-log-topic-prefix: "" #Common prefix for all Kafka IIDR topics.
     #    tables: [table3, table4]

     #  message-format: KCOP_MULTIROW_AUDIT
     #  connection:
     #    brokers:
     #      broker1:
     #        host: localhost
     #        port: 19092
     #    schema-registry-url: "http://localhost:8081"
   ```

## VI Setup Filter Configuration

1. From `$REPLICANT_HOME`, navigate to the filter configuration file:
    ```BASH
    vi filter/db2_filter.yaml
    ```
2. In accordance to you replication needs, specify the data which is to be replicated. Use the format of the example explained below:  

    ```yaml
    allow:
      #In this example, data of object type Table in the catalog tpch  and schema db2user will be replicated
      catalog: tpch
      schema: db2user
      types: [TABLE]

      #From catalog tpch, only the NATION, ORDERS, and PART tables will be replicated.
      #Note: Unless specified, all tables in the catalog will be replicated
      allow:
        NATION:
           #Within NATION, only the US and AUS columns will be replicated
           allow: ["US, AUS"]

        ORDERS:  
           #Within ORDERS, only the product and service columns will be replicated as long as they meet the condition o_orderkey < 5000
           allow: ["product", "service"]
           conditions: "o_orderkey < 5000"

        PART: #All columns in the table PART will be replicated without any predicates
    ```
    The following is a template of the format you must follow:

    ```YAML
    allow:
      catalog: <your_catalog_name>
      schema: <your_schema_name>
      types: <your_object_type>


      allow:
        <your_table_name>:
           allow: ["your_column_name"]
           conditions: "your_condition"

        <your_table_name>:  
           allow: ["your_column_name"]
           conditions: "your_condition"

        <your_table_name>:
          allow: "your_column_name"]
          conditions: "your_condition"         
    ```

## VII Setup Extractor Configuration

For real-time replication, you must create a heartbeat table in the source Db2.

1. Create a heartbeat table in the schema you are going to replicate with the following DDL:
   ```SQL
   CREATE TABLE "<user_database>"."<schema>"."replicate_io_cdc_heartbeat"(
     "timestamp" BIGINT NOT NULL,
     PRIMARY KEY("timestamp"));
   ```

2. Grant ```INSERT```, ```UPDATE```, and ```DELETE``` privileges to the user configured for replication

3. From `$REPLICANT_HOME`, navigate to the extractor configuration file:
   ```BASH
   vi conf/src/db2.yaml
   ```
4. Under the Realtime Section, make the necessary changes as follows:
    ```YAML
    realtime:
      heartbeat:
        enable: true
        catalog: tpch #Replace tpch with your catalog name
        schema-name: db2user #Replace db2user with the name of the schema your heartbeat table is in
        table-name [20.09.14.3]: replicate_io_cdc_heartbeat #Replace replicate_io_cdc_heartbeat with your heartbeat table's name if applicable
        column-name [20.10.07.9]: timestamp #Replace timestamp with your heartbeat table's column name if applicable
    ```
5. Below is a sample extractor file with commonly used parameters:
   ```YAML
   snapshot:
     threads: 16
     fetch-size-rows: 5_000

     _traceDBTasks: true
     #fetch-schemas-from-system-tables: true

     #per-table-config:
     #- catalog: tpch
       #schema: db2user
       #tables:
         #lineitem:
           #row-identifier-key: [l_orderkey, l_linenumber]

   realtime:
     threads: 1
     fetch-size-rows: 10000
     _traceDBTasks: true
     fetch-interval-s: 0
     #replicate-empty-string-as-null: true #This config is relevant for Delimited message type only.

     heartbeat:
       enable: true
       catalog: tpch
       schema: db2user
       #table-name: replicate_io_cdc_heartbeat
       #column-name: timestamp
       interval-ms: 10000
   ```
