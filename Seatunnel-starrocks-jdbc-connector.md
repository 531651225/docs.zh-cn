# 使用 Seatunnel StarRocks JDBC Connector 读取和写入

本文介绍如何利用 Seatunnel StarRocks JDBC Connector 插件将 MySQL、Oracle 等数据库中的数据导入至 StarRocks\读取StarRocks数据库的数据写入其他外部数据源
Seatunnel 支持多种数据源连接器，您可以参考 [Seatunnel - Connectors supported](https://seatunnel.apache.org/docs/category/source-v2/) 了解详情。

目前Seatunnel StarRocks JDBC Connector在 [Seatunnel - 2.3.0-beta](https://github.com/apache/incubator-seatunnel/releases/tag/2.3.0-beta)开始支持，
确保您安装的Seatunnel为 2.3.0-beta 及 之上的版本。

## 安装 Apache SeaTunnel

参考下面文档，进行SeaTunnel实际安装部署
[Seatunnel - Quick start](http://seatunnel.incubator.apache.org/docs/2.3.0-beta/category/start)
[Seatunnel - Deployment](https://seatunnel.incubator.apache.org/docs/2.3.0-beta/deployment)

## StarRocks JDBC Connector使用前的环境准备

### 下载并安装 Mysql Jdbc驱动
> **说明**
>
> 由于 apache license 的原因, 您必须自己提供MySQL JDBC驱动程序，具体操作如下。
 [下载](https://repo1.maven.org/maven2/mysql/mysql-connector-java/8.0.16/mysql-connector-java-8.0.16.jar) mysql jdbc java驱动包，然后将驱动包 mysql-connector-java-8.0.16.jar 拷贝至 **$SEATNUNNEL_HOME/lib** 路径下。


## 创建配置文件

为数据同步作业创建配置文件。

以下示例模拟 从StarRocks数据源 读取数据写入至 StarRocks数据源 作业的配置文件。您可以根据实际使用场景修改相应参数。

```
env {
  execution.parallelism = 1
  job.mode = "BATCH"
}

source {
  Jdbc {
    driver = com.mysql.cj.jdbc.Driver
    url = "jdbc:mysql://fe1_ip:query_port,fe2_ip:query_port"
    user = root
    password = ""
    query = "select k1, k2, k3, k4 from `test`.`source_table`"
  }
}

sink {
  Jdbc {
    driver = com.mysql.cj.jdbc.Driver
    url = "jdbc:mysql://fe1_ip:query_port,fe2_ip:query_port"
    user = root
    password = ""
    query = "INSERT INTO `test`.`sink_table` (k1, k2, k3, k4) VALUES(?,?,?,?)"
  }
}
```

### StarRocks JDBC Source Connector配置项介绍

| 参数                           | 是否必填 | 数据类型   | 描述                                                                                                    |
|------------------------------|------|--------|-------------------------------------------------------------------------------------------------------|
| driver                       | 是    | String | 用于连接到远程数据源的jdbc类名, starrocks 使用 mysql jdbc driver,则为 com.mysql.cj.jdbc.Driver                         |
| url                          | 是    | String | FE 节点的连接地址，用于访问 FE 节点上的 MySQL 客户端。具体格式为 jdbc:mysql://\<FE 节点的 IP 地址\>:\<FE 的 query_port\>，端口号默认为9030。 |
| user                         | 是    | String | StarRocks 中的用户名称。需具备目标数据库表的读权限。用户权限说明，请参见[用户权限](../administration/User_privilege.md)。                 |
| password                     | 是    | String | StarRocks 的用户密码。                                                                                      |
| query                        | 是    | String | Query statement,从starrocks查询数据的SQL语句                                                                  |
| connection_check_timeout_sec | 否    | Int    | 检测connection是否有效，尝试与数据库作连接的超时时间,单位为秒。默认值为30。                                                          |
| partition_column             | 否    | String | 定义并行partition的列名，仅支持数据库中数字类型的列。                                                                       |
| partition_upper_bound        | 否    | String | 扫描的partition_column最大值，如果未设置，SeaTunnel将查询数据库获取最大值。                                                    |
| partition_lower_bound        | 否    | String | 扫描的partition_column最小值，如果未设置，SeaTunnel将查询数据库获取最小值。                                                    |
| partition_num                | 否    | Int    | partition分区数量，仅支持正整数。默认值是作业并行性                                                                        |

- **如何定义分区列**

通过定义分区列，可以实现数据源的并行读取。一般建议partition_column、partition_upper_bound、partition_lower_bound、partition_num 这几个配置项组合使用。
如果配置了partition_column，但是没有配置`partition_upper_bound\partition_lower_bound`，则在任务准备阶段SeaTunnel通过查询数据库获取分区列的`最大值\最小值`

下面举一个实例说明
配置项我们配置为`lower_bound = 1, upper_bound = 10, partition_num = 2，partition_column = "age", query = "select * from test"`
则starrocks source connector在真正执行阶段会将读取的数据拆分成两部分，进行并行读取
每一部分执行sql如下
```
split 1: select * from test  where  (age >= 1 and age < 6)
split 2: select * from test  where  (age >= 6 and age < 11)
```

更详细的配置参数，详见[seatunnel-jdbc-connector](https://seatunnel.incubator.apache.org/docs/connector/source/Jdbc)

### StarRocks JDBC Sink Connector配置项介绍

| 参数                           | 是否必填 | 数据类型   | 描述                                                                                                    |
|------------------------------|------|--------|-------------------------------------------------------------------------------------------------------|
| driver                       | 是    | String | 用于连接到远程数据源的jdbc类名, starrocks 使用 mysql jdbc driver,则为 com.mysql.cj.jdbc.Driver                         |
| url                          | 是    | String | FE 节点的连接地址，用于访问 FE 节点上的 MySQL 客户端。具体格式为 jdbc:mysql://\<FE 节点的 IP 地址\>:\<FE 的 query_port\>，端口号默认为9030。 |
| user                         | 是    | String | StarRocks 中的用户名称。需具备目标数据库表的读权限。用户权限说明，请参见[用户权限](../administration/User_privilege.md)。                 |
| password                     | 是    | String | StarRocks 的用户密码。                                                                                      |
| query                        | 是    | String | Query statement,执行jdbc写入的sql模板                                                                        |
| batch_size                   | 否    | String | 对于批写入，当缓冲记录的数量达到“batch_size”的数量或时间达到“bach_interval_ms”时`，数据将被刷新到数据库中                                  |
| batch_interval_ms            | 否    | String | 对于批写入，当缓冲记录的数量达到“batch_size”的数量或时间达到“bach_interval_ms”时`，数据将被刷新到数据库中                                  |
| connection_check_timeout_sec | 否    | String | flink-connector-starrocks 连接 StarRocks 的时间上限，单位为毫秒，默认值为1000。超过该时间上限，则将报错。                             |
| max_retries                  | 否    | String | jdbc（executeBatch）提交失败的重试次数，默认值为3。超过该数量上限，则将报错。                                                       |

更详细的配置参数，详见[seatunnel-jdbc-connector](https://seatunnel.incubator.apache.org/docs/connector/sink/Jdbc)

## 启动Seatunnel任务

完成配置文件设定后，您需要通过命令行启动数据同步任务。
目前Seatunnel 支持Spark 、Flink 、SeaTunnel Engine 等多种引擎执行方式，
以下示例以Spark引擎使用方式为例, 并基于前述步骤中的配置文件 **starrocks2starrocks.conf** 启动同步任务，并设置了`--deploy-mode`以`--master`的 Spark部署参数。

```shell
cd "apache-seatunnel-incubating-${version}"
./bin/start-seatunnel-spark-connector-v2.sh \
--master local[4] \
--deploy-mode client \
--config ./config/starrocks2starrocks.conf
```

## 数据同步过程中的数据转换
> 注意
>
> 在使用 Seatunnel 进行数据同步过程中时，如果有字段类型转换\数据过滤\多数据源Join等需求时，
> 可以通过Seatunnel提供的 `transform` 插件，进行数据转换处理。
Seatunnel 目前支持丰富的Transform插件，您可以参考 [Seatunnel - Transform supported](https://seatunnel.apache.org/docs/transform/common-options/) 了解详情。


