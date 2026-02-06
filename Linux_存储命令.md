# Linux 存储/磁盘命令：表头翻译 + 记忆法 + 实战解释（加强版，适配 TNAS/TOS）

> 你要求的记忆方式：每个模块都给出 **“命令名来源/缩写（比如 df=Disk Free）”**，方便背诵。  
> 你要求的输出格式：每个命令的“表头翻译”使用 **三行**：  
> **第 1 行：原英文表头/字段** → **第 2 行：中文含义** → **第 3 行：排障看点/解读要点**  
>  
> ⚠️ 高危命令（会改盘/会清数据）会用 **⚠️** 标注，并给出“安全三连确认”。

---
## 1.0 必须记住的句子（你要求原样保留）

> 以下内容 **按你提供的原句** 保留，建议直接背诵：

- df:文件系统磁盘空间占用情况(文件系统的总空间、已用空间、可用空间以及挂载点)
- df -h:硬盘使用情况（容量，已使用容量/百分比，剩余容量，挂载目录,卷情况）
- cat /proc/mdstat：查看阵列同步进度
- fdisk -l：列出所有连接到系统的磁盘设备，包括NAS存储设备
- fdisk /dev/sda：选择磁盘（---w：写入---q：退出不保存）
- （1）---n：创建分区---（1-..）：分区编号（例如：1即sda1）
- （2）--d：删除分区---
- mkfs.ext4 /dev/sdc1：把sdc1区域格式化为ext4格式
- mkfs.vfat /dev/sdc1：把sdc1区域格式化为vfat格式
- blkid /dev/sdc1：查看是否有“UTOSDISK”标记（sda1,sdb1...）

---

## 0. 先把 TNAS/TOS 的存储结构记成一句话（你这台机器就是典型范式）

> **物理盘/分区 → md RAID（/dev/md*）→ LVM（/dev/mapper/vg-lv）→ 文件系统（ext4/btrfs）→ 挂载点（/、/Volume1）**

对照你已经看到的例子：
- `/dev/md9`（raid1）→ `ext4` → `/`（系统根分区）
- `/dev/md0`（raid1）→ `LVM2_member`（PV）→ `vg0-lv0`（LV）→ `btrfs` → `/Volume1`（数据卷）
- `/dev/sda1`（exfat）→ `/Volume1/@usb/...`（USB）
- `/dev/sdb1`（fuseblk）→ `/Volume1/@iscsi/...`（iSCSI）

---

## 0.1 安全三连确认（任何写盘/格式化前必做）
> 只要你准备对 `/dev/sdX`、`/dev/sdX1`、`/dev/mdX` 做写入/删除/格式化，先跑三连确认：

```bash
lsblk -f
blkid /dev/sdX1
mount | grep -E 'sdX1|mdX|vg0-lv0'
```

- 目的：确认这个设备 **有没有被系统/阵列/LVM 使用**；确认 `TYPE` 是不是 `linux_raid_member` / `LVM2_member`（这两种千万别直接 mkfs）。

---

# 1. df ：查看磁盘可用空间（重点）

## 1.0 记忆法（一定要背）
- `df` = **Disk Free**（磁盘可用空间/可用块）
- 口诀：**df 看“盘/挂载点整体”，du 看“目录谁占”**

> df 是按“文件系统视角”统计空间（对已挂载的文件系统有效），本质上读取的是 `statfs`（内核文件系统统计接口），所以看到的是“文件系统的容量视图”，不是“某个目录的文件累加”。

---

## 1.1 df -h
**用途**：以易读单位显示（GiB/TiB），查看“已挂载文件系统”的总/已用/可用、使用率、挂载点。  
**典型场景**：空间不足告警、写入失败、判断是系统盘 `/` 还是数据卷 `/Volume1` 快满。

