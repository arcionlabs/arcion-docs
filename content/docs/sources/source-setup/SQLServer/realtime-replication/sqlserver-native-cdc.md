---
pageTitle: SQL Server native CDC for real-time replication
title: "SQL Server native CDC"
description: "Learn how to use Microsoft SQL Server's native CDC functionality to extract and replicate data in real time."
weight: 3
url: docs/source-setup/sqlserver/realtime-replication/sqlserver-native-cdc
---

# Real-time replication using SQL Server native CDC
For real-time replicaiton from SQL Server, you can choose to use the native change data capture (CDC) functionality of SQL Server to perform data extraction and replication. Follow these steps to set up real-time replication using the SQL Server's native CDC.

## I. Prerequisites
### Required permissions
To allow replication, first verify that the user possesses the necessary permissions on source SQL Server. For more information, see [SQL Server user permissions]({{< relref "../../../source-prerequisites/sqlserver#sql-server-user-permissions" >}}).

## II. Set up connection configuration
Specify the connection details of your SQL Server instance to Replicant in one of the following ways:

- [A connection configuration file](#using-a-connection-configuration-file)
- [Secrets management service](#use-a-secrets-management-service)
- [KeyStore](#using-keystore-for-credentials)

### Use a connection configuration file
The connection configuration fild holds the connection details and login credentials.
You can find a sample connection configuration file `sqlserver_src.yaml` in the `$REPLICANT_HOME/conf/conn` directory. The following configuration parameters are available:

#### `type`
The connection type representing the database. In this case, it's `SQLSERVER`.

#### `host`
The hostname of your SQL Server system.

#### `port`
The port number to connect to the `host`.

#### `username`
The username credential to access the SQL Server system.

#### `password`
The password associated with `username`.

#### `auth-type`
The authentication protocol for the connection. The following protocols are supported:

- `NATIVE` (Default)
- `NLTM`
    
Authentication protocol always defaults to `NATIVE` if you don't explicitly set the `auth-type` parameter.

In case of `NLTM` protocol, provide the [`username`](#username) in `DOMAIN\USER` format—for example, `domain\alex`.

#### `database`
Specify the database name if the source is an Azure SQL Managed Instance.

#### `extractor`
The CDC Extractor to use for real-time replication. 

To use SQL Server's native CDC functionality as the CDC Extractor, follow these steps:

- Set `extractor` to `CDC`.
- Follow the instructions in [Enable CDC in SQL Server](#enable-cdc-in-sql-server).

#### `max-connections` 
The maximum number of connections Replicant uses to load data into the SQL Server system.

The following shows a sample connection configuration:


```YAML
type: SQLSERVER

host: localhost
port: 1433

username: 'alex'
password: 'alex1234'
auth-type: NATIVE
database: 'tpcc'

extractor: CDC

max-connections: 30
```

### Use a secrets management service
You can store your connection credentials in a secrets management service and tell Replicant to retrieve the credentials. For more information, see [Secrets management]({{< ref "docs/security/secrets-management" >}}). 

### Use KeyStore for credentials
Replicant supports consuming login credentials from a _credentials store_. Instead of specifying username and password [in plain text](#use-a-connection-configuration-file), you can keep them in a KeyStore and provide the KeyStore details in the connection configuration file like below:

```YAML
credential-store:
  type: {PKCS12|JKS|JCEKS}
  path: PATH_TO_KEYSTORE_FILE
  key-prefix: PREFIX_OF_THE_KEYSTORE_ENTRY
  password: KEYSTORE_PASSWORD
```

Replace the following:

- *`PATH_TO_KEYSTORE_FILE`*: The path to your KeyStore file.
- *`PREFIX_OF_THE_KEYSTORE_ENTRY`*: The prefix of your KeyStore entries. You can create entries in the credential store using a prefix that preceeds each credential alias. For example, you can create KeyStore entries with aliases `sqlserver_username` and `sqlserver_password`. You can then set `key-prefix` to `sqlserver_`.
- *`KEYSTORE_PASSWORD`*: The KeyStore password. This parameter is optional. If you don’t want to specify the KeyStore password here, then you must use the UUID from your license file as the KeyStore password. Remember to keep your license file somewhere safe in order to keep the KeyStore password secure.


## III. Set up Extractor configuration
To configure real-time replication according to your requirements, specify your configuration in the Extractor configuration file. You can find a sample `sqlserver.yaml` in the `$REPLICANT_HOME/conf/src` directory. 

All configuration parameters for `realtime` mode live under the `realtime` section. The following shows a sample configuration:

```YAML
realtime:
  threads: 4
  fetch-size-rows: 10000
  fetch-duration-per-extractor-slot-s: 3
```

For more information about the configuration parameters in `realtime` mode, see [Realtime mode]({{< ref "../../../configuration-files/extractor-reference#realtime-mode" >}}).

## Enable CDC in SQL Server
CDC allows capturing every DML and some DDL operations that occur in the database. To enable logging of DDL and DML records, you must enable CDC at database and table level.

### Enable for a database
The following stored procedure enables CDC for a database:

```SQL
EXEC sys.sp_cdc_enable_db 
```

For more information, see [Enable change data capture for a database](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/enable-and-disable-change-data-capture-sql-server?view=sql-server-ver16&source=recommendations#enable-for-a-database).

### Enable for a table
The following stored procedure enables CDC for a table:

```SQL
EXEC sys.sp_cdc_enable_table  
@source_schema = N'dbo',  
@source_name   = N'MyTable',  
@role_name     = NULL,  
@supports_net_changes = 1
```

Use the appropriate schema, source, and role names in the preceding command. The `@support_net_changes` parameter only supports a value of `1` if the table has primary key. For more information, see [Enable change data capture for a table](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/enable-and-disable-change-data-capture-sql-server?view=sql-server-ver16&source=recommendations#enable-for-a-table).

Replicant throws error if you don't enable CDC for either database or table.


