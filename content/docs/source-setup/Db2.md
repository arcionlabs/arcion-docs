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
