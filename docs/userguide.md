# TiSpark (version >= 2.0) User Guide

> **Note:**
>
> This is a user guide for TiSpark version >= 2.0. If you are using version earlier than 2.0, refer to [Document for Spark 2.1](./userguide_spark2.1.md)

This document introduces how to set up and use TiSpark, which requires some basic knowledge of Apache Spark. Refer to [Spark website](https://spark.apache.org/docs/latest/index.html) for details.

# TOC
* [Overview](#overview)
* [Prerequisites for setting up TiSpark](#prerequisites-for-setting-up-tispark)
* [Recommended deployment configurations](#recommended-deployment-configurations)
   + [For independent deployment of Spark cluster and TiSpark cluster](#for-independent-deployment-of-spark-cluster-and-tispark-cluster)
   + [For hybrid deployment of TiSpark and TiKV cluster](#for-hybrid-deployment-of-tispark-and-tikv-cluster)
* [Deploy TiSpark](#deploy-tispark)
   + [Deploy TiSpark on existing Spark cluster](#deploy-tispark-on-existing-spark-cluster)
   + [Deploy TiSpark without Spark cluster](#deploy-tispark-without-spark-cluster)
* [Demonstration](#demonstration)
* [TiSparkR](#tisparkr)
* [TiSpark on PySpark](#tispark-on-pyspark)
* [Use TiSpark with Hive](#use-tispark-with-hive)
* [Load Spark DataFrame into TiDB using TiDB Connector](#load-spark-dataframe-into-tidb-using-tidb-connector)
* [Load Spark DataFrame into TiDB using JDBC](#load-spark-dataframe-into-tidb-using-jdbc)
* [Statistics information](#statistics-information)
* [Reading partition table from TiDB](#reading-partition-table-from-tidb)
* [Common port numbers used by Spark cluster](#common-port-numbers-used-by-spark-cluster)
* [FAQ](#13-faq)
* [Errors and Exceptions](#14-errors-and-exceptions)
   + [Netty OutOfDirectMemoryError](#netty-outofdirectmemoryerror)
   + [Chinese characters are garbled](#chinese-characters-are-garbled)
   + [GRPC message exceeds maximum size error](#grpc-message-exceeds-maximum-size-error)
   
## Overview

TiSpark is a thin layer built for running Apache Spark on top of TiDB/TiKV to answer the complex OLAP queries. While enjoying the merits of both the Spark platform and the distributed clusters of TiKV, it is seamlessly integrated with TiDB, the distributed OLTP database, and thus blessed to provide one-stop Hybrid Transactional/Analytical Processing (HTAP) solutions for online transactions and analyses.

It is an OLAP solution that runs Spark SQL directly on TiKV, the distributed storage engine.

The figure below show the architecture of TiSpark.

![image alt text](architecture.png)

+ TiSpark integrates well with the Spark Catalyst Engine. It provides precise control of computing, which allows Spark to read data from TiKV efficiently. It also supports index seek, which significantly improves the performance of the point query execution.
+ It utilizes several strategies to push down computing to reduce the size of dataset handling by Spark SQL, which accelerates query execution. It also uses the TiDB built-in statistical information for the query plan optimization.
+ From the perspective of data integration, TiSpark + TiDB provides a solution that performs both transaction and analysis directly on the same platform without building and maintaining any ETLs. It simplifies the system architecture and reduces the cost of maintenance.
+ In addition, you can deploy and utilize the tools from the Spark ecosystem for further data processing and manipulation on TiDB. For example, using TiSpark for data analysis and ETL, retrieving data from TiKV as a data source for machine learning, generating reports from the scheduling system and so on.

TiSpark relies on the availability of TiKV clusters and PDs. You also need to set up and use the Spark clustering platform.

## Prerequisites for setting up TiSpark

+ The current TiSpark version supports Spark 2.3.x/2.4.x/3.0.x/3.1.x, but does not support any Spark versions earlier than 2.3.
+ TiSpark requires JDK 1.8+ and Scala 2.11/2.12.
+ TiSpark runs in any Spark mode such as `YARN`, `Mesos`, and `Standalone`.

## Recommended deployment configurations

### For independent deployment of Spark cluster and TiSpark cluster

Refer to the [Spark official website](https://spark.apache.org/docs/latest/hardware-provisioning.html) for detailed hardware recommendations.

+ It is recommended to allocate 32G memory for Spark. Reserve at least 25% of the memory for the operating system and the buffer cache.

+ It is recommended to provision at least 8 to 16 cores per machine for Spark. First, you must assign all the CPU cores to Spark.

The following is an example based on the `spark-env.sh` configuration:

```
SPARK_EXECUTOR_MEMORY = 32g
SPARK_WORKER_MEMORY = 32g
SPARK_WORKER_CORES = 8
```

Add the following lines in `spark-defaults.conf`.

```
spark.tispark.pd.addresses ${your_pd_servers}
spark.sql.extensions org.apache.spark.sql.TiExtensions
```

In the first line above, `your_pd_servers` is the PD addresses separated by commas, each in the format of `$your_pd_address:$port`.
For example, `10.16.20.1:2379,10.16.20.2:2379,10.16.20.3:2379`, which means that you have multiple PD servers on `10.16.20.1,10.16.20.2,10.16.20.3` with the port `2379`.

For TiSpark version >= 2.5.0, please add the following additional configuration to enable `Catalog` provided by `spark-3.0`.
```
spark.sql.catalog.tidb_catalog  org.apache.spark.sql.catalyst.catalog.TiCatalog`
spark.sql.catalog.tidb_catalog.pd.addresses  ${your_pd_adress}
```

### For hybrid deployment of TiSpark and TiKV cluster

For the hybrid deployment of TiSpark and TiKV, add the resources required by TiSpark to the resources reserved in TiKV, and allocate 25% of the memory for the system.

## Deploy TiSpark

Download the TiSpark's jar package from [here](https://github.com/pingcap/tispark/releases).

### Deploy TiSpark on existing Spark cluster

You do not need to reboot the existing Spark cluster for TiSpark to run on it. Instead, use Spark's `--jars` parameter to introduce TiSpark as a dependency:

```
spark-shell --jars $your_path_to/tispark-${name_with_version}.jar
```

To deploy TiSpark as a default component, place the TiSpark jar package into each node's jar path on the Spark cluster and restart the Spark cluster:

```
cp $your_path_to/tispark-${name_with_version}.jar $SPARK_HOME/jars
```

In this way, you can use either `Spark-Submit` or `Spark-Shell` to use TiSpark directly.

### Deploy TiSpark without Spark cluster

Without a Spark cluster, it is recommended that you use the Spark Standalone mode by placing a compiled version of Spark on each node on the cluster. For any problem, refer to the [official Spark website](https://spark.apache.org/docs/latest/spark-standalone.html). You are also welcome to [file an issue](https://github.com/pingcap/tispark/issues/new) on GitHub.

#### Step 1: Download and install

Download [Apache Spark](https://spark.apache.org/downloads.html).

+ For the Standalone mode without Hadoop support, use Spark **2.3.x/2.4.x** and any version of pre-build with Apache Hadoop 2.x with Hadoop dependencies.

+ If you need to use the Hadoop cluster, choose the corresponding Hadoop version. You can also build Spark from the [Spark 2.3 source code](https://spark.apache.org/docs/2.3.4/building-spark.html) or [Spark 2.4 source code](https://spark.apache.org/docs/2.4.4/building-spark.html) to match the previous version of the official Hadoop 2.6.

> **Note:**
>
> Confirm the Spark version that your TiSpark version supports.

Suppose you already have a Spark binary, and the current PATH is `SPARKPATH`, copy the TiSpark jar package to the `$SPARKPATH/jars` directory.

#### Step 2: Start a Master node

Execute the following command on the selected Spark-Master node:

```
cd $SPARKPATH

./sbin/start-master.sh
```

After the command is executed, a log file is printed on the screen. Check the log file to confirm whether the Spark-Master is started successfully.

Open the [http://spark-master-hostname:8080](http://spark-master-hostname:8080) to view the cluster information (if you do not change the default port number of Spark-Master).

When you start Spark-Worker, you can also use this panel to confirm whether the Worker is joined to the cluster.

#### Step 3: Start a Worker node

Similarly, start a Spark-Worker node by executing the following command:

```
./sbin/start-slave.sh spark://spark-master-hostname:7077
```

After the command returns, also check whether the Worker node is joined to the Spark cluster correctly from the panel.

Repeat the above command on all Worker nodes. After all the Workers are connected to the Master, you have a Standalone mode Spark cluster.

#### Step 4: Spark SQL shell and JDBC Server

Use Spark's ThriftServer and SparkSQL directly because TiSpark now supports Spark 2.3/2.4.

## Demonstration

This section briefly introduces how to use Spark SQL for OLAP analysis (assuming that you have successfully started the TiSpark cluster as described above).

The following example uses a table named `lineitem` in the `tpch` database.

1. Add the entry below in your `./conf/spark-defaults.conf`, assuming that your PD node is located at `192.168.1.100`, port `2379`.

    ```
    spark.tispark.pd.addresses 192.168.1.100:2379
    spark.sql.extensions org.apache.spark.sql.TiExtensions
    spark.sql.catalog.tidb_catalog org.apache.spark.sql.catalyst.catalog.TiCatalog
    spark.sql.catalog.tidb_catalog.pd.addresses 192.168.1.100:2379
    ```

2. In the Spark-Shell, enter the following command:

    ```
    spark.sql("use tidb_catalog.tpch")
    ```

3. Call Spark SQL directly:

    ```
    spark.sql("select count (*) from lineitem").show
    ```

    The result:

    ```
    +-------------+
    | Count (1) |
    +-------------+
    | 600000000 |
    +-------------+
    ```

TiSpark's SQL Interactive shell is almost the same as spark-sql shell.

```
spark-sql> use tpch;
Time taken: 0.015 seconds

spark-sql> select count(*) from lineitem;
2000
Time taken: 0.673 seconds, Fetched 1 row(s)
```

For the JDBC connection with Thrift Server, try various JDBC-supported tools including SQuirreL SQL and hive-beeline.

For example, to use it with beeline:

```
./beeline
Beeline version 1.2.2 by Apache Hive
beeline> !connect jdbc:hive2://localhost:10000

1: jdbc:hive2://localhost:10000> use testdb;
+---------+--+
| Result  |
+---------+--+
+---------+--+
No rows selected (0.013 seconds)

select count(*) from account;
+-----------+--+
| count(1)  |
+-----------+--+
| 1000000   |
+-----------+--+
1 row selected (1.97 seconds)
```

## TiSparkR

TiSparkR is a thin layer built for supporting R language with TiSpark. Refer to [this document](../R/README.md) for TiSparkR guide.

## TiSpark on PySpark

TiSpark on PySpark is a Python package build to support Python language with TiSpark. Refer to [this document](../python/README.md) for TiSpark on PySpark guide .

## Use TiSpark with Hive

To use TiSpark with Hive:

1. Set the environment variable `HADOOP_CONF_DIR` to your Hadoop's configuration folder.

2. Copy `hive-site.xml` to the `spark/conf` folder before you start Spark.

    ```
    val tisparkDF = spark.sql("select * from tispark_table").toDF
    tisparkDF.write.saveAsTable("hive_table") // save table to hive
    spark.sql("select * from hive_table a, tispark_table b where a.col1 = b.col1").show // join table across Hive and Tispark
    ```

## Load Spark DataFrame into TiDB using TiDB Connector
TiSpark natively supports writing data to TiKV via Spark Data Source API and guarantees ACID.

For example:

```scala
// tispark will send `lock table` command to TiDB via JDBC
val tidbOptions: Map[String, String] = Map(
  "tidb.addr" -> "127.0.0.1",
  "tidb.password" -> "",
  "tidb.port" -> "4000",
  "tidb.user" -> "root",
  "spark.tispark.pd.addresses" -> "127.0.0.1:2379"
)

val customer = spark.sql("select * from customer limit 100000")

customer.write
.format("tidb")
.option("database", "tpch_test")
.option("table", "cust_test_select")
.options(tidbOptions)
.mode("append")
.save()
```

See [here](./datasource_api_userguide.md) for more details.

## Load Spark DataFrame into TiDB using JDBC

While TiSpark provides a direct way to load data into your TiDB cluster, you can also do it by using JDBC.

For example:

```scala
import org.apache.spark.sql.execution.datasources.jdbc.JDBCOptions

val customer = spark.sql("select * from customer limit 100000")
// you might repartition source to make it balanced across nodes
// and increase concurrency
val df = customer.repartition(32)
df.write
.mode(saveMode = "append")
.format("jdbc")
.option("driver", "com.mysql.jdbc.Driver")
 // replace the host and port with yours and be sure to use rewrite batch
.option("url", "jdbc:mysql://127.0.0.1:4000/test?rewriteBatchedStatements=true")
.option("useSSL", "false")
// as tested, setting to `150` is a good practice
.option(JDBCOptions.JDBC_BATCH_INSERT_SIZE, 150)
.option("dbtable", s"cust_test_select") // database name and table name here
.option("isolationLevel", "NONE") // set isolationLevel to NONE
.option("user", "root") // TiDB user here
.save()
```

Please set `isolationLevel` to `NONE` to avoid large single transactions which might lead to TiDB OOM and also avoid the `ISOLATION LEVEL does not support` error (TiDB currently only supports `REPEATABLE-READ`).

## Statistics information

TiSpark uses the statistic information for:

+ Determining which index to use in your query plan with the lowest estimated cost.
+ Small table broadcasting, which enables efficient broadcast join.

For TiSpark to use the statistic information, first make sure that relevant tables have been analyzed.

See [here](https://github.com/pingcap/docs/blob/master/statistics.md) for more details about how to analyze tables.

Since TiSpark 2.0, statistics information is default to auto-load.

## Reading partition table from TiDB

TiSpark reads the range and hash partition table from TiDB.

Currently, TiSpark doesn't support a MySQL/TiDB partition table syntax `select col_name from table_name partition(partition_name)`, but you can still use `where` condition to filter the partitions.

TiSpark decides whether to apply partition pruning according to the partition type and the partition expression associated with the table.

Currently, TiSpark partially apply partition pruning on range partition.

The partition pruning is applied when the partition expression of the range partition is one of the following:

+ column expression
+ `YEAR` (expression) where the expression is a column and its type is datetime or string literal
that can be parsed as datetime.

If partition pruning is not applied, TiSpark's reading is equivalent to doing a table scan over all partitions.

## Common port numbers used by Spark cluster

|Port Name| Default Port Number   | Configuration Property   | Notes|
|---------------| ------------- |-----|-----|
|Master web UI  | `8080`  | spark.master.ui.port  or SPARK_MASTER_WEBUI_PORT| The value set by `spark.master.ui.port` takes precedence.  |
|Worker web UI  |  `8081`  | spark.worker.ui.port or SPARK_WORKER_WEBUI_PORT  | The value set by `spark.worker.ui.port` takes precedence.|
|History server web UI   |  `18080`  | spark.history.ui.port  |Optional; it is only applied if you use the history server.   |
|Master port   |  `7077`  |   SPARK_MASTER_PORT  |   |
|Master REST port   |  `6066`  | spark.master.rest.port  | Not needed if you disable the `REST` service.   |
|Worker port |  (random)   |  SPARK_WORKER_PORT |   |
|Block manager port  |(random)   | spark.blockManager.port  |   |
|Shuffle server  |  `7337`   | spark.shuffle.service.port  |  Optional; it is only applied if you use the external shuffle service.  |
|  Application web UI  |  `4040`  |  spark.ui.port | If `4040` has been occupied, then `4041` is used. |

## FAQ

Q: What are the pros and cons of independent deployment as opposed to a shared resource with an existing Spark / Hadoop cluster?

A: You can use the existing Spark cluster without a separate deployment, but if the existing cluster is busy, TiSpark will not be able to achieve the desired speed.

Q: Can I mix Spark with TiKV?

A: If TiDB and TiKV are overloaded and run critical online tasks, consider deploying TiSpark separately.

You also need to consider using different NICs to ensure that OLTP's network resources are not compromised so that online business is not affected.

If the online business requirements are not high or the loading is not large enough, you can mix TiSpark with TiKV deployment.

Q: How to use PySpark with TiSpark?

A: Follow [TiSpark on PySpark](../python/README.md).

Q: How to use SparkR with TiSpark?

A: Follow [TiSpark on SparkR](../R/README.md).

## Errors and Exceptions

### Netty OutOfDirectMemoryError

Netty's `PoolThreadCache` may hold some unused memory, which may cause the following error.

```
Caused by: shade.io.netty.handler.codec.DecoderException: shade.io.netty.util.internal.OutOfDirectMemoryError
```

The following configurations can be used to avoid the error.

```
--conf "spark.driver.extraJavaOptions=-Dshade.io.netty.allocator.type=unpooled"
--conf "spark.executor.extraJavaOptions=-Dshade.io.netty.allocator.type=unpooled"
```

### Chinese characters are garbled

The following configurations can be used to avoid the garbled chinese characters problem.

```
--conf "spark.driver.extraJavaOptions=-Dfile.encoding=UTF-8"
--conf "spark.executor.extraJavaOptions=-Dfile.encoding=UTF-8"
```
### GRPC message exceeds maximum size error

The maximum message size of GRPC java lib is 2G. The following error will be thrown if there is a huge region in TiKV whose size is more than 2G.

```
Caused by: shade.io.grpc.StatusRuntimeException: RESOURCE_EXHAUSTED: gRPC message exceeds maximum size 2147483647
```

Use `SHOW TABLE [table_name] REGIONS [WhereClauseOptional]` to check whether there is a huge region in TiKV.