### 1.1.1 表头翻译（按你要求三行）
```text
Filesystem    Size   Used   Avail  Use%  Mounted on
文件系统/设备名  总容量   已用   可用    使用率  挂载点
看点：先看 Use% 找快满的“挂载点”；再看 Mounted on 判断“哪棵目录树”；最后看 Filesystem 识别是 md/mapper/usb/iscsi
```

### 1.1.2 每列怎么“秒懂”（具体解释）
- **Filesystem**：通常是设备名  
  - `/dev/md9`：md RAID（软件阵列）  
  - `/dev/mapper/vg0-lv0`：LVM 逻辑卷（卷组 vg0 下的 lv0）  
  - `/dev/sda1`：物理盘分区（多见 USB）  
- **Mounted on**：决定“满的是哪一棵目录树”  
  - `/`：系统盘  
  - `/Volume1`：主数据卷  
  - `/Volume1/@usb/...`：USB  
  - `/Volume1/@iscsi/...`：iSCSI 相关卷  
- **Use%**：最关键；>90% 就要预警，接近 100% 会导致服务异常

### 1.1.3 实战用法（直接复制）
```bash
df -h
df -h /Volume1
df -h /          # 只看系统根分区
df -hT           # 也可以用 -T 同时看类型
```

### 1.1.4 常见坑
- df 看到空间很大，但仍提示写入失败：  
  - 可能是 inode 满 → 用 `df -i`  
  - 可能是文件系统只读（ro） → 用 `mount` 看是否 `ro`
- btrfs 受快照/回收站影响：df 显示的可用空间可能与“实际可回收空间”存在差异 → 用 `btrfs filesystem usage`

---

## 1.2 df（不带 -h，单位是 1K-blocks）
**用途**：和 `df -h` 相同，只是单位不直观（按 1KB 块）。  
**你贴的输出就是这一种。**

### 1.2.1 表头翻译（三行）
```text
Filesystem           1K-blocks    Used  Available Use% Mounted on
文件系统/设备名         总容量(1KB块)  已用   可用     使用率  挂载点
看点：字段同 df -h，只是容量单位变成 1KB；排障仍优先看 Use% 与 Mounted on
```

---

## 1.3 df -Th（显示文件系统类型）
## 1.3.0 记忆法
- `-T` = **Type**（类型），`-h` = Human readable（易读单位）

### 1.3.1 表头翻译（三行）
```text
Filesystem  Type  Size  Used  Avail  Use%  Mounted on
文件系统名    类型   总量   已用   可用    使用率  挂载点
看点：Type=ext4 常见系统盘；Type=btrfs 常见数据卷；Type=tmpfs 为内存盘；Type=exfat 常见U盘
```

### 1.3.2 实战价值
- 一眼区分 `/` 是 ext4 还是 btrfs
- 一眼识别 `/Volume1` 是 btrfs（支持子卷/快照），所以同设备多挂载点往往正常（subvol）

---

## 1.4 df -i（inode 使用情况）
## 1.4.0 记忆法
- `-i` = inode（索引节点/文件条目计数）

### 1.4.1 表头翻译（三行）
```text
Filesystem    Inodes  IUsed   IFree  IUse% Mounted on
文件系统名     inode总数 已用inode 剩余inode inode使用率 挂载点
看点：IUse% 100% 时就算 Avail 很大也写不进；海量小文件/日志/缓存会吃 inode；btrfs/exfat 常显示 0 或 '-' 属统计方式差异
```

### 1.4.2 典型故障表现
- “磁盘还有空间，但创建文件失败 / No space left on device”
- 解决思路：清理海量小文件、合并碎文件、清日志、减少临时文件

---

# 2. du ：查看目录/文件占用（定位谁占空间）

## 2.0 记忆法
- `du` = **Disk Usage**（磁盘占用）
- 口诀：**df 看挂载点，du 找目录罪魁祸首**

> du 是从目录树向下遍历统计“文件大小累加”，所以在大目录上可能比较慢。

---

