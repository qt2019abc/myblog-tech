# HBase 备份和恢复方法说明

HBase 是构建在 HDFS 之上的分布式列式数据库，适合低延迟随机读写场景。它的数据并不是普通表格文件，而是由 namespace、table、column family、region、HFile、WAL 以及 HBase 元数据共同组成。因此，HBase 备份恢复不能简单理解为复制 `/hbase/data` 目录，更推荐使用 HBase 原生 Snapshot 机制。

本文基于一次 CDH 6.3.2 / HBase 2.1.0 环境下的测试记录，整理 HBase 基础操作、Snapshot 备份、ExportSnapshot 导出、同集群恢复、异机恢复和恢复校验流程。文中的主机名、用户、nameservice、作业 ID、时间戳和内部路径均已脱敏或泛化。

## 一、HBase 备份要保护什么

HBase 表由多层对象组成：

- namespace：命名空间，用于组织表。
- table：业务表，例如 `testspace:user`。
- column family：列族，例如 `info`。
- row key：行键，是 HBase 数据分布和查询的核心。
- region：表按 row key 范围切分后的分片。
- HFile：最终落在 HDFS 上的数据文件。
- WAL：写前日志，用于故障恢复。
- snapshot manifest：快照元信息，记录快照涉及的表结构和 HFile 引用。

所以 HBase 备份的核心目标是：在某个时间点获得表结构和数据文件引用的一致视图，并能在同集群或目标集群中重新恢复为可用表。

## 二、测试表准备

进入 HBase Shell：

```bash
hbase shell
```

查看命名空间：

```ruby
list_namespace
```

创建测试命名空间：

```ruby
create_namespace 'testspace'
```

创建测试表：

```ruby
create 'testspace:user', {NAME => 'info'}
```

查看表结构：

```ruby
desc 'testspace:user'
```

写入测试数据：

```ruby
put 'testspace:user', 'row001', 'info:name', 'zhangsan'
put 'testspace:user', 'row002', 'info:name', 'wangwu'
```

查询数据：

```ruby
get 'testspace:user', 'row001'
scan 'testspace:user'
```

预期可以看到两行数据：

```text
row001  column=info:name, value=zhangsan
row002  column=info:name, value=wangwu
```

这张表后续会作为备份和恢复演示对象。

## 三、为什么推荐 Snapshot

HBase Snapshot 是 HBase 原生的一致性备份机制。它不会立即复制整张表的数据，而是记录某个时间点的表结构和 HFile 引用，因此创建速度通常很快。

Snapshot 的优势包括：

- 秒级创建快照，对在线业务影响较小。
- 保存表结构、列族配置、region 信息和数据文件引用。
- 支持通过 `clone_snapshot` 快速恢复为新表。
- 支持通过 `restore_snapshot` 恢复原表。
- 支持通过 `ExportSnapshot` 导出到其他 HDFS 路径或异机集群。

相比直接复制 HDFS 目录，Snapshot 更能保证 HBase 层面的元数据和数据一致性。

## 四、备份前检查清单

执行 HBase 备份前，建议检查：

- HBase Shell 是否可用。
- ZooKeeper 和 HBase Master 是否可连接。
- namespace 是否存在。
- 表是否存在。
- 表处于 `ENABLED` 还是 `DISABLED` 状态。
- 是否存在正在运行的 snapshot 或 export 任务。
- 备份目标 HDFS 路径是否可写。
- 是否需要 Kerberos `kinit`。
- 表大小和 region 数量是否适合当前备份窗口。
- 是否启用 MOB、TTL、压缩、BloomFilter 等列族特性。

常用命令：

```ruby
status
list_namespace
list
list_namespace_tables 'testspace'
describe 'testspace:user'
list_snapshots
```

## 五、创建表快照

在 HBase Shell 中执行：

```ruby
snapshot 'testspace:user', 'snap_testspace_user_20260702'
```

查看快照：

```ruby
list_snapshots
```

如果只是同集群快速恢复，快照保留在当前 HBase 集群内即可。若要做跨集群备份、离线归档或灾备演练，还需要把快照导出到备份 HDFS 路径。

## 六、导出快照到备份目录

在服务器终端执行：

```bash
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot snap_testspace_user_20260702 \
  -copy-to /backup/hbase/testspace/user/snap_testspace_user_20260702
```

如果是导出到远端 HDFS，可以使用完整 HDFS 地址：

```bash
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot snap_testspace_user_20260702 \
  -copy-to hdfs://<backup-nameservice>/backup/hbase/testspace/user/snap_testspace_user_20260702
```

导出完成后检查目录：

```bash
hdfs dfs -ls /backup/hbase/testspace/user/snap_testspace_user_20260702
```

通常会看到类似结构：

```text
/backup/hbase/testspace/user/snap_testspace_user_20260702/.hbase-snapshot
/backup/hbase/testspace/user/snap_testspace_user_20260702/archive
```

其中 `.hbase-snapshot` 保存快照元信息，`archive` 保存导出的 HFile 相关内容。

## 七、同集群恢复

同集群恢复分为两类：恢复到原表，或克隆为新表。

### 1. 克隆为新表

这是最推荐的恢复验证方式，因为不会破坏原表。

```ruby
clone_snapshot 'snap_testspace_user_20260702', 'testspace:user_restore'
```

验证表和数据：

```ruby
list_namespace_tables 'testspace'
scan 'testspace:user_restore'
```

