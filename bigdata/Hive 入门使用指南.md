# Hive 入门使用指南

Hive 是 Hadoop 生态中常用的数据仓库工具。它允许用户用接近 SQL 的 HiveQL 查询 HDFS 上的数据，并把数据库、表、字段、分区、存储位置等信息维护在元数据库中。对于刚接触大数据平台的人来说，理解 Hive 最重要的不是背语法，而是先建立三个概念：HiveQL 是入口，HDFS 是数据存储位置，Metastore 是表结构和位置的登记簿。

本文基于一套 CDH 6.3.2 测试环境整理，演示如何通过 Beeline 连接 HiveServer2、创建表、插入数据、查看表文件和查询元数据库。文中的主机名、用户名、数据库密码和内部路径均已脱敏或泛化，示例仅用于说明操作思路。

## 一、环境背景

测试环境为 CDH 6.3.2 部署的 Hadoop 集群，Hive 版本为 `2.1.1-cdh6.3.2`。常见组件关系如下：

- HDFS：保存 Hive 表的真实数据文件。
- HiveServer2：对外提供 JDBC 服务，Beeline 通过它提交 HiveQL。
- Beeline：Hive 官方推荐的命令行客户端。
- Metastore：保存 Hive 元数据，常见后端是 MySQL、PostgreSQL 等关系型数据库。
- YARN / MapReduce：Hive 在执行部分写入或计算任务时，会提交底层计算作业。

一个典型访问链路可以理解为：

```text
Beeline 客户端
  -> HiveServer2
  -> Hive Metastore 查询表结构和存储位置
  -> HDFS 读取或写入数据文件
  -> 必要时提交 YARN / MapReduce 作业
```

## 二、先看 HDFS 数据环境

Hive 的表数据通常存放在 HDFS 上。进入 Hive 操作前，可以先查看 HDFS 根目录，建立对集群目录结构的直观认识：

```bash
hdfs dfs -ls /
```

常见目录包括：

```text
/hbase
/solr
/tmp
/user
```

其中 Hive 默认仓库目录通常位于：

```text
/user/hive/warehouse
```

这个目录非常关键。默认数据库 `default` 下的内部表，如果没有额外指定 `LOCATION`，一般会在该目录下创建对应表目录。例如表名为 `test` 时，默认存储位置可能类似：

```text
/user/hive/warehouse/test
```

需要注意，HDFS 上存在 HBase、Solr、YARN、Hive 等多个组件目录时，不要随意修改其他组件目录权限或删除文件。Hive 入门练习应尽量限制在自己的数据库或测试表目录内。

## 三、使用 Beeline 连接 Hive

Hive 2.x 之后，推荐使用 Beeline 连接 HiveServer2。基础连接命令如下：

```bash
beeline -u jdbc:hive2://<hive-server-host>:10000 -n <hive-user>
```

例如：

```bash
beeline -u jdbc:hive2://hive-server.example.com:10000 -n hive_user
```

连接成功后，可以看到类似信息：

```text
Connected to: Apache Hive (version 2.1.1-cdh6.3.2)
Driver: Hive JDBC (version 2.1.1-cdh6.3.2)
Beeline version 2.1.1-cdh6.3.2 by Apache Hive
```

如果启动时出现 `SLF4J: Class path contains multiple SLF4J bindings` 之类提示，通常是客户端 classpath 中存在多个日志绑定。只要连接成功、SQL 可以正常执行，入门阶段可以先不处理这类告警。

## 四、基础 HiveQL 操作

连接成功后，可以先查看当前数据库：

```sql
select current_database();
```

输出通常为：

```text
default
```

查看当前库下的表：

```sql
show tables;
```

如果是新环境，可能没有任何业务表。可以创建一张最简单的测试表：

```sql
create table test (
  id int,
  name string
);
```

再次查看表列表：

```sql
show tables;
```

可以看到：

```text
test
```

插入几条测试数据：

```sql
insert into test values
  (1, 'zhangsan'),
  (2, 'lisi'),
  (3, 'wangwu');
```

查询数据：

```sql
select * from test;
```

预期结果：

```text
+----------+------------+
| test.id  | test.name  |
+----------+------------+
| 1        | zhangsan   |
| 2        | lisi       |
| 3        | wangwu     |
+----------+------------+
```

## 五、为什么 insert 可能会触发权限问题

在测试环境中，`create table` 可以成功，但执行 `insert into` 时曾遇到如下错误：

```text
org.apache.hadoop.security.AccessControlException:
Permission denied: user=<hive-user>, access=WRITE, inode="/user":hdfs:supergroup:drwxr-xr-x
```

这类问题容易让初学者困惑：为什么建表可以成功，插入数据却失败？

原因通常是 Hive 写入数据时需要为当前执行用户准备 HDFS 工作目录，常见路径是：

```text
/user/<hive-user>
```

如果该目录不存在，或者当前用户没有写权限，Hive 在提交 MapReduce 作业或创建临时文件时就会失败。

可以由 HDFS 管理员创建用户目录并授权：

```bash
sudo -u hdfs hdfs dfs -mkdir -p /user/<hive-user>
sudo -u hdfs hdfs dfs -chown <hive-user>:<hive-user> /user/<hive-user>
```

授权后检查：