## 2.1 du（默认当前目录）
### 2.1.1 输出格式解释（三行）
```text
<Size>    <Path>
占用大小    路径
看点：Size 是该路径下所有文件累计占用；Path 是目录/文件；用于从大到小定位
```

---

## 2.2 du -sh（汇总 + 易读单位）
## 2.2.0 记忆法
- `-s` = summary（只要总数）
- `-h` = human readable（易读单位）

```bash
du -sh /Volume1
du -sh /Volume1/*
```

---

## 2.3 du -h --max-depth=1（按层级统计）
## 2.3.0 记忆法
- `--max-depth=1`：只统计到 1 层（一级目录）

```bash
du -h --max-depth=1 /Volume1 | sort -hr | head
```

---

## 2.4 du 常见技巧（排障更快）
- 排除某些目录：`du -h --exclude=@recycle --max-depth=1 /Volume1`
- 找大文件：`find /Volume1 -type f -size +5G -print0 | xargs -0 ls -lh | sort -hk5`

---

# 3. lsblk ：查看块设备拓扑（磁盘/分区/阵列/LVM/挂载点）

## 3.0 记忆法
- `lsblk` = **List Block Devices**（列出块设备）
- 口诀：**不认盘先 lsblk，认“谁在用”看 MOUNTPOINTS**

---

## 3.1 lsblk
### 3.1.1 表头翻译（三行）
```text
NAME  MAJ:MIN RM SIZE RO TYPE  MOUNTPOINTS
名称   主:次设备号 可移动 容量 只读 类型   挂载点(可多行)
看点：TYPE=disk/part/raid*/lvm；RM=1 多为U盘/USB；MOUNTPOINTS 能确认“这个设备正在被哪个目录树使用”
```

### 3.1.2 TYPE 列常见取值（必会）
- `disk`：整块物理盘（如 sda/sdb/sdza）
- `part`：分区（如 sda1/sdza4）
- `raid1/raid5...`：md 阵列层（如 md9/md0）
- `lvm`：LVM 逻辑卷（如 vg0-lv0）

---

## 3.2 lsblk -f（显示文件系统、UUID、卷标）
## 3.2.0 记忆法
- `-f` = filesystem info（文件系统信息）

### 3.2.1 表头翻译（三行）
```text
NAME  FSTYPE  FSVER  LABEL  UUID  FSAVAIL  FSUSE%  MOUNTPOINTS
名称   文件系统 版本   卷标    UUID  可用空间  使用率    挂载点
看点：LABEL/UUID 用于稳定识别盘；FSUSE% 快速看满不满；同一设备多挂载点常见于 btrfs 子卷挂载
```

---

## 3.3 lsblk -o（自定义列）
```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS
```

---

# 4. blkid ：查看设备身份（UUID/LABEL/TYPE）

## 4.x 必背一句话（按你给的原句）
- blkid /dev/sdc1：查看是否有“UTOSDISK”标记（sda1,sdb1...）


## 4.0 记忆法
- `blkid` = **Block ID**（块设备身份信息：UUID/类型/卷标）
- 口诀：**要做持久挂载/确认是不是阵列成员 → blkid**

---

## 4.1 blkid 输出字段翻译（三行）
```text
UUID  LABEL  TYPE  PARTUUID  PARTLABEL  BLOCK_SIZE  UUID_SUB  PTTYPE
UUID  卷标    类型   分区UUID   分区标签     块大小     子UUID    分区表类型
看点：TYPE=linux_raid_member 表示 RAID 成员(别 mkfs)；TYPE=LVM2_member 表示 LVM PV(别 mkfs)；UUID 用于 fstab 稳定挂载
```

## 4.2 TYPE 常见值（必须会判读）
- `linux_raid_member`：md RAID 成员分区（真正的数据在 /dev/mdX 上）
- `LVM2_member`：LVM 物理卷 PV（真正的文件系统在 /dev/mapper/vg-lv 上）
- `ext4 / btrfs / xfs`：真正可挂载的文件系统
- `swap`：交换分区
- `vfat/exfat`：U盘/外设常见

