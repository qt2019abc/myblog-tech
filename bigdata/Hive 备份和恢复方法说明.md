# Hive 备份和恢复方法说明

Hive 的备份恢复比普通文件备份更容易踩坑，因为一张 Hive 表通常由两部分共同组成：Metastore 中的元数据，以及 HDFS 上的实际数据文件。只备份其中一部分，都可能导致恢复后“表能看到但数据不对”，或者“数据文件还在但 Hive 查不到表”。

本文基于一次 CDH 6.3.2 环境下的测试记录，整理 Hive 表级、分区级和库级备份恢复方法。文中的主机名、用户名、nameservice、作业 ID、内部路径和账号信息均已脱敏或泛化，示例用于说明操作思路。

## 一、Hive 备份到底要保护什么

Hive 表不是一个单独文件，而是一组元数据和数据文件的组合。

```text
Hive Metastore 元数据
+
HDFS 上的实际数据文件
```

备份时通常要关注以下对象：

- database
- table
- partition
- schema
- serde
- storage format
- location
- managed table / external table
- view
- function
- HDFS data files

其中最关键的是一致性：Metastore 中登记的表结构、分区和 `LOCATION`，必须和 HDFS 上的数据目录匹配。

## 二、备份前检查清单

执行备份前，建议先确认以下事项：

- HiveServer2 是否可连接。
- Metastore 是否可访问。
- 目标 database 是否存在。
- 目标 table 是否存在。
- 表类型是 Managed Table、External Table、View 还是 ACID Table。
- 表是否为分区表。
- 表的 HDFS `LOCATION` 是哪里。
- 当前用户是否有 Hive `SELECT` 权限和 HDFS 读权限。
- 目标备份路径是否存在且可写。
- 跨集群恢复时，目标集群的库名、表名、用户组和 warehouse 路径是否一致。

可以通过以下 HiveQL 获取表信息：

```sql
SHOW CREATE TABLE default.test;
DESCRIBE FORMATTED default.test;
SHOW PARTITIONS default.test;
```

如果目标表不是分区表，`SHOW PARTITIONS` 会返回错误，提示该表不是分区表。这不是备份失败，而是说明后续不需要按分区粒度处理。

## 三、准备 HDFS 备份目录

备份目录建议单独规划，例如：

```text
/backup/hive/<database>/<table>/<backup_id>
```

示例：

```bash
sudo -u hdfs hdfs dfs -mkdir -p /backup/hive
sudo -u hdfs hdfs dfs -chown -R <backup-user>:<backup-group> /backup/hive
sudo -u hdfs hdfs dfs -chmod -R 750 /backup/hive
```

测试环境中为了快速验证，可能会把 `/backup` 设置为较宽松权限。但生产环境不建议使用 `777`，更推荐为备份服务账号授权最小可用权限。

检查目录：

```bash
hdfs dfs -ls /backup/hive
```

## 四、方式一：使用 EXPORT / IMPORT

`EXPORT TABLE` 和 `IMPORT TABLE` 是 Hive 内置的表级备份恢复方式，适合中小规模表、测试验证、单表迁移和部分分区恢复。

它的优点是简单直接：`EXPORT` 会同时导出表元数据和数据，`IMPORT` 会按照导出的元数据重建表或分区。

### 1. 表级备份

```sql
EXPORT TABLE default.test
TO 'hdfs:///backup/hive/default/test/backup_20260702_120000';
```

备份完成后，可以查看 HDFS 目录：

```bash
hdfs dfs -ls /backup/hive/default/test/backup_20260702_120000
```

正常情况下会看到类似结构：

```text
/backup/hive/default/test/backup_20260702_120000/_metadata
/backup/hive/default/test/backup_20260702_120000/data
```

其中：

- `_metadata` 保存表结构、存储格式等元数据信息。
- `data` 保存表对应的数据文件。

### 2. 表级恢复到新表

恢复到新表是最安全的验证方式，不会覆盖原表。

```sql
IMPORT TABLE default.test_restore
FROM 'hdfs:///backup/hive/default/test/backup_20260702_120000';
```

恢复后验证：

```sql
SHOW TABLES;
SELECT * FROM default.test_restore;
```

如果能看到 `test_restore`，并且数据行数、字段内容符合预期，说明这次表级恢复链路可用。

### 3. 分区级备份

分区表可以只导出指定分区：

```sql
EXPORT TABLE ods.order
PARTITION (dt='2026-07-02')
TO 'hdfs:///backup/hive/ods/order/dt=2026-07-02/backup_20260702_120000';
```

