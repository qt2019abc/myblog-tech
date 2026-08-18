# Linux 使用 iSCSI 挂载远端 LUN 并恢复数据

在备份恢复、虚拟机磁盘挂载、远程块存储访问和文件级恢复场景中，经常需要把存储侧导出的 LUN 通过 iSCSI 挂载到 Linux 服务器上。对操作系统来说，远端 LUN 登录成功后会表现为一块普通的本地块设备，后续可以继续识别分区、LVM 和文件系统，并以只读方式挂载读取数据。

本文基于一次 openEuler 环境下的 iSCSI 恢复验证记录整理，介绍从登录 Target、定位 IQN 对应磁盘、识别 LVM、只读挂载 XFS 文件系统，到安全卸载和退出会话的完整流程。文中的 IP、IQN、主机名、用户名、文件名和可能涉及凭证的信息均已脱敏或泛化。

## 一、适用场景

iSCSI 挂载远端 LUN 常见于以下场景：

- 备份系统导出恢复卷，需要在 Linux 上验证数据。
- 虚拟机磁盘被映射为块设备，需要做文件级恢复。
- 存储快照或克隆卷需要临时挂载检查。
- 跨主机迁移时，需要读取旧系统盘中的配置或业务文件。
- 灾难恢复演练中，需要验证远端块存储是否可读。

这类操作的核心原则是：未知恢复盘优先只读挂载，先确认结构和数据，再决定是否执行写入类修复操作。

## 二、核心概念

### 1. iSCSI

iSCSI 是 Internet Small Computer Systems Interface 的缩写，它通过 TCP/IP 网络传输 SCSI 命令。Linux Initiator 登录远端 Target 后，远端 LUN 会在本机呈现为块设备，例如：

```text
/dev/sdb
```

上层仍然按标准 Linux 存储栈处理：

```text
iSCSI LUN
  -> SCSI Block Device
  -> Partition
  -> LVM
  -> Filesystem
  -> mount
```

### 2. Initiator、Target、Portal 和 LUN

Initiator 是主动连接 iSCSI 存储的一端，也就是 Linux 客户端。常用工具是：

```bash
iscsiadm
```

Target 是提供块存储服务的一端。Target 通常用 IQN 标识，例如：

```text
iqn.2026-08.example:restore-lun-001
```

Portal 是 Target 的网络入口，通常是：

```text
<target-ip>:3260
```

其中 `3260` 是 iSCSI 常用端口。LUN 是 Target 对外暴露的逻辑块设备，一个 Target 下可以有一个或多个 LUN。

### 3. Node 与 Session

`iscsiadm` 中有两个容易混淆的概念：

- Node：保存在本机的 Target 配置记录。
- Session：当前已经建立的 iSCSI 登录连接。

查看 Node：

```bash
iscsiadm -m node
```

查看 Session：

```bash
iscsiadm -m session
```

执行 `logout` 后，Session 会断开，但 Node 记录通常仍然保留，方便下次重新登录。

## 三、安装 iSCSI Initiator 工具

openEuler、RHEL、CentOS 类系统可以安装：

```bash
dnf install -y open-iscsi
```

Ubuntu、Debian 类系统通常使用：

```bash
apt-get install -y open-iscsi
```

确认工具可用：

```bash
iscsiadm --version
```

启动服务：

```bash
systemctl enable --now iscsid
```

查看本机 Initiator IQN：

```bash
cat /etc/iscsi/initiatorname.iscsi
```

如果存储侧做了访问控制，需要把这个 Initiator IQN 加入 Target 的允许列表。

## 四、发现并登录 Target

假设存储侧提供的信息如下：

```text
Target Portal: <target-ip>:3260
Target IQN: iqn.2026-08.example:restore-lun-001
```

可以先发现 Target：

```bash
iscsiadm -m discovery \
  -t sendtargets \
  -p <target-ip>:3260
```

登录指定 Target：

```bash
iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --login
```

如果环境启用了 CHAP 认证，不要把密码直接写入文档或脚本仓库。可以通过运维密钥系统下发，或在受控终端中配置：

```bash
iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --op update \
  -n node.session.auth.authmethod \
  -v CHAP

iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --op update \
  -n node.session.auth.username \
  -v <chap-username>

iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --op update \
  -n node.session.auth.password \
  -v '<chap-password>'
```

