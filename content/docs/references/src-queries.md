---
pageTitle: Replicate custom queries, macros, UDFs, stored procedures
title: Replication of source queries
description: "Learn how to replicate the results of custom-queries, macros, UDFs, stored procedures from source to the target database systems."
weight: 13
bookHidden: false
---

# Replication of source queries
Replicant can perform replication of the results of source queries from source database systems to the target database systems. Source queries can be custom queries, macros, user-defined functions (UDFs), and stored procedures.
 
Source query replication is supported only for snapshot mode.

To use this feature, follow the instructions in the subsequent sections.

## Configure `src-queries` parameters
The configuration for source queries lives in a YAML configuration file. You can find sample configuration files in the `$REPLICANT_HOME/conf/query/src` directory, where `$REPLICANT_HOME` is [your Arcion self-hosted download]({{< ref "docs/quickstart/#ii-download-replicant-and-create-a-home-repository" >}}).

The file starts with a top-level `src-queries` field, indicating that it's a source queries configuration. Under this field, the following parameters are available to configure your queries and macros:

### `catalog`
The source catalog (if applicable) whose queries or macros you need to replicate.

### `schema`
The source schema (if applicable) whose queries or macros you need to replicate.

### `macros`
Here you can specify your macros. Each macro goes in a separate line and follows this syntax:

```YAML
MACRO_NAME: MACRO_COMMAND
```

`MACRO_COMMAND` is the exact macro that you want Replicant to replicate. `MACRO_NAME` acts as a logical name of the macro command. Replicant uses the same name for the table containing the results of this macro.

{{< hint "info" >}}
**Note:** In order to specify incremental replication from a macro, the macro must accept a value of `delta-snapshot-key` as one of its arguments. This value works as an initial maximum value. 

You must specify specify this value in the macro command at the appropriate argument location by enclosing it in `$` characters.

[See the example](#example) to better understand how to define the value within the macro.

For more information about `delta-snapshot-key` in Extractor configuration, see [Delta snapshot mode]({{< ref "docs/references/extractor-reference#delta-snapshot-mode" >}}). 
{{< /hint >}}

### `queries`
Here you can specify your queries. Each query goes in a separate line and follows [the same syntax as `macros`](#macros):

```YAML
QUERY_NAME: SQL_QUERY
```

`SQL_QUERY` is the exact SQL query that you want Replicant to replicate. `QUERY_NAME` acts as a logical name of the query. Replicant uses the same name for the table containing the results of this query.

[See the example](#example) to better understand how to define queries.


### Example configuration
The following is a sample configuration for SingleStore:

```YAML
src-queries :
- catalog : tpch
  macros :
    partsupp_macro: "call tpch.partsupp_macro('2020-02-17 04:35:17.520000')"
    partsupp_macro: "call tpch.partsupp_macro($'2020-02-17 04:35:17.520000'$)"
  queries :
    PART_sql: "select P_PARTKEY, P_NAME, blitzz_io_delta_snapshot_key from tpch.PART where replicate_io_delta_snapshot_key >= $'2020-07-21 05:43:24'$ "
```

## Whitelist queries in the filter file
After you've configured [the `src-queries` configuration file](#configure-src-queries-parameters), include the special tag `QUERY` inside your filter `types` list. This instructs Replicant to allow all the queries under that specific catalog or schema. For example:

```YAML
allow:
- schema: "tpch"
  types: [QUERY]
```

If the filter `types` list contains any other type besides `QUERY`, you must explicitly specify the logical names under the `allow` field. These logical names are [`MACRO_NAME`](#macros) and [`QUERY_NAME`](#queries) that you defined in [the `src-queries` configuration file](#configure-src-queries-parameters). For example, the following sample specifies the `ng_test_tbd_sql` query and the tables under the `allow` field.

```YAML
allow:
- schema : "tpch"
  types: [TABLE, QUERY]
  allow:
    nation:
    region:
    ng_test_tbd_sql:
    supplier:  
```

## Run Replicant
After you have a `src-queries` configuration file ready, run Replicant with the `--src-queries` option. The option takes the full path to your `src-queries` configuration file as its argument. For example:

```sh
./bin/replicant full conf/conn/singlestore.yaml conf/conn/db2.yaml \
--src-queries conf/query/src/singlestore_queries.yaml
```