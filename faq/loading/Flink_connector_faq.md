# Flink Connector常见问题

## flink-connector-jdbc_2.11sink到StarRocks时间落后8小时

**问题描述：**

Flink中localtimestap函数生成的时间，在Flink中时间正常，sink到StarRocks后发现时间落后8小时。已确认Flink所在服务器与StarRocks所在服务器时区均为Asia/ShangHai东八区。Flink版本为1.12，驱动为flink-connector-jdbc_2.11，需要如何处理？

**解决方案：**

可以在Flink sink表中配置时区参数'server-time-zone' = 'Asia/Shanghai'，或同时在jdbc url里添加&serverTimezone=Asia/Shanghai。示例如下：

```sql
CREATE TABLE sk (
    sid int,
    local_dtm TIMESTAMP,
    curr_dtm TIMESTAMP
)
WITH (
    'connector' = 'jdbc',
    'url' = 'jdbc:mysql://192.168.110.66:9030/sys_device?characterEncoding=utf-8&serverTimezone=Asia/Shanghai',
    'table-name' = 'sink',
    'driver' = 'com.mysql.jdbc.Driver',
    'username' = 'sr',
    'password' = 'sr123',
    'server-time-zone' = 'Asia/Shanghai'
);
```

## [Flink导入] 在starrocks集群上部署的kafka集群的数据可以导入，其他kafka集群无法导入

**问题描述：**

```SQL
failed to query wartermark offset, err: Local: Bad message format
```

**解决方案：**

kafka通信会用到hostname，需要在starrocks集群节点配置kafka主机名解析/etc/hosts。

## 没有查询时 BE 内存处于打满状态，且cpu也是打满状态

**问题描述:**

打满是什么原因引起的？

**解决方案:**

因为会定期收集统计信息，不是长期占用cpu，10g以内的内存使用完不会释放，be会自己管理，可以通过tc_use_memory_min限制大小。

tc_use_memory_min，默认等于10737418240。解释：TCmalloc 最小保留内存，只有超过这个值，StarRocks才将空闲内存返还给操作系统，可以去到BE 配置文件" be.conf"中去配置。

## Be申请的内存不会释放给操作系统

这是一种正常现象，因为数据库从操作系统获得的大块的内存分配，在分配的时候会多预留，释放时候会延后，为了重复利用，因为大块内存的分配的代价比较大. 建议测试环境验证时，对内存使用进行监控，在较长的周期内看内存是否能够完成释放。

## 关于下载flink connector后不生效问题

**问题描述：**

该包需要通过阿里云镜像地址来获取。

**解决方案:**

请确认 /etc/maven/settings.xml的 mirror 部分是否配置了全部通过阿里云镜像获取。

 如果是请修改为如下：

 <mirror>
    <id>aliyunmaven </id>
    <mirrorf>central</mirrorf>
    <name>阿里云公共仓库</name>
    <url>https：//maven.aliyun.com/repository/public</url>
</mirror>

## Flink-connector-StarRocks中sink.buffer-flush.interval-ms参数的含义

**问题描述:**

```plain text
+----------------------+--------------------------------------------------------------+
|         Option       | Required |  Default   | Type   |       Description           |
+-------------------------------------------------------------------------------------+
|  sink.buffer-flush.  |  NO      |   300000   | String | the flushing time interval, |
|  interval-ms         |          |            |        | range: [1000ms, 3600000ms]  |
+----------------------+--------------------------------------------------------------+
```

如果这个参数设置为15s，但是checkpoint interval=5mins,那么这个值还生效么？

**解决方案:**

三个阈值先达到其中的哪一个，那一个就先生效，是和checkpoint interval设置的值没关系的，checkpoint interval 对于 exactly once 才有效，at_least_once 用 interval-ms。
