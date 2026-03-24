# NAS 中“安全擦除 / 抹除磁盘”操作前后对比验证

## 1. 目的

本文用于指导在 NAS 中对磁盘执行：

- 安全擦除
- 抹除磁盘

前后的命令行验证。  
要求是 **一条一条执行**，并根据每条命令输出判断结果是否符合预期。

---

## 2. 功能区别

## 2.1 安全擦除

产品说明：

> 安全擦除主要用于防止数据泄密与敏感数据的保护。  
> 在执行安全擦除时，系统将向磁盘随机写入 0 和 1，直到覆盖所有磁盘空间。  
> 安全擦除后的磁盘将无法进行数据恢复。

### 预期重点

- 清除旧数据痕迹
- 清除分区表
- 清除文件系统/RAID/LVM 等签名
- 最终磁盘更接近 **完全裸盘**
- 一般预期 `Partition Table: unknown`

---

## 2.2 抹除磁盘

产品说明：

> 抹除磁盘相当于初始化磁盘。  
> 磁盘上的所有阵列信息、分区信息与文件系统都将被抹除。

### 预期重点

- 清除阵列信息
- 清除分区信息
- 清除文件系统信息
- 恢复为可重新初始化、可重新建池的空盘
- 允许出现两种结果：
  - `Partition Table: unknown`
  - `Partition Table: gpt`，但 **无任何分区项**

---

## 3. 验证前注意事项

## 3.1 必须先确认目标盘

先确认待操作磁盘设备名，例如：

- `/dev/sdd`：安全擦除示例
- `/dev/sdc`：抹除磁盘示例

## 3.2 禁止误操作系统盘/业务盘

执行前先确认该磁盘：

- 不是系统盘
- 不属于当前存储池成员盘
- 不承载当前业务卷
- 不在挂载中

## 3.3 建议记录操作前截图/日志

建议保存：

- UI 页面截图
- 命令输出日志
- 操作前/后对比结果

---

# 4. 操作前验证命令（执行操作前）

> 以下命令建议 **一条一条执行**，每执行一条就记录输出。

假设目标盘为：

```bash
/dev/sdX
```

实际执行时替换为：

```bash
/dev/sdd
# 或
/dev/sdc
```

---

## 4.1 查看整机磁盘与分区情况

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

### 判断方法

#### 若磁盘已存在分区
会看到类似：

```text
sdc           disk  238.5G
├─sdc1        part    285M vfat
├─sdc2        part    7.6G
├─sdc3        part    1.9G
├─sdc4        part  101.9G
└─sdc5        part  126.7G
```

说明：

- 该磁盘当前存在分区结构
- 后续执行抹除/安全擦除后，这些分区应消失

#### 若磁盘为裸盘
会看到类似：

```text
sdd           disk  238.5G
```

说明：

- 当前只有整盘，无分区

---

## 4.2 查看指定磁盘分区详情

```bash
lsblk /dev/sdX -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

### 判断方法

- 若存在 `sdX1、sdX2...`，说明有分区
- 若只剩 `sdX`，说明无分区

---

## 4.3 查看分区表信息

```bash
parted /dev/sdX print
```

### 判断方法

#### 若存在分区表
例如：

```text
Partition Table: gpt
Number  Start   End     Size    File system  Name     Flags
 1      ...
 2      ...
