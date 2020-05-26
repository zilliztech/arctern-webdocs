# JDBC 连接

Arctern-Spark 可借助 Spark 的 JDBC 连接功能，完成数据从数据库的导入和导出。以下例子将展示如何利用 JDBC 从 PostGIS 中导入数据，更多详细信息请查看 [Spark 官方文档](https://spark.apache.org/docs/latest/sql-data-sources-load-save-functions.html)。

## PostGIS 配置信息

假设 PostGIS 的相关配置如下：

| 配置 | 值 |
|:-----------|:----------|
|IP address |  172.17.0.2|
|port | 5432|
|database name | test|
|user name | acterner|
|password | acterner|

使用如下命令测试 postgis 连接：

```bash
$ psql test -h 172.17.0.2  -p 5432 -U arcterner
```

## JDBC 数据导入示例

在提交 Spark 任务时，需要指定 JDBC 驱动。请从 [PostgreSQL 官网](https://jdbc.postgresql.org/download.html)下载其最新的 JDBC 驱动。以下示例使用的的驱动为 `postgresql-42.2.11.jar`。

以下命令为 Arctern-Spark 通过 JDBC 从 PostGIS 导入数据的示例：

```bash
$ ./bin/spark-submit  --driver-class-path ~/postgresql-42.2.11.jar --jars ~/postgresql-42.2.11.jar ~/query_postgis.py 
```

其中，`query_postgis.py` 的具体代码如下：

```python
from pyspark.sql import SparkSession
from arctern_pyspark import register_funcs
if __name__ == "__main__":

    # 创建 SparkSession 并对其进行配置
    spark = SparkSession \
        .builder \
        .appName("polygon test") \
        .getOrCreate()
    spark.conf.set("spark.sql.execution.arrow.pyspark.enabled", "true")

    # 注册 Arctern-Spark 提供的函数
    register_funcs(spark)
    
    # 数据导入
    spark.read.format("jdbc") \
              .option("url", "jdbc:postgresql://172.17.0.2:5432/test?user=arcterner&password=arcterner") \
              .option("query", "select st_astext(geos) as geos from simple") \
              .load() \
              .createOrReplaceTempView("simple")

    # 对处理数据并打印结果
    spark.sql("select ST_IsSimple(ST_GeomFromText(geos)) from simple").show(20,0)

    # 数据导出
    spark.write.format("jdbc") \
               .option("url", "jdbc:postgresql://172.17.0.2:5432/test?user=arcterner&password=arcterner") \
               .option("query", "select st_astext(geos) as geos from simple") \
               .load() \
               .save()
    spark.stop()
```
上述代码的执行结果如下：

```python
+----------------------------------+                                            
|ST_IsSimple(ST_GeomFromText(geos))|
+----------------------------------+
|true                              |
|true                              |
|false                             |
|true                              |
+----------------------------------+
```