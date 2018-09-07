# PXF JDBC plugin

The PXF JDBC plugin allows to access external databases that implement [the Java Database Connectivity API](http://www.oracle.com/technetwork/java/javase/jdbc/index.html). Both read (SELECT) and write (INSERT) operations are supported by the plugin.

PXF JDBC plugin is a JDBC client. The host running the external database does not need to deploy PXF.


## Prerequisites

Check the following before using the PXF JDBC plugin:

* The PXF JDBC plugin is installed on all PXF nodes;
* The JDBC driver for external database is installed on all PXF nodes;
* All PXF nodes are allowed to connect to the external database.


## Limitations

Both **PXF table** **and** a **table in external database** **must have the same definiton**. Their columns must have the same names, and the columns' types must correspond.

**Not all data types are supported** by the plugin. The following PXF data types are supported:

* `INTEGER`, `BIGINT`, `SMALLINT`
* `REAL`, `FLOAT8`
* `NUMERIC`
* `BOOLEAN`
* `VARCHAR`, `BPCHAR`, `TEXT`
* `DATE`
* `TIMESTAMP`
* `BYTEA`

The `<full_external_table_name>` (see below) **must not match** the [pattern](https://docs.oracle.com/javase/7/docs/api/java/util/regex/Pattern.html) `/.*/[0-9]*-[0-9]*_[0-9]*` (the name must not start with `/` and have an ending that consists of `/` and three groups of numbers of arbitrary length, the first two separated by `-` and the last two separated by `_`. For example, the following table name is not allowed: `/public.table/1-2_3`).

At the moment, one PXF external table cannot serve both SELECT and INSERT queries. A separate PXF external table is required for each type of queries.


## Syntax
```
CREATE [ READABLE | WRITABLE ] EXTERNAL TABLE <table_name> (
    { <column_name> <data_type> [, ...] | LIKE <other_table> }
)
LOCATION (
    'pxf://<full_external_table_name>?<pxf_parameters><jdbc_required_parameters><jdbc_login_parameters><plugin_parameters>'
)
FORMAT 'CUSTOM' (FORMATTER={'pxfwritable_import' | 'pxfwritable_export'})
```

The **`<pxf_parameters>`** are:
```
{
PROFILE=JDBC
|
FRAGMENTER=org.apache.hawq.pxf.plugins.jdbc.JdbcPartitionFragmenter
&ACCESSOR=org.apache.hawq.pxf.plugins.jdbc.JdbcAccessor
&RESOLVER=org.apache.hawq.pxf.plugins.jdbc.JdbcResolver
}
```

The **`<jdbc_required_parameters>`** are:
```
&JDBC_DRIVER=<external_database_jdbc_driver>
&DB_URL=<external_database_url>
```

The **`<jdbc_login_parameters>`** are **optional**, but if provided, both of them must be present:
```
&USER=<external_database_login>
&PASS=<external_database_password>
```

The **`<plugin_parameters>`** are **optional**.

The meaning of `BATCH_SIZE` is given in section [batching of INSERT queries](#Batching).

The meaning of other parameters is given in section [partitioning](#Partitioning).
```
{
&BATCH_SIZE=<batch_size>
|
&PARTITION_BY=<column>:<column_type>
&RANGE=<start_value>:<end_value>
[&INTERVAL=<value>[:<unit>]]
}
```


## SELECT queries

The PXF JDBC plugin allows to perform SELECT queries to external tables.

To perform SELECT queries, create an `EXTERNAL READABLE TABLE` or just an `EXTERNAL TABLE` with `FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import')` in PXF.

The `BATCH_SIZE` parameter is *not used* in such tables. *However*, if this parameter is present, its value will be checked for correctness (it must be an integer).


## INSERT queries

The PXF JDBC plugin allows to perform INSERT queries to external tables.

To perform INSERT queries, create an `EXTERNAL WRITABLE TABLE` with `FORMAT 'CUSTOM' (FORMATTER='pxfwritable_export')` in PXF.

The `PARTITION_BY`, `RANGE` and `INTERVAL` is ignored in such tables.


### Batching

The INSERT queries can be batched. This may significantly increase perfomance if batching is supported by the external database.

If an **external database does not support transactions** and the INSERTion of one of the tuples failed, some tuples still may be inserted into the external database. The actual result depends on the external database.

To enable batching, create an external table with the parameter `BATCH_SIZE` set to:
* `integer > 1`. Batches of the given size will be used;
* `integer < 0`. A batch of infinite size will be used (all tuples will be sent in one huge JDBC query). Note that this behaviour **may cause errors**, as each database has its own limit on the size of JDBC queries;
* `0` or `1`. Batching will not be used.

Batching must be supported by the JDBC driver of an external database. If the driver does not support batching, it will not be used, but PXF plugin will try to INSERT values anyway, and an information message will be added to PXF logs.

By default (`BATCH_SIZE` is absent), batching is not used.


## Partitioning

PXF JDBC plugin supports simultaneous *read* access to an external table from multiple PXF segments. This function is called partitioning.


### Syntax
Use the following `<plugin_parameters>` (mentioned above) to activate partitioning:

```
&PARTITION_BY=<column>:<column_type>
&RANGE=<start_value>:<end_value>
[&INTERVAL=<value>[:<unit>]]
```

* The `PARTITION_BY` parameter indicates which column to use as a partition column. Only one column can be used as the partition column
    * The `<column>` is the name of a partition column;
    * The `<column_type>` is the data type of a partition column. At the moment, the **supported types** are `INT`, `DATE` and `ENUM`.

* The `RANGE` parameter indicates the range of data to be queried.
    * If the partition type is `ENUM`, the `RANGE` parameter must be a list of values, each of which forms its own fragment;
    * If the partition type is `INT` or `DATE`, the `RANGE` parameter must be a finite left-closed range ( `... >= start_value AND ... < end_value`);
    * For `DATE` partitions, the date format must be `yyyy-MM-dd`.

* The `INTERVAL` parameter is **required** for `INT` and `DATE` partitions. It is ignored if `<column_type>` is `ENUM`.
    * The `<value>` is the size of each fragment (the last one may be made smaller by the plugin);
    * The `<unit>` **must** be provided if `<column_type>` is `DATE`. `year`, `month` and `day` are supported. This parameter is ignored in case of any other `<column_type>`.

Example partitions:
* `&PARTITION_BY=id:int&RANGE=42:142&INTERVAL=2`
* `&PARTITION_BY=createdate:date&RANGE=2008-01-01:2010-01-01&INTERVAL=1:month`
* `&PARTITION_BY=grade:enum&RANGE=excellent:good:general:bad`


### Mechanism

Note that **by default PXF does not support more than 100 fragments**.

If partitioning is activated, the SELECT query is split into a set of small queries, each of which is called a *fragment*. All the fragments are processed by separate PXF instances simultaneously. If there are more fragments than PXF instances, some instances will process more than one fragment; if only one PXF instance is available, it will process all the fragments.

Extra query constraints (`WHERE` expressions) are automatically added to each fragment to guarantee that every tuple of data is retrieved from the external database exactly once.


### Partitioning example
Consider the following MySQL table:
```
CREATE TABLE sales (
    id int primary key,
    cdate date,
    amt decimal(10,2),
    grade varchar(30)
)
```
and the following HAWQ table:
```
CREATE EXTERNAL TABLE sales(
    id integer,
    cdate date,
    amt float8,
    grade text
)
LOCATION ('pxf://sales?PROFILE=JDBC&JDBC_DRIVER=com.mysql.jdbc.Driver&DB_URL=jdbc:mysql://192.168.200.6:3306/demodb&PARTITION_BY=cdate:date&RANGE=2008-01-01:2010-01-01&INTERVAL=1:year')
FORMAT 'CUSTOM' (FORMATTER='pxfwritable_import');
```

The PXF JDBC plugin will generate two fragments  for a query `SELECT * FROM sales`. Then HAWQ will assign each of them to a separate PXF segment. Each segment will perform the SELECT query, and the first one will get tuples with `cdate` values for year `2008`, while the second will get tuples for year `2009`. Then each PXF segment will send its results back to HAWQ, where they will be "concatenated" and returned.


## Examples

The following example shows how to access a MySQL table via JDBC.

Suppose MySQL instance is available at `192.168.200.6:3306`. A table in MySQL is created:
```
use demodb;
create table myclass(
    id int(4) not null primary key,
    name varchar(20) not null,
    degree double(16,2)
);
```

Then some data is inserted into MySQL table:
```
insert into myclass values(1, 'tom', 90);
insert into myclass values(2, 'john', 94);
insert into myclass values(3, 'simon', 79);
```

The MySQL JDBC driver files (JAR) are copied to `/usr/lib/pxf` on all cluster nodes and a line is added to `/etc/pxf/conf/pxf-public.classpath`:
```
/usr/lib/pxf/mysql-connector-java-*.jar
```

After this, all PXF segments are restarted.

Then a table in HAWQ is created:
```
CREATE EXTERNAL TABLE myclass(
    id integer,
    name text,
    degree float8
)
LOCATION (
    'pxf://localhost:51200/demodb.myclass?PROFILE=JDBC&JDBC_DRIVER=com.mysql.jdbc.Driver&DB_URL=jdbc:mysql://192.168.200.6:3306/demodb&USER=root&PASS=root'
)
FORMAT 'CUSTOM' (
    FORMATTER='pxfwritable_import'
);
```

Finally, a query to a HAWQ external table is made:
```
SELECT * FROM myclass;
SELECT id, name FROM myclass WHERE id = 2;
```