```

说明：

- 当前存在 GPT 分区表
- 且已有分区

#### 若结果为：
```text
Partition Table: gpt
Number  Start  End  Size  File system  Name  Flags
```

说明：

- 保留 GPT 标签
- 但没有分区项
- 这对“抹除磁盘”来说通常可接受

#### 若结果为：
```text
Error: /dev/sdX: unrecognised disk label
Partition Table: unknown
```

说明：

- 无可识别分区表
- 更接近完全裸盘
- 这是“安全擦除”最理想的结果

---

## 4.4 查看 fdisk 分区信息

```bash
fdisk -l /dev/sdX
```

### 判断方法

#### 若列出分区项
例如：

```text
Device         Start       End   Sectors   Size Type
/dev/sdc1 ...
/dev/sdc2 ...
```

说明：

- 当前分区仍存在

#### 若只显示整盘信息，不显示分区项
说明：

- 当前分区已删除或不存在

---

## 4.5 查看签名信息

```bash
wipefs -n /dev/sdX
```

### 判断方法

#### 若有输出
说明盘上仍存在可识别签名，例如：

- 文件系统签名
- GPT/MBR 签名
- RAID 签名
- LVM 签名

#### 若无输出
说明：

- 没有被 `wipefs` 识别到的签名
- 符合清理干净的预期

---

## 4.6 查看块设备类型识别

```bash
blkid /dev/sdX
```

### 判断方法

#### 若有输出
例如出现：

- `TYPE="ext4"`
- `TYPE="LVM2_member"`
- `TYPE="linux_raid_member"`

说明：

- 旧结构签名仍在

#### 若无输出
说明：

- 当前没有 TYPE/UUID
- 更符合裸盘预期

---

## 4.7 查看 RAID superblock

```bash
mdadm --examine /dev/sdX
```

### 判断方法

#### 若识别到 md superblock
说明：

- 磁盘仍有 RAID 元数据

#### 若提示未检测到
例如：

```text
mdadm: No md superblock detected on /dev/sdX.
```

说明：

- 没有 RAID superblock

---

## 4.8 查看 LVM 归属

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

### 判断方法

- 若 `pvs` 中有目标盘相关链路，说明盘仍参与 LVM
- 若没有目标盘引用，说明未参与当前卷组链路

---

## 4.9 查看挂载状态

```bash
mount | grep sdX
```

### 判断方法

- 若有输出，说明该盘仍在挂载使用
- 若无输出，说明当前无直接挂载痕迹

---

# 5. 执行操作

## 5.1 执行“安全擦除”

在 UI 中对目标空闲磁盘执行：

- 更多
- 安全擦除
- 确认

## 5.2 执行“抹除磁盘”

在 UI 中对目标空闲磁盘执行：

- 更多
- 抹除磁盘
- 确认

---

# 6. 操作后验证命令（执行操作后）

> 操作完成后，重复执行以下命令。  
> 推荐与操作前输出做逐项对比。

---

## 6.1 查看整机磁盘与分区情况

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

### 判断方法

#### 安全擦除后理想预期
例如：

```text
sdd           disk  238.5G
```

说明：

- 无分区节点
- 符合裸盘预期

#### 抹除磁盘后理想预期
例如：

```text
sdc           disk  238.5G
```

说明：

- 原有分区已删除
- 当前磁盘为空盘状态

---

## 6.2 查看指定磁盘分区详情

```bash
lsblk /dev/sdX -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

### 判断方法

- 若操作后只剩 `sdX`
- 不再有 `sdX1~sdXn`
- 说明分区结构已被清除

---

## 6.3 查看 parted 结果

```bash
parted /dev/sdX print
```

### 判断方法

#### 安全擦除通过标准（推荐）
```text
Error: /dev/sdX: unrecognised disk label
Partition Table: unknown
```

说明：

- 磁盘标签与分区表均不可识别
- 更符合安全擦除“完全裸盘”预期

#### 抹除磁盘通过标准 1
```text
Partition Table: gpt
Number  Start  End  Size  File system  Name  Flags
```

说明：

- 保留 GPT 标签
- 但无分区项
- 符合“初始化磁盘”预期

#### 抹除磁盘通过标准 2
```text
Partition Table: unknown
```

说明：

- 比“空 GPT”更彻底，也可判定通过

---

## 6.4 查看 fdisk 结果

```bash
fdisk -l /dev/sdX
```

### 判断方法

#### 通过标准
- 只显示整盘信息
- 不显示任何 `/dev/sdX1~sdXn` 分区项

说明：

- 分区已被删除

---

## 6.5 查看签名结果

```bash
wipefs -n /dev/sdX
```

### 判断方法

#### 通过标准
- 无输出

说明：

- 无可识别签名残留

---

## 6.6 查看 blkid 结果

```bash
blkid /dev/sdX
```

### 判断方法

#### 通过标准
- 无输出

说明：

- 无 TYPE/UUID
- 当前无文件系统/RAID/LVM 类型识别

---

## 6.7 查看 mdadm 结果

```bash
mdadm --examine /dev/sdX
```

### 判断方法

#### 通过标准
- 不识别 RAID superblock
- 或提示未检测到 md superblock

---

## 6.8 查看 LVM 结果

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

### 判断方法

#### 通过标准
- 不出现目标盘相关 PV/VG/LV 引用

---

# 7. 结合实际示例说明

## 7.1 示例一：`/dev/sdd` 执行安全擦除

### 操作后结果

