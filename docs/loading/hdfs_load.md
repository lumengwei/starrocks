# Load data from HDFS

StarRocks supports using the [Broker Load](../sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md) to load large amounts of data from HDFS into StarRocks.

Broker Load runs in asynchronous loading mode. After you submit a load job, StarRocks asynchronously runs the job, and you need to use the [SHOW LOAD](../sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md) statement or the `curl` command to check the result of the job.

Broker Load ensures the transactional atomicity of each load job that is run to load multiple data files, which means that the loading of multiple data files in one load job must all succeed or fail. It never happens that the loading of some data files succeeds while the loading of the other files fails.

Additionally, Broker Load supports data transformation at data loading and supports data changes made by UPSERT and DELETE operations during data loading. For more information, see [Transform data at loading](../loading/Etl_in_loading.md) and [Change data through loading](../loading/Load_to_Primary_Key_tables.md).

> **NOTICE**
>
> You can load data into StarRocks tables only as a user who has the INSERT privilege on those StarRocks tables. If you do not have the INSERT privilege, follow the instructions provided in [GRANT](../sql-reference/sql-statements/account-management/GRANT.md) to grant the INSERT privilege to the user that you use to connect to your StarRocks cluster.

## Background information

In v2.4 and earlier, StarRocks depends on brokers to set up connections between your StarRocks cluster and your external storage system when it runs a Broker Load job. Therefore, you need to input `WITH BROKER "<broker_name>"` to specify the broker you want to use in the load statement. This is called "broker-based loading." A broker is an independent, stateless service that is integrated with a file-system interface. With brokers, StarRocks can access and read data files that are stored in your external storage system, and can use its own computing resources to pre-process and load the data of these data files.

From v2.5 onwards, StarRocks no longer depends on brokers to set up connections between your StarRocks cluster and your external storage system when it runs a Broker Load job. Therefore, you no longer need to specify a broker in the load statement, but you still need to retain the `WITH BROKER` keyword. This is called "broker-free loading."

