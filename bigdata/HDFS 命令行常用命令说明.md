# HDFS 命令行常用命令说明

HDFS 是 Hadoop 生态中的分布式文件系统，常用于保存 Hive、HBase、Spark、MapReduce 等组件的数据。对日常运维和数据开发来说，`hdfs dfs` 是最常用的命令行入口，可以完成目录管理、文件上传下载、内容查看、删除、校验等操作。

本文基于一次 HDFS 命令行实践记录整理，并补充常见背景知识和使用注意事项。文中的主机名、用户名、nameservice、HDFS 内部路径和时间信息均已脱敏或泛化。

## 一、HDFS 命令行的基本形态

HDFS 常用命令一般以 `hdfs dfs` 开头：

```bash
hdfs dfs <sub-command> <args>
```

例如查看根目录：

```bash
hdfs dfs -ls /
```

它和 Linux 本地文件命令很像，但操作对象是 HDFS，而不是当前机器的本地磁盘。比如：

- `hdfs dfs -ls` 查看 HDFS 目录。
- `hdfs dfs -mkdir` 创建 HDFS 目录。
- `hdfs dfs -put` 上传本地数据到 HDFS。
- `hdfs dfs -get` 下载 HDFS 数据到本地。
- `hdfs dfs -cat` 输出 HDFS 文件内容。
- `hdfs dfs -rm` 删除 HDFS 文件。

如果集群开启 Kerberos，执行这些命令前通常还需要先 `kinit` 获取票据。

## 二、创建目录

创建一个测试目录：

```bash
hdfs dfs -mkdir -p /tmp/hdfsbasic
```

这里的 `-p` 和 Linux `mkdir -p` 类似：如果父目录不存在，会自动创建；如果目录已经存在，也不会因为重复创建直接失败。

查看目录：

```bash
hdfs dfs -ls /tmp
```

建议测试和临时数据放在 `/tmp` 或明确规划的测试目录中，不要直接在 Hive、HBase 等组件目录下随意创建文件。

## 三、上传文件

可以把本地文件上传到 HDFS：

```bash
hdfs dfs -put ./1.log /tmp/hdfsbasic/1.log
```

也可以通过标准输入直接写入 HDFS 文件：

```bash
printf 'log-demo\n' | hdfs dfs -put - /tmp/hdfsbasic/1.log
```

这里的 `-` 表示从标准输入读取内容。这个方法适合快速创建测试文件，不需要先在本地落盘。

上传后查看：

```bash
hdfs dfs -ls /tmp/hdfsbasic
```

输出类似：

```text
-rw-r--r--   3 <hdfs-user> <hdfs-group> 9 <date> /tmp/hdfsbasic/1.log
```

字段含义大致如下：

- 第一列是权限。
- 第二列是副本数。
- 后面依次是 owner、group、文件大小、时间和路径。

## 四、查看文件内容

查看小文件内容：

```bash
hdfs dfs -cat /tmp/hdfsbasic/1.log
```

输出：

```text
log-demo
```

对于大文件，不建议直接 `cat` 到终端。可以结合 `head`、`tail` 或抽样工具：

```bash
hdfs dfs -cat /path/bigfile | head
hdfs dfs -tail /path/bigfile
```

注意：`hdfs dfs -cat` 会把文件内容从 DataNode 读取到客户端机器。如果文件很大，会消耗 HDFS I/O、网络带宽和客户端资源。

## 五、下载文件到本地

把 HDFS 文件下载到当前本地目录：

```bash
hdfs dfs -get /tmp/hdfsbasic/1.log .
```

查看本地文件：

```bash
cat 1.log
```

如果目标本地文件已经存在，`-get` 可能会失败。可以先换一个目录，或者确认后删除本地旧文件。

常见替代命令：

```bash
hdfs dfs -copyToLocal /tmp/hdfsbasic/1.log .
```

`-get` 和 `-copyToLocal` 在日常使用中基本可以理解为同类操作。

## 六、删除文件和目录

删除单个文件：

```bash
hdfs dfs -rm /tmp/hdfsbasic/1.log
```

递归删除目录：

```bash
hdfs dfs -rm -r /tmp/hdfsbasic
```

如果集群启用了 Trash，删除时可能会看到类似提示：

```text
Moved: 'hdfs://<nameservice>/tmp/hdfsbasic/2.log'
to trash at: hdfs://<nameservice>/user/<hdfs-user>/.Trash/Current/tmp/hdfsbasic/2.log
```

这说明文件没有立刻物理删除，而是被移动到当前用户的回收站目录。是否启用 Trash、保留多久，取决于集群配置。

如果明确要跳过回收站，需要非常谨慎：

```bash
hdfs dfs -rm -skipTrash /path/file
```

生产环境中不建议随意使用 `-skipTrash`。

## 七、HDFS 文件不能直接原地修改

HDFS 更适合一次写入、多次读取的场景，不适合像本地文件系统一样频繁原地修改文件。

如果想修改一个已有文件，常见做法是：