---

# 5. mount / findmnt ：挂载关系与参数（解释 btrfs 子卷非常关键）

## 5.0 记忆法
- `mount`：**挂载列表 + 参数**
- `findmnt`：**更像表格的 mount（更易筛选）**

---

## 5.1 mount（输出格式三行解释）
```text
<source> on <target> type <fstype> (<options>)
源设备   挂载到  目标目录   类型     参数列表
看点：rw/ro 决定可写；btrfs 常见 subvol=... 表示子卷；noatime/relatime 影响性能；discard=async 与SSD回收相关
```

---

## 5.2 btrfs 子卷相关参数（你 TNAS 输出里就有）
常见字段解释：
- `subvol=/@`：挂载的是名为 `@` 的子卷（很多系统用它当“主数据子卷”）
- `subvolid=256`：子卷 ID（内部编号）
- `subvol=/@/homes`：home 子卷
- `space_cache=v2`：空间缓存
- `discard=async`：异步 TRIM（SSD 更相关）
- `noatime`：不更新访问时间（减少写入）

---

## 5.3 findmnt（如果系统有）
```text
SOURCE  TARGET  FSTYPE  OPTIONS
源设备   挂载点   类型     参数
看点：用 TARGET 快速定位某目录属于哪个设备；OPTIONS 看 subvol/ro/rw 等；比 mount 更像表格
```

示例：
```bash
findmnt -o SOURCE,TARGET,FSTYPE,OPTIONS | grep -E 'Volume1|md9|vg0-lv0|@usb|@iscsi'
```

---

# 6. 阵列（md RAID）：状态、同步进度、降级判读

## 6.x 必背一句话（按你给的原句）
- cat /proc/mdstat：查看阵列同步进度


## 6.0 记忆法
- `md` = Multiple Device（多设备组成阵列）
- `mdadm` = **Multiple Device Admin**（阵列管理工具）
- `/proc/mdstat`：阵列状态“实时看板”
- 口诀：**阵列慢/担心降级 → 先看 /proc/mdstat**

---

## 6.1 cat /proc/mdstat
### 6.1.1 输出结构解释（三行）
```text
mdX : active raid1  sdXn[0] sdYn[1]
阵列名： 活跃  RAID级别  成员分区[槽位] 成员分区[槽位]
看点：raid1/5/6/10；成员后面的 [0]/[1] 是槽位；出现 resync/recovery/reshape 表示在同步/重建；[UU] 正常，[_U] 缺盘降级
```

### 6.1.2 关键字（必背）
- `resync`：同步（常见于新建阵列或加盘后）
- `recovery`：重建（坏盘更换后）
- `reshape`：阵列形态调整（迁移/扩容）
- `[UU]`：盘齐；`[_U]`：缺盘（降级）

---

## 6.2 mdadm --detail /dev/mdX（若系统有）
```text
Raid Level  Array Size  Raid Devices  Active Devices  State
RAID级别      阵列容量     预期盘数       活跃盘数        状态
看点：State=clean/degraded/recovering；Active < Raid Devices = 降级；同步时会显示 Rebuild/Resync 相关字段
```

---

# 7. LVM：存储池/卷（PV/VG/LV）全套表头翻译

## 7.0 记忆法（超好背）
- `pv` = **Physical Volume**（物理卷：把设备纳入 LVM 管理）
- `vg` = **Volume Group**（卷组：多个 PV 组成“池”）
- `lv` = **Logical Volume**（逻辑卷：从池里切出来的“卷”）
- 口诀：**PV=盘，VG=池，LV=卷**

---

## 7.1 pvs（物理卷）
```text
PV   VG   Fmt  Attr  PSize  PFree
物理卷 卷组 格式 属性  总大小  剩余
看点：PV 常是 /dev/md0 或 /dev/sdXn；PFree>0 表示池还有空间；Attr 看是否可用/是否被分配
```