```bash
lsblk /dev/sdd -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

结果表现为：

```text
sdd  disk 238.5G
```

```bash
parted /dev/sdd print
```

结果：

```text
Error: /dev/sdd: unrecognised disk label
Partition Table: unknown
```

```bash
wipefs -n /dev/sdd
```

结果：

```text
# 无输出
```

```bash
blkid /dev/sdd
```

结果：

```text
# 无输出
```

### 判断

说明：

- 分区表已清除
- 分区节点已消失
- 可识别签名已清除
- 磁盘恢复为未初始化裸盘状态

### 结论

```md
/dev/sdd 执行安全擦除后，磁盘无分区表、无分区节点、无可识别签名，恢复为未初始化裸盘状态，符合安全擦除预期。
```

---

## 7.2 示例二：`/dev/sdc` 执行抹除磁盘

### 操作前结果

```bash
lsblk /dev/sdc -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

结果：

```text
sdc  disk 238.5G
├─sdc1 part 285M vfat
├─sdc2 part 7.6G
├─sdc3 part 1.9G
├─sdc4 part 101.9G
└─sdc5 part 126.7G
```

```bash
parted /dev/sdc print
```

结果：

```text
Partition Table: gpt
# 存在 1~5 分区
```

### 操作后结果

```bash
lsblk /dev/sdc -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

结果：

```text
sdc  disk 238.5G
```

```bash
parted /dev/sdc print
```

结果：

```text
Partition Table: gpt

Number  Start  End  Size  File system  Name  Flags
```

```bash
fdisk -l /dev/sdc
```

结果：

- 仅显示整盘信息
- 不再显示 `/dev/sdc1~sdc5`

### 判断

说明：

- 原有分区已被删除
- 当前无分区结构
- GPT 标签仍保留
- 该盘已恢复为空盘可重用状态

### 结论

```md
/dev/sdc 执行抹除磁盘后，原有分区结构已被删除，当前处于保留 GPT 磁盘标签但无分区的空盘状态，符合抹除磁盘预期。
```

---

# 8. 安全擦除与抹除磁盘的判断区别

## 8.1 安全擦除重点

重点看：

- `lsblk` 是否无分区
- `parted` 是否为 `unknown`
- `wipefs -n` 是否无输出
- `blkid` 是否无输出

### 推荐结论
更强调 **“彻底清空痕迹”**

---

## 8.2 抹除磁盘重点

重点看：

- 原有分区是否消失
- 原有文件系统/阵列是否失效
- 是否能恢复为空盘可重用状态

### 推荐结论
更强调 **“初始化磁盘，清空结构信息”**

### 注意
对抹除磁盘来说：

- `Partition Table: unknown` 可以通过
- `Partition Table: gpt` 但无分区项，也可以通过

---

# 9. 推荐执行顺序（可直接照着跑）

假设目标盘为 `/dev/sdX`

## 9.1 操作前执行

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

```bash
lsblk /dev/sdX -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

```bash
parted /dev/sdX print
```

```bash
fdisk -l /dev/sdX
```

```bash
wipefs -n /dev/sdX
```

```bash
blkid /dev/sdX
```

```bash
mdadm --examine /dev/sdX
```

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

## 9.2 在 UI 中执行操作

- 安全擦除 或 抹除磁盘

## 9.3 操作后执行

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

```bash
lsblk /dev/sdX -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINT
```

```bash
parted /dev/sdX print
```

```bash
fdisk -l /dev/sdX
```

```bash
wipefs -n /dev/sdX
```

```bash
blkid /dev/sdX
```

```bash
mdadm --examine /dev/sdX
```

```bash
pvs
```

```bash
vgs
```

```bash
lvs
```

---

# 10. 最终结论模板

## 10.1 安全擦除结论模板

```md
对 /dev/sdX 执行“安全擦除”后，通过 `lsblk`、`parted`、`wipefs`、`blkid` 验证，目标盘已无分区节点、无分区表、无可识别签名，恢复为未初始化裸盘状态，符合安全擦除预期。
```

## 10.2 抹除磁盘结论模板

```md
对 /dev/sdX 执行“抹除磁盘”后，通过 `lsblk`、`parted`、`fdisk`、`pvs/vgs/lvs` 验证，目标盘原有分区结构及业务存储信息已被清除，当前恢复为空盘可重用状态，符合抹除磁盘预期。
```

---

# 11. 一句话总结

## 安全擦除
**看“unknown + 无签名 + 无分区”**

## 抹除磁盘
**看“无分区 + 无业务结构”即可，空 GPT 也可通过**
