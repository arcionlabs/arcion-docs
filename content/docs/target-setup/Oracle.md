---
title: Oracle
---
# Destination: Oracle

The extracted `replicant-cli` will be referred to as the `$REPLICANT_HOME` directory.

## I. Setup Shared Directory

1. Create a directory shared between Replicant host and Oracle host with ```READ``` and ```WRITE``` access
2. One way to create the shared directory is using NFS. You can follow NFS recommendation:

From here onwards, the directory created in step one will be considered to have the following path: ```/data/shared_fs```.

## II. Setup Oracle User Permissions
The following step must be executed in an Oracle client.

1. Grant the following privileges to the host replicant user:
   ```SQL
    GRANT CREATE TABLE TO <USERNAME>;
    --If you are unable to provide the permission above, you must manually create all the tables

    GRANT CREATE ANY DIRECTORY TO <USERNAME>;
    --If you are unable to provide the permission above, you must manually create the following directories:
    CREATE OR REPLACE DIRECTORY csv_data_dir AS '/data/shared_fs';
    CREATE OR REPLACE DIRECTORY csv_log_dir AS '/data/shared_fs';


    GRANT ALTER TABLE TO <USERNAME>;
    --
    ```
2. Manually create user schema and a schema named io_replicate. Grant both of them permission to access a tablespace

## III. Setup Connection Configuration

1. From `$REPLICANT_HOME`, navigate to the sample connection configuration file:
    ```BASH
    vi conf/conn/oracle_dst.yaml
    ```

2. Make the necessary changes as follows:
      ```YAML
      type: ORACLE

      host: localhost #Replace localhost with your oracle host name
      port: 1521 #Replace the default port number 1521 if needed
      service-name: IO #Replace IO with the service name that contains the schema you will be replicated

      username: 'REPLICANT' #Replace REPLICANT with your username of the user that connects to your oracle server
      password: 'Replicant#123' #Replace Replicant#123 with the your user's password

      stage:
        type: SHARED_FS
        root-dir: /data/shared_fs  #Enter the path of the shared directory

      max-connections: 30 #Maximum number of connections replicant can open in Oracle
    ```

## IV. Setup Applier Configuration

Replicant supports creating/loading tables at the partition and subpartition levels. Follow the instructions below if you want to change the behavior.

1. From `$REPLICANT_HOME`, navigate to the Applier Configuration File:
   ```BASH
   vi conf/dst/oracle.yaml
   ```

2. Make the necessary changes as shown below:
   ```YAML
   snapshot:
     enable-partition-load: true #true by default only if source type is oracle
     disable-partition-wise-load-on-failure: false

     bulk-load : Blitzz can leverage underlying support of FILEbased bulk loading into the target system.
       enable: true|false #To enable/disable bulk loading.
       type: FILE
       serialize: true/false. #If the files generated should be applied in serial/parallel fashion
       method : EXTERNAL_TABLE|SQL_LOADER. #Either external table based approach or sql loader based approach can be taken to perform bulk load.
   ```
For a detailed explanation of configuration parameters in the applier file, read: [Applier Reference]({{< ref "/docs/references/applier-reference" >}} "Applier Reference"). 