登录后查看会话：

```bash
iscsiadm -m session
```

输出类似：

```text
tcp: [1] <target-ip>:3260,1 iqn.2026-08.example:restore-lun-001 (non-flash)
```

说明 iSCSI Session 已建立。

## 五、确认 IQN 对应哪个本地磁盘

登录后，Linux 会通过 SCSI 子系统把远端 LUN 注册为本地块设备。不要直接假设它一定是 `/dev/sdb`，更推荐通过 `/dev/disk/by-path/` 定位。

```bash
ls -l /dev/disk/by-path/ | \
  grep 'iqn.2026-08.example:restore-lun-001'
```

输出可能类似：

```text
ip-<target-ip>:3260-iscsi-iqn.2026-08.example:restore-lun-001-lun-0 -> ../../sdb
ip-<target-ip>:3260-iscsi-iqn.2026-08.example:restore-lun-001-lun-0-part1 -> ../../sdb1
ip-<target-ip>:3260-iscsi-iqn.2026-08.example:restore-lun-001-lun-0-part2 -> ../../sdb2
```

可以得到映射关系：

```text
<target-ip>:3260
  -> iqn.2026-08.example:restore-lun-001
  -> LUN 0
  -> /dev/sdb
  -> /dev/sdb1, /dev/sdb2
```

自动化脚本也应优先使用 `/dev/disk/by-path/`，因为 `/dev/sdX` 的命名会受登录顺序、本机磁盘数量和系统重启影响。

## 六、分析磁盘结构

查看块设备、文件系统和 LVM 信息：

```bash
lsblk -f
```

恢复盘可能呈现为一块原 Linux 系统盘：

```text
sdb
├─sdb1          xfs
└─sdb2          LVM2_member
  ├─vg_os-swap  swap
  ├─vg_os-home  xfs
  └─vg_os-root  xfs
```

这表示：

```text
/dev/sdb
  -> /dev/sdb1              XFS
  -> /dev/sdb2              LVM Physical Volume
  -> VG: vg_os
  -> LV: vg_os-root, vg_os-home, vg_os-swap
```

如果看不到 LVM 设备，可以尝试扫描：

```bash
pvscan
vgscan
lvscan
vgchange -ay <vg-name>
```

如果恢复盘中的 VG 名称与本机已有 VG 重名，不要贸然激活。可以使用 `vgimportclone` 处理克隆盘 VG UUID 和名称冲突，或者在隔离主机上操作。

## 七、只读挂载文件系统

恢复验证建议优先只读挂载，避免对恢复盘产生不必要写入。

创建挂载目录：

```bash
mkdir -p /mnt/iscsi_root
```

如果文件系统是 XFS，可以使用：

```bash
mount -o ro,norecovery,nouuid \
  /dev/mapper/<vg-name>-root \
  /mnt/iscsi_root
```

参数说明：

- `ro`：只读挂载。
- `norecovery`：不进行 XFS 日志恢复，降低写入风险。
- `nouuid`：忽略 XFS UUID 冲突，适合克隆盘、快照盘、备份恢复盘。

查看挂载内容：

```bash
ls -lah /mnt/iscsi_root
```

如果能看到类似系统目录，说明根文件系统已成功识别：

```text
bin -> usr/bin
boot
dev
etc
home
opt
root
usr
var
```

## 八、验证恢复数据

可以先查看目标目录：

```bash
ls -lah /mnt/iscsi_root/<path-to-verify>/
```

再读取抽样文件：

```bash
cat /mnt/iscsi_root/<path-to-verify>/<sample-file>
```

确认挂载点和文件系统类型：

```bash
df -hT | grep -E 'iscsi|vg_os|/mnt/iscsi_root'
```

输出类似：

```text
/dev/mapper/<vg-name>-root  xfs  50G  4G  46G  8%  /mnt/iscsi_root
```

到这里，完整读取链路已经验证成功：

```text
iSCSI Target 登录
  -> LUN 映射
  -> Linux 块设备识别
  -> 分区识别
  -> LVM 识别
  -> 文件系统识别
  -> 原数据正常读取
```

## 九、安全卸载和断开

完成验证后，先卸载文件系统：

```bash
umount /mnt/iscsi_root
```

确认没有挂载残留：

```bash
findmnt | grep -E 'iscsi|vg_os|/mnt/iscsi_root'
```