常用增强：
```bash
pvs -o pv_name,vg_name,pv_size,pv_free,pv_used
```

---

## 7.2 vgs（卷组：可类比“存储池”）
```text
VG   #PV  #LV  #SN  Attr  VSize  VFree
卷组  PV数 LV数 快照数 属性  总大小  剩余
看点：VFree 决定是否还能扩卷；#PV/#LV 反映结构规模；Attr 体现是否激活/是否可写
```

---

## 7.3 lvs（逻辑卷：可类比“卷”）
```text
LV   VG   Attr  LSize  Pool  Origin  Data%  Meta%
逻辑卷 卷组 属性  大小   池    源卷     数据占用 元数据占用
看点：普通卷关注 LSize；thinpool/快照才更关注 Data%/Meta%；Attr 可判断是否激活、是否只读
```

---

## 7.4 lvs -a -o +devices（最关键：追溯“卷到底用哪块盘”）
```text
LV   VG   Attr  LSize  Devices
逻辑卷 卷组 属性  大小   底层设备映射
看点：Devices 会显示 /dev/md0(0) 或 /dev/sdXn(...)；用于判定“哪些盘是数据卷底座 → 千万别 fdisk/mkfs”
```

---

# 8. 分区（Partition）与格式化（Filesystem）：从“裸盘”到“可用目录”

## 8.x 必背一句话（按你给的原句）
- fdisk -l：列出所有连接到系统的磁盘设备，包括NAS存储设备
- fdisk /dev/sda：选择磁盘（---w：写入---q：退出不保存）
- （1）---n：创建分区---（1-..）：分区编号（例如：1即sda1）
- （2）--d：删除分区---


## 8.0 记忆法（背了就不容易搞混）
- `fdisk`：**分区**（Partition Table 操作工具）
- `mkfs`：**Make File System**（创建文件系统=格式化）
- 口诀：**先分区（fdisk），再格式化（mkfs），再挂载（mount），最后验证（lsblk/df/blkid）**

---

## 8.1 fdisk -l（只读安全）
```text
Device  Boot  Start  End  Sectors  Size  Id  Type
设备名  启动标志 起始  结束   扇区数   大小  类型ID 分区类型
看点：确认目标盘/分区号；Type 判定是否 EFI/Linux；Size 是否符合预期；Start/End 用于核对分区范围
```

---

## 8.2 fdisk /dev/sdX（⚠️高危：w 写入后立即生效）
### 8.2.1 交互命令对照（三行）
```text
p   n   d   w   q
打印分区表 新建分区 删除分区 写入保存 退出不保存
看点：w=不可逆写入(危险点)；q=保命键；任何操作前先 p 再确认设备是不是阵列成员/系统盘
```

---

## 8.3 mkfs.*（⚠️高危：格式化会清空该分区文件系统结构）

### 8.3.x 必背一句话（按你给的原句）
- mkfs.ext4 /dev/sdc1：把sdc1区域格式化为ext4格式
- mkfs.vfat /dev/sdc1：把sdc1区域格式化为vfat格式

- `mkfs.ext4`：ext4 格式化
- `mkfs.vfat`：FAT/vfat 格式化
- `mkfs.btrfs`：btrfs 格式化

### 8.3.1 通用“输出字段理解”（不同 mkfs 输出略不同，给你一套通用读法）
```text
Block size  Blocks  Inodes  UUID  Label
块大小       块数量   inode数  UUID  卷标
看点：Label/UUID 用于识别；Block size 影响性能/小文件；mkfs 输出后必须用 blkid/lsblk -f 复核目标是否正确
```

### 8.3.2 格式化后必做验证
```bash
blkid /dev/sdX1
lsblk -f
df -Th | grep -E 'sdX1|/mnt'
```

---