When your data is stored in HDFS, you may encounter situations where broker-free loading does not work. This can happen when your data is stored across multiple HDFS clusters or when you have configured multiple Kerberos users. In these situations, you can resort to using broker-based loading instead. To do this successfully, make sure that at least one independent broker group is deployed. For information about how to specify authentication configuration and HA configuration in these situations, see [HDFS](../sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md#hdfs).

> **NOTE**
>
> You can use the [SHOW BROKER](../sql-reference/sql-statements/Administration/SHOW%20BROKER.md) statement to check for brokers that are deployed in your StarRocks cluster. If no brokers are deployed, you can deploy brokers by following the instructions provided in [Deploy a broker](../deployment/deploy_broker.md).

## Supported data file formats

Broker Load supports the following data file formats:

- CSV

- Parquet

- ORC

> **NOTE**
>
> For CSV data, take note of the following points:
>
> - You can use a UTF-8 string, such as a comma (,), tab, or pipe (|), whose length does not exceed 50 bytes as a text delimiter.
> - Null values are denoted by using `\N`. For example, a data file consists of three columns, and a record from that data file holds data in the first and third columns but no data in the second column. In this situation, you need to use `\N` in the second column to denote a null value. This means the record must be compiled as `a,\N,b` instead of `a,,b`. `a,,b` denotes that the second column of the record holds an empty string.

## How it works

After you submit a load job to an FE, the FE generates a query plan, splits the query plan into portions based on the number of available BEs and the size of the data file you want to load, and then assigns each portion of the query plan to an available BE. During the load, each involved BE pulls the data of the data file from your external storage system, pre-processes the data, and then loads the data into your StarRocks cluster. After all BEs finish their portions of the query plan, the FE determines whether the load job is successful.

The following figure shows the workflow of a Broker Load job.

![Workflow of Broker Load](../assets/broker_load_how-to-work_en.png)

## Prepare data examples

1. Log in to your HDFS cluster and create two CSV-formatted data files, `file1.csv` and `file2.csv`, in a specified path (for example, `/user/starrocks/`). Both files consist of three columns, which represent user ID, user name, and user score in sequence.

   - `file1.csv`

     ```Plain
     1,Lily,21
     2,Rose,22
     3,Alice,23
     4,Julia,24
     ```

   - `file2.csv`

     ```Plain
     5,Tony,25
     6,Adam,26
     7,Allen,27
     8,Jacky,28
     ```

2. Log in to your StarRocks database (for example, `test_db`) and create two Primary Key tables, `table1` and `table2`. Both tables consist of three columns: `id`, `name`, and `score`, of which `id` is the primary key.

   ```SQL
   CREATE TABLE `table1`
      (
          `id` int(11) NOT NULL COMMENT "user ID",
          `name` varchar(65533) NULL DEFAULT "" COMMENT "user name",
          `score` int(11) NOT NULL DEFAULT "0" COMMENT "user score"
      )
          ENGINE=OLAP
          PRIMARY KEY(`id`)
          DISTRIBUTED BY HASH(`id`);
             
   CREATE TABLE `table2`
      (
          `id` int(11) NOT NULL COMMENT "用户 ID",
          `name` varchar(65533) NULL DEFAULT "" COMMENT "user name",
          `score` int(11) NOT NULL DEFAULT "0" COMMENT "user score"
      )
          ENGINE=OLAP
          PRIMARY KEY(`id`)
          DISTRIBUTED BY HASH(`id`);
   ```

## Create a load job

Note that the following examples use the CSV format and the simple authentication method. For information about how to load data in other formats, how to specify HA configurations, and about the authentication parameters that you need to configure when using the Kerberos authentication method, see [BROKER LOAD](../sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md).

### Load a single data file into a single table

#### Example

Execute the following statement to load the data of `file1.csv` into `table1`:

```SQL
LOAD LABEL test_db.label_brokerload_singlefile_singletable
(
    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
)
WITH BROKER
(
    "username" = "<hdfs_username>",
    "password" = "<hdfs_password>"
)
PROPERTIES
(
    "timeout" = "3600"
);
```

#### Query data

After you submit the load job, you need to use [SHOW LOAD](../sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md) or `curl` to view the result of the load job. For more information, see the "[View a load job](#view-a-load-job)" section of this topic.

After you confirm that the load job is successful, you can use [SELECT](../sql-reference/sql-statements/data-manipulation/SELECT.md) to query the data of `table1`:

```SQL
SELECT * FROM table1;
+------+-------+-------+
| id   | name  | score |
+------+-------+-------+
|    1 | Lily  |    21 |
|    2 | Rose  |    22 |
|    3 | Alice |    23 |
|    4 | Julia |    24 |
+------+-------+-------+
4 rows in set (0.01 sec)
```

### Load multiple data files into a single table

#### Example

Execute the following statement to load the data of all data files (`file1.csv` and `file2.csv`) stored in the `/user/starrocks/` path of your HDFS cluster into `table1`:

```SQL
LOAD LABEL test_db.label_brokerload_allfile_singletable
(
    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/*")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
)
WITH BROKER
(
    "username" = "<hdfs_username>",
    "password" = "<hdfs_password>"
)
PROPERTIES
(
    "timeout" = "3600"
);
```

#### Query data

After you submit the load job, you need to use [SHOW LOAD](../sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md) or `curl` to view the result of the load job. For more information, see the "[View a load job](#view-a-load-job)" section of this topic.

After you confirm that the load job is successful, you can use [SELECT](../sql-reference/sql-statements/data-manipulation/SELECT.md) to query the data of `table1`:

```SQL
SELECT * FROM table1;
+------+-------+-------+
| id   | name  | score |
+------+-------+-------+
|    1 | Lily  |    21 |
|    2 | Rose  |    22 |
|    3 | Alice |    23 |
|    4 | Julia |    24 |
|    5 | Tony  |    25 |
|    6 | Adam  |    26 |
|    7 | Allen |    27 |
|    8 | Jacky |    28 |
+------+-------+-------+
4 rows in set (0.01 sec)
```

### Load multiple data files into multiple tables

#### Example

Execute the following statement to load the data of `file1.csv` and `file2.csv` into `table1` and `table2`, respectively:

```SQL
LOAD LABEL test_db.label_brokerload_multiplefile_multipletable
(
    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/file1.csv")
    INTO TABLE table1
    COLUMNS TERMINATED BY ","
    (id, name, score)
    ,
    DATA INFILE("hdfs://<hdfs_host>:<hdfs_port>/user/starrocks/file2.csv")
    INTO TABLE table2
    COLUMNS TERMINATED BY ","
    (id, name, score)
)
WITH BROKER
(
    "username" = "<hdfs_username>",
    "password" = "<hdfs_password>"
)
PROPERTIES
(
    "timeout" = "3600"
);
```

#### Query data

After you submit the load job, you need to use [SHOW LOAD](../sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md) or `curl` to view the result of the load job. For more information, see the "[View a load job](#view-a-load-job)" section of this topic.

After you confirm that the load job is successful, you can use [SELECT](../sql-reference/sql-statements/data-manipulation/SELECT.md) to query the data of `table1` and `table2`:

1. Query `table1`:

   ```SQL
   SELECT * FROM table1;
   +------+-------+-------+
   | id   | name  | score |
   +------+-------+-------+
   |    1 | Lily  |    21 |
   |    2 | Rose  |    22 |
   |    3 | Alice |    23 |
   |    4 | Julia |    24 |
   +------+-------+-------+
   4 rows in set (0.01 sec)
   ```

2. Query `table2`:

   ```SQL
   SELECT * FROM table2;
   +------+-------+-------+
   | id   | name  | score |
   +------+-------+-------+
   |    5 | Tony  |    25 |
   |    6 | Adam  |    26 |
   |    7 | Allen |    27 |
   |    8 | Jacky |    28 |
   +------+-------+-------+
   4 rows in set (0.01 sec)
   ```

## View a load job

Broker Load allows you to view a lob job by using the SHOW LOAD statement or the `curl` command.

### Use SHOW LOAD

For more information, see [SHOW LOAD](../sql-reference/sql-statements/data-manipulation/SHOW%20LOAD.md).

### Use curl

The syntax is as follows:

```Bash
curl --location-trusted -u <username>:<password> \
    'http://<fe_host>:<fe_http_port>/api/<database_name>/_load_info?label=<label_name>'
```

> **NOTE**
>
> - If you use an account for which no password is set, you need to input only `<username>:`.
> - You can use [SHOW FRONTENDS](../sql-reference/sql-statements/Administration/SHOW%20FRONTENDS.md) to view the IP address and HTTP port of the FE node.

For example, you can run the following command to view the information about one of the preceding load jobs, whose label is `label_brokerload_multiplefile_multipletable`, in the `test_db` database:

```Bash
curl --location-trusted -u <username>:<password> \
    'http://<fe_host>:<fe_http_port>/api/test_db/_load_info?label=label_brokerload_multiplefile_multipletable'
```

The `curl` command returns the information about the load job as a JSON object `jobInfo`:

```JSON
{"jobInfo":{"dbName":"test_db","tblNames":["table1","table2"],"label":"label_brokerload_multiplefile_multipletable","state":"FINISHED","failMsg":"","trackingUrl":""},"status":"OK","msg":"Success"}
```

The following table describes the parameters in `jobInfo`.

| **Parameter** | **Description**                                              |
| ------------- | ------------------------------------------------------------ |
| dbName        | The name of the database into which data is loaded           |
| tblNames      | The name of the table into which data is loaded.             |
| label         | The label of the load job.                                   |
| state         | The status of the load job. Valid values:<ul><li>`PENDING`: The load job is in queue waiting to be scheduled.</li><li>`QUEUEING`: The load job is in the queue waiting to be scheduled.</li><li>`LOADING`: The load job is running.</li><li>`PREPARED`: The transaction has been committed.</li><li>`FINISHED`: The load job succeeded.</li><li>`CANCELLED`: The load job failed.</li></ul>For more information, see the "Asynchronous loading" section in [Overview of data loading](../loading/Loading_intro.md). |
| failMsg       | The reason why the load job failed. If the `state` value for the load job is `PENDING`, `LOADING`, or `FINISHED`, `NULL` is returned for the `failMsg` parameter. If the `state` value for the load job is `CANCELLED`, the value returned for the `failMsg` parameter consists of two parts: `type` and `msg`.<ul><li>The `type` part can be any of the following values:</li><ul><li>`USER_CANCEL`: The load job was manually canceled.</li><li>`ETL_SUBMIT_FAIL`: The load job failed to be submitted.</li><li>`ETL-QUALITY-UNSATISFIED`: The load job failed because the percentage of unqualified data exceeds the value of the `max-filter-ratio` parameter.</li><li>`LOAD-RUN-FAIL`: The load job failed in the `LOADING` stage.</li><li>`TIMEOUT`: The load job failed to finish within the specified timeout period.</li><li>`UNKNOWN`: The load job failed due to an unknown error.</li></ul><li>The `msg` part provides the detailed cause of the load failure.</li></ul> |
| trackingUrl   | The URL that is used to access the unqualified data detected in the load job. You can use the `curl` or `wget` command to access the URL and obtain the unqualified data. If no unqualified data is detected, `NULL` is returned for the `trackingUrl` parameter. |
| status        | The status of the HTTP request for the load job. Valid values: `OK` and `Fail`. |
| msg           | The error information of the HTTP request for the load job.  |

## Cancel a load job

When a load job is not in the **CANCELLED** or **FINISHED** stage, you can use the [CANCEL LOAD](../sql-reference/sql-statements/data-manipulation/CANCEL%20LOAD.md) statement to cancel the job.

For example, you can execute the following statement to cancel a load job, whose label is `label1`, in the database `test_db`:

```SQL
CANCEL LOAD
FROM test_db
WHERE LABEL = "label1";
```

## Job splitting and concurrent running

A Broker Load job can be split into one or more tasks that concurrently run. The tasks within a load job are run within a single transaction. They must all succeed or fail. StarRocks splits each load job based on how you declare `data_desc` in the `LOAD` statement:

- If you declare multiple `data_desc` parameters, each of which specifies a distinct table, a task is generated to load the data of each table.

- If you declare multiple `data_desc` parameters, each of which specifies a distinct partition for the same table, a task is generated to load the data of each partition.

Additionally, each task can be further split into one or more instances, which are evenly distributed to and concurrently run on the BEs of your StarRocks cluster. StarRocks splits each task based on the following [FE configurations](../administration/Configuration.md#fe-configuration-items):

- `min_bytes_per_broker_scanner`: the minimum amount of data processed by each instance. The default amount is 64 MB.

- `load_parallel_instance_num`: the number of concurrent instances allowed in each load job on an individual BE. The default number is 1.
  
  You can use the following formula to calculate the number of instances in an individual task:

  **Number of instances in an individual task = min(Amount of data to be loaded by an individual task/`min_bytes_per_broker_scanner`,`load_parallel_instance_num` x Number of BEs)**

In most cases, only one `data_desc` is declared for each load job, each load job is split into only one task, and the task is split into the same number of instances as the number of BEs.

## Related configuration items

The [FE configuration item](../administration/Configuration.md#fe-configuration-items) `max_broker_load_job_concurrency` specifies the maximum number of tasks that can be concurrently run for Broker Load within your StarRocks cluster.

In StarRocks v2.4 and earlier, if the total number of tasks generated for Broker Load jobs that are submitted within a specific period of time exceeds the maximum number, excessive jobs are queued and scheduled based on their submission time.

Since StarRocks v2.5,  if the total number of tasks generated for Broker Load jobs that are submitted within a specific period of time exceeds the maximum number, excessive jobs are queued and scheduled based on their priorities. You can specify a priority for a job by using the `priority` parameter at job creation. See [BROKER LOAD](../sql-reference/sql-statements/data-manipulation/BROKER%20LOAD.md#opt_properties). You can also use [ALTER LOAD](../sql-reference/sql-statements/data-manipulation/ALTER%20LOAD.md) to modify the priority of an existing job that is in the **QUEUEING** or **LOADING** state.