如果恢复盘中包含 swap，不应在恢复验证过程中启用原系统 swap。检查：

```bash
swapon --show
```

如果发现恢复盘 swap 被启用，应先关闭：

```bash
swapoff /dev/mapper/<vg-name>-swap
```

停用恢复盘对应 VG：

```bash
vgchange -an <vg-name>
```

再次确认块设备状态：

```bash
lsblk -f
```

最后退出 iSCSI Session：

```bash
iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --logout
```

确认没有活动会话：

```bash
iscsiadm -m session
```

如果输出类似下面内容，说明已经断开：

```text
iscsiadm: No active sessions.
```

## 十、为什么 logout 后 node 记录还在

执行 `--logout` 只会断开当前 Session，不会删除本机保存的 Node 配置。

查看 Node：

```bash
iscsiadm -m node
```

如果后续还要继续使用，可以保留 Node，并重新登录：

```bash
iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  --login
```

如果确认不再使用，可以删除 Node：

```bash
iscsiadm -m node \
  -T iqn.2026-08.example:restore-lun-001 \
  -p <target-ip>:3260 \
  -o delete
```

## 十一、标准操作流程

完整流程可以归纳为：

```text
获取 Target Portal 和 IQN
  -> 发现并登录 Target
  -> 验证 iSCSI Session
  -> 通过 /dev/disk/by-path 定位 LUN
  -> 使用 lsblk -f 分析分区、LVM、文件系统
  -> 只读挂载
  -> 验证数据
  -> umount
  -> 按需 swapoff
  -> vgchange -an
  -> iscsiadm --logout
  -> 按需删除 Node
```

## 十二、常用命令速查

发现 Target：

```bash
iscsiadm -m discovery -t sendtargets -p <target-ip>:3260
```

登录：

```bash
iscsiadm -m node \
  -T <target-iqn> \
  -p <target-ip>:3260 \
  --login
```

查看 Session：

```bash
iscsiadm -m session
```

查看 Node：

```bash
iscsiadm -m node
```

定位 IQN 对应磁盘：

```bash
ls -l /dev/disk/by-path/ | grep '<target-iqn>'
```

查看磁盘结构：

```bash
lsblk -f
```

XFS 只读挂载：

```bash
mount -o ro,norecovery,nouuid \
  /dev/mapper/<vg-lv> \
  /mnt/iscsi_root
```

卸载：

```bash
umount /mnt/iscsi_root
```

停用 VG：

```bash
vgchange -an <vg-name>
```

退出 Session：

```bash
iscsiadm -m node \
  -T <target-iqn> \
  -p <target-ip>:3260 \
  --logout
```

删除 Node：

```bash
iscsiadm -m node \
  -T <target-iqn> \
  -p <target-ip>:3260 \
  -o delete
```

## 十三、注意事项

- 不要对未知 LUN 执行 `mkfs`、`pvcreate`、`vgcreate`、`lvcreate` 等初始化操作。
- 恢复验证优先使用只读挂载，XFS 克隆盘常用 `ro,norecovery,nouuid`。
- 不要依赖固定 `/dev/sdX`，应通过 `/dev/disk/by-path/` 根据 Portal、IQN 和 LUN 定位。
- 如果使用 CHAP 认证，不要把密码写入博客、脚本仓库、截图或工单。
- 如果恢复盘 LVM 与本机 VG 重名，先处理 VG 冲突，再激活挂载。
- 不要在文件系统仍挂载、swap 仍启用或 LVM 仍活动时直接强制退出 iSCSI。
- 退出顺序建议为：停止访问、`umount`、`swapoff`、`vgchange -an`、`iscsiadm --logout`。

## 十四、总结

Linux 通过 iSCSI 读取远端 LUN，本质上是把网络块存储接入本机存储栈。恢复验证的关键不是“能不能登录成功”，而是能否准确识别 LUN 对应设备、正确处理分区和 LVM，并以尽量少写入的方式读取数据。

对于备份即时挂载、虚拟机磁盘恢复和文件级恢复场景，这套流程可以作为标准 runbook：先登录，后识别，再只读挂载，验证完成后按顺序卸载、停用 LVM 并退出 Session。只要把只读、安全退出和敏感信息保护做好，iSCSI 恢复会是一条非常稳的工具链。