## 8.4 wipefs（⚠️极高危：擦除签名）
## 8.4.0 记忆法
- `wipefs`：wipe filesystem signatures（擦除文件系统/RAID/LVM 签名）

```text
offset  type
偏移量  签名类型
看点：用于清除旧文件系统/旧RAID/LVM残留签名；先用 -n 预演，确认无误再 -a
```

安全预演：
```bash
wipefs -n /dev/sdX1
```

---

# 9. 文件系统进阶：ext4 vs btrfs（你 TNAS 的 Volume1 是 btrfs）

## 9.0 记忆法
- ext4：传统稳定，常见系统盘
- btrfs：子卷/快照/校验，常见 NAS 数据卷

---

## 9.1 ext4：查看信息与检查
### 9.1.1 tune2fs -l /dev/md9（查看 ext4 元信息）
```text
Filesystem state  Mount count  Check interval  Last checked
文件系统状态        挂载次数      检查周期         上次检查时间
看点：state=clean/dirty；出现错误时安排离线检查；系统盘不要随便在线 fsck 修复
```

### 9.1.2 fsck.ext4 -n（仅检查，不修复）
```bash
fsck.ext4 -n /dev/md9
```

---

## 9.2 btrfs：子卷、空间、快照（适配 /Volume1）
### 9.2.1 btrfs subvolume list /Volume1
```text
ID  gen  top level  path
子卷ID  代数/版本  顶层ID   路径
看点：path=/@ 常是主数据子卷；/home 常是 /@/homes；用于解释“同一设备多挂载点”
```

### 9.2.2 btrfs filesystem usage /Volume1（比 df 更真实）
```text
Device size  Used  Free (estimated)  Data/Metadata/System
设备大小       已用   估算可用           数据/元数据/系统区占用
看点：btrfs 的元数据(Metadata)会影响可用空间；快照/回收站会让 df 看起来“可用少”，需要结合 usage 判断
```

---

# 10. 盘健康与性能（排障闭环）

## 10.0 记忆法
- S.M.A.R.T.：盘自检指标（是否劣化）
- iostat：I/O 延迟（是不是盘/阵列在忙）

---

## 10.1 smartctl -a /dev/sdX（S.M.A.R.T.）
```text
SMART overall-health self-assessment test result
SMART 总体健康自检结果
看点：PASSED/FAILED；重点看重映射/待定扇区/CRC错误，CRC错误多可能是线材/背板问题
```

---

## 10.2 iostat -x 1（若系统有）
```text
r/s  w/s  await  svctm  %util
读次数 写次数 平均等待 服务时间 利用率
看点：await 高=延迟大；%util 长期 100%=磁盘打满；阵列 resync 时这些指标会上升属于正常
```

---

# 11. 模块—命令速查（用于面试/日常复习）
- **空间**：`df -h/-Th/-i`（整体） + `du -sh/--max-depth`（谁占）
- **认盘拓扑**：`lsblk -f`（树 + FSTYPE/UUID/LABEL） + `blkid`（身份）
- **挂载参数**：`mount` / `findmnt`
- **阵列**：`cat /proc/mdstat` + `mdadm --detail`
- **LVM（池/卷）**：`pvs/vgs/lvs -a -o +devices`
- **分区/格式化**：`fdisk -l`（只读） / `fdisk`（⚠️写入） / `mkfs.*`（⚠️清空） / `wipefs`（⚠️擦签名）
- **文件系统进阶**：`tune2fs/fsck`（ext4） + `btrfs subvolume/filesystem usage`
- **健康/性能**：`smartctl` + `iostat`

---

## 12. 你下一步想继续加深的话（建议）
你已经抓到完整链路了，下一步最“贴 TNAS 实际”的强化是：
1) 把 `cat /proc/mdstat` 的实际输出贴出来，我可以按 `[UU]`、resync 进度逐项解释  
2) 跑 `pvs/vgs/lvs -a -o +devices`，我可以把“vg0-lv0 底层到底是哪块 md/分区”画成拓扑图