恢复分区时，建议先恢复到临时表或临时库中验证，再决定是否覆盖生产分区。

## 五、方式二：Metastore 元数据 + HDFS 数据备份

如果要做更可控的产品化备份，不能只依赖 `EXPORT / IMPORT`。更通用的方式是把元数据和数据文件分开备份。

基本流程如下：

```text
SHOW CREATE TABLE
  -> DESCRIBE FORMATTED
  -> SHOW PARTITIONS
  -> 记录表 LOCATION
  -> DistCp 复制 HDFS 数据目录
  -> 保存 DDL、分区信息、表属性和备份 manifest
```

### 1. 导出元数据信息

```sql
SHOW CREATE TABLE default.test;
DESCRIBE FORMATTED default.test;
SHOW PARTITIONS default.test;
```

`SHOW CREATE TABLE` 可以用于重建表结构，`DESCRIBE FORMATTED` 可以确认表类型、Owner、Location、InputFormat、OutputFormat 和 SerDe。

一个简化后的表结构示例：

```sql
CREATE TABLE `default.test`(
  `id` int,
  `name` string)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://<nameservice>/user/hive/warehouse/test';
```

### 2. 使用 DistCp 复制数据

`DistCp` 适合复制 HDFS 上的数据目录，尤其适合大表、跨集群或需要单独控制数据复制策略的场景。

```bash
hadoop distcp \
  hdfs://<source-nameservice>/user/hive/warehouse/test \
  hdfs://<backup-nameservice>/backup/hive/default/test/backup_20260702_data
```

复制完成后检查：

```bash
hdfs dfs -ls /backup/hive/default/test/
```

可以看到类似目录：

```text
/backup/hive/default/test/backup_20260702_data
```

需要注意，DistCp 只复制 HDFS 文件，不会自动保存表结构、分区、权限、统计信息和表属性。因此必须配套保存 DDL 和元数据清单。

## 六、DistCp 纯数据备份的恢复流程

如果备份集只有 HDFS 数据，没有 `EXPORT` 生成的 `_metadata`，恢复时需要先重建表结构，再把数据放回表的 `LOCATION`。

### 1. 创建恢复表

恢复表结构必须和原表保持一致。示例：

```sql
CREATE TABLE default.test_restore2 (
  id int,
  name string
)
ROW FORMAT SERDE
  'org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe'
STORED AS INPUTFORMAT
  'org.apache.hadoop.mapred.TextInputFormat'
OUTPUTFORMAT
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://<nameservice>/user/hive/warehouse/test_restore2';
```

### 2. 回拷数据

```bash
hadoop distcp -overwrite \
  hdfs://<backup-nameservice>/backup/hive/default/test/backup_20260702_data \
  hdfs://<target-nameservice>/user/hive/warehouse/test_restore2
```

`-overwrite` 表示目标目录已有同名文件时直接覆盖。生产恢复前应确认目标表目录是否为空，避免误覆盖仍需保留的数据。

### 3. 刷新和验证

对于非分区表，通常直接查询验证即可：

```sql
SELECT * FROM default.test_restore2;
```

对于分区表，数据目录回拷完成后，需要让 Hive 识别分区：

```sql
MSCK REPAIR TABLE default.test_restore2;
SHOW PARTITIONS default.test_restore2;
```

必要时重新计算统计信息：

```sql
ANALYZE TABLE default.test_restore2 COMPUTE STATISTICS;
```

## 七、方式三：库级批量备份

库级备份本质上是对库下所有对象进行编排，而不是一个单独命令解决所有问题。建议按表生成备份计划。

基本流程如下：

```text
列出 database 下所有 table
  -> 判断 managed / external / view / partitioned / acid
  -> 为每张表生成备份策略
  -> 表级并发备份
  -> 生成 database 级 manifest
```

不同对象的备份重点不同：

| 表类型 | 备份重点 |
| --- | --- |
| Managed Table | 元数据 + Hive 管理的数据目录 |
| External Table | 元数据 + 外部 `LOCATION` 数据 |
| Partitioned Table | 分区元数据 + 分区路径 |
| View | 只备份视图定义，不备份数据 |
| ACID Table | 关注事务一致性、base 文件和 delta 文件 |

库级备份尤其需要 manifest 文件，否则恢复时很难判断每张表来自哪个备份点、使用哪种恢复方式。

## 八、备份集 manifest 建议

每次备份建议保存一份结构化 manifest，用于记录备份对象、路径、表类型、分区、DDL 和时间。