如果能看到恢复表和原始数据，说明快照可用。

### 2. 恢复到原表

恢复到原表适合灾难恢复，但风险更高。它会把原表恢复到快照时间点，快照之后写入的数据可能被覆盖。

```ruby
disable 'testspace:user'
restore_snapshot 'snap_testspace_user_20260702'
enable 'testspace:user'
```

执行前建议先确认：

- 业务已经停止写入或允许回滚。
- 当前表已经额外备份或保留二次快照。
- 操作人明确知道快照时间点。
- 已获得变更审批或人工二次确认。

## 八、异机或跨集群恢复

跨集群恢复的思路是：先把导出的 snapshot 放到目标集群 HBase 可识别的位置，再在目标集群执行 `clone_snapshot` 或 `restore_snapshot`。

### 1. 把备份快照导入目标集群

在目标集群执行：

```bash
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot snap_testspace_user_20260702 \
  -copy-from hdfs://<backup-nameservice>/backup/hbase/testspace/user/snap_testspace_user_20260702 \
  -copy-to hdfs://<target-nameservice>/hbase
```

如果备份目录已经在目标集群本地 HDFS 上，也可以简化为：

```bash
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot snap_testspace_user_20260702 \
  -copy-from /backup/hbase/testspace/user/snap_testspace_user_20260702 \
  -copy-to /hbase
```

### 2. 在目标集群克隆为新表

进入目标集群 HBase Shell：

```bash
hbase shell
```

确认命名空间存在：

```ruby
list_namespace
create_namespace 'testspace'
```

如果命名空间已经存在，重复创建会报错，可以忽略或改为先判断。

克隆快照：

```ruby
clone_snapshot 'snap_testspace_user_20260702', 'testspace:user_restore'
```

验证数据：

```ruby
scan 'testspace:user_restore'
```

## 九、在线快照与离线快照

大多数场景可以使用在线快照：

```ruby
snapshot 'testspace:user', 'snap_testspace_user_20260702'
```

在线快照通常不需要长时间停表，对读写影响较小。

如果环境较老、业务对一致性要求极高，或明确希望在表停止写入后创建快照，可以使用离线方式：

```ruby
disable 'testspace:user'
snapshot 'testspace:user', 'snap_testspace_user_20260702'
enable 'testspace:user'
```

离线快照会影响表可用性，应安排在维护窗口执行。

## 十、辅助命令

查看所有快照：

```ruby
list_snapshots
```

删除无用快照：

```ruby
delete_snapshot 'snap_testspace_user_20260702'
```

查看命名空间下的表：

```ruby
list_namespace_tables 'testspace'
```

删除测试表：

```ruby
disable 'testspace:user_restore'
drop 'testspace:user_restore'
```

## 十一、备份集 manifest 建议

如果要把 HBase 备份做成长期可维护的流程，建议为每个备份集保存 manifest。

示例：

```json
{
  "type": "hbase",
  "namespace": "testspace",
  "table": "user",
  "full_table_name": "testspace:user",
  "snapshot_name": "snap_testspace_user_20260702",
  "backup_path": "hdfs:///backup/hbase/testspace/user/snap_testspace_user_20260702",
  "backup_mode": "snapshot_export",
  "table_schema": {
    "column_families": ["info"],
    "ttl": "FOREVER",
    "compression": "NONE"
  },
  "created_at": "2026-07-02T12:00:00"
}
```

manifest 至少应记录：

- namespace 和表名。
- snapshot 名称。
- 备份路径。
- 列族配置。
- TTL、压缩、BloomFilter 等关键属性。
- 备份时间和操作者。
- 源集群和目标集群信息。

## 十二、恢复校验

恢复完成后，建议执行以下检查：

```ruby
list_namespace_tables 'testspace'
describe 'testspace:user_restore'
count 'testspace:user_restore'
scan 'testspace:user_restore', {LIMIT => 10}
```

同时检查 HBase 状态：

```ruby
status
```

如果是跨集群恢复，还应确认：

- 目标命名空间和表名符合预期。
- 列族配置和源表一致。
- 抽样数据正确。
- Region 能正常上线。
- 业务用户具备访问权限。
- 不再需要的临时快照已清理。

## 十三、注意事项

- 不建议直接复制 HBase 底层 `/hbase/data` 目录作为主要备份方式。
- 恢复原表前要明确快照时间点，避免覆盖快照之后的新数据。
- `clone_snapshot` 更适合演练、审计和数据比对，风险低于直接覆盖原表。
- 跨集群恢复要关注 HBase 版本、HDFS 路径、权限、Kerberos、压缩编码和集群配置差异。
- 大表导出快照会触发 MapReduce 作业，应关注 YARN 资源和导出耗时。
- 删除快照前要确认没有克隆表或备份流程仍依赖该快照。

## 十四、总结

HBase 备份恢复的推荐主线是：先创建 Snapshot，必要时使用 ExportSnapshot 导出到备份 HDFS，再通过 clone_snapshot 或 restore_snapshot 恢复。日常演练优先选择克隆为新表，确认数据和表结构无误后再考虑覆盖恢复原表。

对于生产环境，真正可靠的 HBase 备份不只是几条命令，还应包含备份前检查、备份集 manifest、跨集群导出、恢复演练、数据校验和快照生命周期管理。只有完成过恢复验证的备份，才算是可用备份。