1. 写入一个新文件。
2. 校验新文件内容。
3. 删除或归档旧文件。
4. 将新文件移动到目标路径。

如果直接覆盖已存在文件，`hdfs dfs -put` 可能会报错：

```text
put: `/tmp/hdfsbasic/2.log': File exists
```

可以先删除旧文件，再上传新文件：

```bash
hdfs dfs -rm /tmp/hdfsbasic/2.log
printf 'log2-demo-modify\n' | hdfs dfs -put - /tmp/hdfsbasic/2.log
```

也可以在确认风险后使用覆盖参数：

```bash
hdfs dfs -put -f ./2.log /tmp/hdfsbasic/2.log
```

但在生产路径中，建议优先使用“新路径写入 + 校验 + rename 切换”的方式，减少半写入或误覆盖风险。

## 八、文件摘要和一致性校验

日常排查中，经常需要比较两个 HDFS 文件是否一致。常见方法有两类：HDFS 自带 checksum，以及标准哈希算法。

### 1. 使用 hdfs dfs -checksum

```bash
hdfs dfs -checksum /path/file1
hdfs dfs -checksum /path/file2
```

这个命令通常比 `cat | sha256sum` 更适合同一 HDFS 集群内的快速比较，因为它利用 HDFS 已有的块校验信息，不需要把完整文件内容都传输到客户端。

不过要注意，`hdfs dfs -checksum` 的结果可能受以下因素影响：

- 块大小。
- 分块方式。
- `bytes-per-checksum`。
- 文件创建时的底层参数。
- 不同集群之间的配置差异。

因此，checksum 不同不一定百分百代表内容不同，尤其是在跨集群或写入配置不同的情况下；checksum 相同通常可以用于同配置环境下的快速判断。

### 2. 使用 sha256sum

如果需要和本地文件使用统一算法比较，或需要跨集群做内容级校验，可以使用：

```bash
hdfs dfs -cat /path/bigfile | sha256sum
```

这是流式处理，不会把整个文件一次性加载到内存，因此大文件也能运行。但它必须完整读取一次 HDFS 文件，并把数据传输到客户端机器，再由客户端计算摘要。

主要成本包括：

- 完整读取一次文件，消耗 HDFS I/O。
- 数据从 DataNode 传输到客户端，消耗网络带宽。
- SHA-256 计算占用客户端 CPU。
- 多个超大文件并发计算可能影响集群和客户端。

## 九、如何选择校验方式

可以按场景选择：

| 场景 | 推荐方式 |
| --- | --- |
| 同集群、相同写入配置下快速比较 | `hdfs dfs -checksum` |
| 跨集群比较文件内容 | `hdfs dfs -cat ... \| sha256sum` |
| HDFS 文件与本地文件比较 | 对 HDFS 和本地文件分别计算 SHA-256 |
| 超大文件频繁比对 | 上传时计算并保存 SHA-256 元数据 |

如果数据链路对准确性要求高，建议在文件生成或上传阶段就保存业务侧哈希值，而不是每次排查时重新读取全量文件。

## 十、常用命令清单

目录操作：

```bash
hdfs dfs -ls /
hdfs dfs -ls /tmp
hdfs dfs -mkdir -p /tmp/hdfsbasic
```

上传和下载：

```bash
hdfs dfs -put ./1.log /tmp/hdfsbasic/1.log
printf 'log-demo\n' | hdfs dfs -put - /tmp/hdfsbasic/1.log
hdfs dfs -get /tmp/hdfsbasic/1.log .
```

查看内容：

```bash
hdfs dfs -cat /tmp/hdfsbasic/1.log
hdfs dfs -tail /tmp/hdfsbasic/1.log
```

删除：

```bash
hdfs dfs -rm /tmp/hdfsbasic/1.log
hdfs dfs -rm -r /tmp/hdfsbasic
```

校验：

```bash
hdfs dfs -checksum /tmp/hdfsbasic/1.log
hdfs dfs -cat /tmp/hdfsbasic/1.log | sha256sum
```

## 十一、使用建议

- 测试命令尽量放在 `/tmp` 或独立测试目录。
- 删除生产目录前先 `hdfs dfs -ls` 确认路径。
- 大文件不要随意 `cat` 到终端。
- 批量删除前先小范围验证匹配规则。
- 对关键数据保留 checksum 或业务哈希。
- 生产环境谨慎使用 `-skipTrash` 和覆盖写入。
- 遇到权限问题时，同时检查 HDFS owner、group、ACL 和当前认证用户。

## 十二、总结

HDFS 命令行的学习可以从最常用的几个动作开始：创建目录、上传文件、查看文件、下载文件、删除文件和校验文件。理解这些命令后，再进一步学习权限、ACL、Trash、配额、快照和跨集群复制，会更容易把 HDFS 当作一个可运维的分布式文件系统来使用。

在实际工作中，最容易出问题的往往不是命令本身，而是路径、权限、文件大小和删除范围。多做一次 `-ls`，少做一次误删，HDFS 会温和很多。