示例：

```json
{
  "type": "hive",
  "database": "ods",
  "table": "order",
  "backup_mode": "export",
  "backup_path": "hdfs:///backup/hive/ods/order/backup_20260702_120000",
  "table_type": "MANAGED_TABLE",
  "is_partitioned": true,
  "partitions": [
    "dt=2026-07-01",
    "dt=2026-07-02"
  ],
  "ddl": "CREATE TABLE ...",
  "location": "hdfs:///warehouse/tablespace/managed/hive/ods.db/order",
  "created_at": "2026-07-02T12:00:00"
}
```

如果备份系统后续要做自动化恢复，manifest 是非常关键的输入。它可以帮助恢复程序判断应该走 `IMPORT`、重建表加 DistCp，还是只恢复视图定义。

## 九、恢复策略选择

### 1. 恢复到新表

这是最推荐的验证方式，适合演练和数据核对。

```sql
IMPORT TABLE ods.order_restore
FROM 'hdfs:///backup/hive/ods/order/backup_20260702_120000';
```

恢复后校验行数、分区、字段和抽样数据，再决定是否切换业务读取目标。

### 2. 覆盖恢复原表

覆盖恢复风险较高，建议按以下流程执行：

```text
检查目标表是否存在
  -> 备份当前目标表 DDL
  -> 重命名旧表或保留快照
  -> DROP / TRUNCATE 目标表
  -> IMPORT 或重建表
  -> 恢复权限
  -> 修复分区
  -> 校验数据量和抽样数据
```

示例：

```sql
ALTER TABLE ods.order RENAME TO ods.order_before_restore_20260702;

IMPORT TABLE ods.order
FROM 'hdfs:///backup/hive/ods/order/backup_20260702_120000';
```

### 3. 分区级恢复

分区级恢复适合只恢复某一天或某小时数据。

```sql
ALTER TABLE ods.order DROP PARTITION (dt='2026-07-02');

ALTER TABLE ods.order ADD PARTITION (dt='2026-07-02')
LOCATION 'hdfs:///warehouse/tablespace/managed/hive/ods.db/order/dt=2026-07-02';
```

恢复后执行：

```sql
MSCK REPAIR TABLE ods.order;
ANALYZE TABLE ods.order COMPUTE STATISTICS;
```

## 十、恢复校验

恢复完成后不要只看命令是否成功，还要做数据和元数据校验。

建议至少检查：

- `SHOW TABLES` 能看到恢复后的表。
- `DESCRIBE FORMATTED` 中的 `LOCATION` 符合预期。
- `SELECT COUNT(*)` 与备份前记录一致。
- 抽样查询结果符合预期。
- 分区表的 `SHOW PARTITIONS` 结果完整。
- HDFS 表目录下文件数量和大小合理。
- 业务用户仍有正确的 Hive 和 HDFS 权限。

常用命令：

```sql
SHOW TABLES;
DESCRIBE FORMATTED default.test_restore;
SELECT COUNT(*) FROM default.test_restore;
SELECT * FROM default.test_restore LIMIT 10;
```

HDFS 检查：

```bash
hdfs dfs -ls /user/hive/warehouse/test_restore
hdfs dfs -du -h /user/hive/warehouse/test_restore
```

## 十一、注意事项

- External Table 恢复时，必须确认原始 `LOCATION` 是否仍然有效。
- Managed Table 恢复时，要注意目标 Hive 的默认 warehouse 路径是否和源环境一致。
- View 只需要恢复定义，不需要恢复数据文件。
- 分区表恢复后，应校验分区是否完整，并根据需要执行 `MSCK REPAIR TABLE`。
- ACID 表恢复要额外关注 base/delta 文件和事务一致性，不能只按普通目录复制来理解。
- 跨集群恢复要处理数据库名、表名、HDFS 路径、用户、用户组和权限差异。
- 生产环境不要把备份目录设置成全员可写，避免备份集被误删或覆盖。

## 十二、总结

Hive 备份恢复的核心是同时照顾元数据和 HDFS 数据。`EXPORT / IMPORT` 简单，适合表级和分区级恢复；`SHOW CREATE TABLE + DESCRIBE FORMATTED + DistCp` 更可控，适合大表、跨集群和产品化备份；库级备份则需要按表编排，并用 manifest 记录每个对象的恢复方式。

实际落地时，建议先从“恢复到新表”的演练开始，确认备份集可用、数据可查、权限可用，再设计覆盖恢复和分区级恢复流程。备份没有经过恢复验证，就不能算真正可用。