```bash
hdfs dfs -ls /user
```

能看到类似目录即可：

```text
drwxr-xr-x   - <hive-user> <hive-user> 0 <date> /user/<hive-user>
```

生产环境中不建议直接使用 `root` 作为 Hive 连接用户。更好的方式是创建专门的业务用户，并配合 Ranger、Sentry 或 HDFS ACL 做权限管理。

## 六、查看表结构和存储位置

Hive 表创建后，可以用 `desc formatted` 查看完整元信息：

```sql
desc formatted test;
```

重点关注几个字段：

```text
Database:      default
Owner:         <hive-user>
Location:      hdfs://<nameservice>/user/hive/warehouse/test
Table Type:    MANAGED_TABLE
InputFormat:   org.apache.hadoop.mapred.TextInputFormat
OutputFormat:  org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat
SerDe Library: org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe
```

这些信息说明：

- `Database` 表示表所属数据库。
- `Owner` 表示表创建者。
- `Location` 表示表数据在 HDFS 上的实际目录。
- `Table Type` 为 `MANAGED_TABLE` 时表示内部表。
- `InputFormat`、`OutputFormat` 和 `SerDe` 决定 Hive 如何读写和解析文件。

内部表和外部表的区别尤其重要：

- 内部表由 Hive 管理生命周期，删除表时通常会删除表数据。
- 外部表只由 Hive 管理元数据，删除表时通常不会删除外部数据目录。

入门练习可以先使用内部表，但生产中导入已有数据时，外部表更常见，也更不容易误删原始数据。

## 七、查看 HDFS 上的表文件

通过 `desc formatted` 拿到表的 `Location` 后，可以直接查看 HDFS 文件：

```bash
hdfs dfs -ls /user/hive/warehouse/test
```

插入数据后，可能看到类似文件：

```text
-rwxrwxrwt   3 <hive-user> hive 27 <date> /user/hive/warehouse/test/000000_0
```

这说明 Hive 表并不是神秘的黑盒。对普通文本表来说，它的底层就是 HDFS 目录中的一个或多个数据文件。Hive 的价值在于给这些文件叠加了表结构、字段类型、分区信息和 SQL 查询能力。

## 八、理解 Hive 元数据库

Hive Metastore 负责保存元数据。以 MySQL 作为元数据库为例，管理员可以登录 MySQL 查看：

```bash
mysql -u <metastore-user> -p
```

选择 Hive 元数据库：

```sql
use hive;
show tables;
```

常见核心表如下：

| 表名 | 作用 |
| --- | --- |
| `DBS` | 保存 Hive 数据库信息，例如 `default` 和自定义库 |
| `TBLS` | 保存 Hive 表信息，例如表名、表类型、所属数据库 |
| `SDS` | 保存表的存储描述信息，例如 HDFS 路径、输入输出格式 |
| `COLUMNS_V2` | 保存字段名、字段类型、字段注释 |
| `PARTITIONS` | 保存分区表的分区记录 |
| `VERSION` | 保存 Metastore 版本信息 |

查询数据库信息：

```sql
select * from DBS;
```

查询某张表的元信息：

```sql
select TBL_ID, DB_ID, TBL_NAME, TBL_TYPE, CREATE_TIME
from TBLS
where TBL_NAME = 'test';
```

可能得到类似结果：

```text
+--------+-------+----------+---------------+-------------+
| TBL_ID | DB_ID | TBL_NAME | TBL_TYPE      | CREATE_TIME |
+--------+-------+----------+---------------+-------------+
|  15693 |     1 | test     | MANAGED_TABLE |  1782974201 |
+--------+-------+----------+---------------+-------------+
```

一般不建议直接修改 Metastore 表。日常运维和开发应通过 HiveQL 管理元数据，直接操作 Metastore 只适合只读排查或在明确方案下进行修复。

## 九、入门常用命令清单

Beeline 连接：

```bash
beeline -u jdbc:hive2://<hive-server-host>:10000 -n <hive-user>
```

HiveQL 基础操作：

```sql
show databases;
use default;
show tables;
select current_database();

create table test (
  id int,
  name string
);

insert into test values
  (1, 'zhangsan'),
  (2, 'lisi'),
  (3, 'wangwu');

select * from test;
desc formatted test;
drop table test;
```

HDFS 检查：

```bash
hdfs dfs -ls /
hdfs dfs -ls /user
hdfs dfs -ls /user/hive/warehouse
hdfs dfs -ls /user/hive/warehouse/test
```

用户目录授权：

```bash
sudo -u hdfs hdfs dfs -mkdir -p /user/<hive-user>
sudo -u hdfs hdfs dfs -chown <hive-user>:<hive-user> /user/<hive-user>
```

## 十、总结

Hive 入门可以按一条主线理解：客户端用 Beeline 连接 HiveServer2，HiveServer2 解析 HiveQL，并通过 Metastore 找到表结构和 HDFS 存储位置，最终读取或写入 HDFS 数据。

刚开始学习时，建议重点掌握四件事：会连接 Beeline，会创建和查询表，会从 `desc formatted` 找到 HDFS 位置，会判断权限问题是不是来自 HDFS 用户目录。把这几个环节串起来后，再继续学习分区表、外部表、文件格式、压缩、执行引擎和权限体系，会顺畅很多。
