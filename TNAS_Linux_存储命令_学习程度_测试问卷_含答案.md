# TNAS/Linux 存储排查命令学习程度测评问卷（含答案与评分）

> 目的：检验你对 **磁盘/分区/文件系统、阵列（md RAID）、LVM（存储池/卷）、挂载、空间不足与 inode、常用管道命令** 等知识点的掌握程度。  
> 说明：本问卷分为选择题、填空题、判断题、简答题、场景题与实操题。  
> 建议：先独立作答；最后对照“答案与评分”自测。

---

## 评分规则与等级（建议）

- 选择题：每题 2 分  
- 填空题：每空 2 分（多空按空计分）  
- 判断题：每题 1 分  
- 简答题：每题 6 分（要点 3~4 个为满分）  
- 场景题：每题 10 分（思路 + 命令 + 判读要点）  
- 实操题：每题 10 分（命令正确性 + 判读 + 风险提示）

> 等级参考（按总分换算）  
- 90%+：熟练（可独立排查）  
- 70%~89%：合格（可在指导下排查）  
- 50%~69%：入门（需要补强核心链路）  
- <50%：需重学（先把“必背一句话 + 关键输出字段”打牢）

---

# A. 选择题（共 15 题）

**A1.** `df` 主要用于查看什么？  
A. 文件内容  
B. 文件系统磁盘空间占用情况（总/已用/可用/挂载点）  
C. 进程状态  
D. 网络连接  

**A2.** `df -h` 中的 `-h` 表示：  
A. hidden  
B. human-readable（人类可读单位）  
C. host  
D. hash  

**A3.** 当出现“磁盘还有空间，但创建文件失败 / No space left on device”，最优先需要再查哪个命令？  
A. `df -i`  
B. `whoami`  
C. `date`  
D. `ps aux`  

**A4.** `df -Th` 的 `-T` 主要用于显示：  
A. 时间  
B. 文件系统类型（ext4/btrfs/xfs…）  
C. 线程  
D. 端口  

**A5.** `du` 默认统计的是：  
A. 某个文件系统总容量  
B. 目录/文件的实际占用（递归）  
C. CPU 占用  
D. 网络带宽  

**A6.** `du -h --max-depth=1 /Volume1` 的 `--max-depth=1` 表示：  
A. 只统计 /Volume1 自身，不统计子目录  
B. 统计到 1 层子目录（一级目录）  
C. 统计到 1 个文件  
D. 只统计隐藏目录  

**A7.** `sort -hr` 的含义是：  
A. 按字典序升序排序  
B. 按人类可读单位排序并倒序（最大在前）  
C. 仅排序数字  
D. 随机排序  

**A8.** `head` 默认输出：  
A. 最后 10 行  
B. 前 10 行  
C. 前 100 行  
D. 不输出  

**A9.** `find ... -print0 | xargs -0 ...` 的目的主要是：  
A. 提高 CPU 使用率  
B. 正确处理带空格/特殊字符的文件名  
C. 输出 JSON  
D. 自动删除文件  

**A10.** `lsblk` 中的 `TYPE=part` 表示：  
A. 物理磁盘  
B. 分区（partition）  
C. 网络设备  
D. 目录  

**A11.** `lsblk -f` 最适合用来查看：  
A. 文件内容  
B. FSTYPE/LABEL/UUID/挂载点  
C. 端口占用  
D. 用户组权限  

**A12.** `/proc/mdstat` 的主要用途是：  
A. 查看网络状态  
B. 查看 md 阵列状态/同步进度  
C. 查看系统时间  
D. 查看 Python 版本  

**A13.** 在 `mdadm --detail /dev/md0` 输出中，判断阵列是否“有冗余”的关键字段之一是：  
A. `Creation Time`  
B. `Raid Devices` / 成员列表行数  
C. `UUID`  
D. `Name`  

**A14.** `Intent Bitmap : Internal` 表示：  
A. 禁用位图  
B. 使用内部写入位图加速异常后的 resync  
C. 使用外部文件位图  
D. 表示磁盘坏道  

**A15.** LVM 中 `VG`、`LV`、`PV` 的对应关系最正确的是：  
A. LV 由 PV 切出来，PV 由 VG 切出来  
B. VG 由 PV 组成，LV 从 VG 划分  
C. PV 从 LV 划分  
D. 三者无关系  

---

# B. 填空题（共 12 题）

**B1.** `df` 的英文全称/含义：`df = _____________________（__________）`  

**B2.** `df` 输出表头中，`Filesystem / Used / Available / Use% / Mounted on` 的中文含义分别是：  
- Filesystem：__________  
- Used：__________  
- Available：__________  
- Use%：__________  
- Mounted on：__________  

**B3.** `df -i` 输出表头中，`Inodes / IUsed / IFree / IUse%` 的中文含义分别是：__________ / __________ / __________ / __________  

**B4.** `cat /proc/mdstat` 中，`[x/y]` 的含义是：__________ / __________  

**B5.** `cat /proc/mdstat` 中，健康位 `[U]` 里的 `U` 表示：__________；若缺盘常用符号是：__________  

**B6.** `mdadm --detail` 中，`State : clean` 表示阵列：__________  

**B7.** `lsblk` 输出中，`disk / part / raid1 / lvm` 的含义分别是：__________ / __________ / __________ / __________  

**B8.** `blkid` 输出常见字段 `LABEL / UUID / TYPE` 的含义分别是：__________ / __________ / __________  

**B9.** 在 TNAS 的输出里，`/dev/md9` 挂载到 `/`，说明它是：__________ 分区/阵列（填：系统根/数据卷/交换分区…）  

**B10.** `--exclude=@recycle` 常用于在统计目录占用时排除：__________ 目录的影响。  

**B11.** `sudo -l` 用于查看：__________  

**B12.** `fdisk -l` 的 `-l` 表示：__________（英文关键词即可）  

---

# C. 判断题（共 10 题）

（正确写“对”，错误写“错”）

**C1.** `du` 显示的是文件系统总容量，等价于 `df`。  2
**C2.** `df -h` 的 Use% 可以用来快速判断某个挂载点是否接近满盘。  1
**C3.** inode 耗尽时，`df -h` 一定会显示 Available 为 0。  2
**C4.** `sort -hr` 中 `-h` 表示按“人类可读单位（K/M/G/T）”参与排序。  1
**C5.** `find ... -print0 | xargs -0 ...` 可以避免文件名含空格导致的拆分错误。  1
**C6.** `mdadm --detail` 是只读查看命令，不会写入阵列配置。  1
**C7.** `Raid Level : raid1` 就一定代表有两块盘做镜像。  2
**C8.** `lsblk -f` 可以同时看到 FSTYPE 和挂载点。  1
**C9.** 在 fdisk 交互界面里，只有输入 `w` 才会真正写入磁盘分区表。  1
**C10.** `mount` 和 `findmnt` 都能查看挂载信息，但 `findmnt` 更结构化。  1

---

# D. 简答题（共 8 题）

**D1.** 解释 `df` 与 `du` 的区别，并分别给出一个适用场景（各 1 个）。  

**D2.** 什么情况下会出现“磁盘还有空间，但创建文件失败 / No space left on device”？请给出至少 2 个原因，并写出对应排查命令。  

**D3.** 解释 `lsblk` 输出中 `disk / part / raid1 / lvm` 的关系链路，并结合 TNAS 举一个例子（如：sdza4 → md0 → vg0-lv0 → /Volume1）。  

**D4.** 用自己的话解释“位图（bitmap）”在 md RAID 里的作用。为什么它能让异常后的同步更快？  

**D5.** `mdadm --detail /dev/md0` 里，哪些字段可以用来判断阵列是否降级/是否健康？至少列出 4 个字段并说明。  

**D6.** 解释 `sudo -l` 的作用；如果你看到 “This account is currently not available”，但 `id` 显示 `uid=0`，你如何理解这个现象？  

**D7.** `mount` 输出行：`/dev/md9 on / type ext4 (...)`，如何判断“它是什么盘/哪个分区来的”？你会用哪些命令把链路追到物理盘？（至少 3 条命令）  

**D8.** `fdisk /dev/sda` 为什么会提示 “This disk is currently in use - repartitioning is probably a bad idea.”？此时你应该怎么做才安全？  

---

# E. 场景题（共 6 题）

**E1.（空间告警）** 业务反馈：`/Volume1` 写入失败，提示 `No space left on device`。你执行 `df -h /Volume1` 发现还有 300GB 可用。请写出你的排查步骤（命令序列 + 判读点）。  

**E2.（定位大目录）** `/Volume1` 的 Use% 从 20% 迅速涨到 85%。请写出用于定位“哪个一级目录占用最大”的命令，并解释管道中每一段的意义。  

**E3.（定位大文件）** 需要找出 `/Volume1` 下所有大于 5G 的文件并按大小排序输出。请写出完整命令，并解释为什么要用 `-print0` 和 `xargs -0`。  

**E4.（阵列健康检查）** 运维怀疑阵列降级。请写出你会检查的 3 条命令，并说明你在输出里重点看哪些字段/关键词来判定降级与同步进度。  

**E5.（卷/存储池链路确认）** 需要确认 `/Volume1` 是由哪些底层设备组成的（磁盘→分区→md→LVM→文件系统）。请写出一套“从挂载点出发逐层追溯”的命令序列。  

**E6.（USB 与 iSCSI 识别）** 系统里同时插了 USB 盘（挂载到 `/Volume1/@usb/...`）并挂载了 iSCSI（挂载到 `/Volume1/@iscsi/...`）。请写出 2~3 条命令，验证它们分别对应哪个设备、文件系统类型是什么。  

---

# F. 实操题（共 4 题，偏动手）

> 说明：实操题以“命令 + 你预计看到的关键输出字段”作答即可（不用真的执行）。

**F1.（追溯 /Volume1 来源）**  
写出命令：从 `/Volume1` 追溯到 `SOURCE`，再追溯到底层 `md` 或 `sdXn`，最终定位到物理盘名。要求：给出至少 4 条命令，并说明每条命令你想确认的点。  

**F2.（inode 耗尽验证）**  
写出命令：验证是否 inode 耗尽导致写入失败，并给出一种“最安全的清理思路”（不要求真删系统关键文件）。  

**F3.（md 阵列是否单盘/双盘判定）**  
写出命令：判定 `md0` 是否真正具备 RAID1 镜像冗余，并说明判定依据（字段/输出形态）。  

**F4.（LVM 底层 devices 追溯）**  
写出命令：查看 `vg0-lv0` 的底层 devices，并进一步定位其来源设备（如 `/dev/md0`），最后关联到成员分区（如 `/dev/sdza4`）。  

---

# ✅ 答案与评分要点（自测用）

## A. 选择题答案
A1-B；A2-B；A3-A；A4-B；A5-B；A6-B；A7-B；A8-B；A9-B；A10-B；A11-B；A12-B；A13-B；A14-B；A15-B

## B. 填空题参考答案
B1：disk free（磁盘可用空间）  
B2：Filesystem=文件系统/来源设备；Used=已用；Available=可用；Use%=使用率；Mounted on=挂载点  
B3：Inodes=inode 总数；IUsed=已用 inode；IFree=可用 inode；IUse%=inode 使用率  
B4：应有成员数 / 在线成员数  
B5：在线正常（Up）；缺失为 `_`  
B6：一致、无同步任务、健康（无 degraded/failed）  
B7：disk=物理磁盘；part=分区；raid1=md 软件阵列设备；lvm=LVM 逻辑卷  
B8：LABEL=卷标；UUID=唯一标识；TYPE=文件系统类型/成员类型  
B9：系统根（/）  
B10：回收站  
B11：当前用户可用的 sudo 权限/允许执行的命令列表  
B12：list

## C. 判断题答案
C1-错；C2-对；C3-错；C4-对；C5-对；C6-对；C7-错；C8-对；C9-对；C10-对

## D. 简答题评分要点（每题 6 分）
D1：df=文件系统整体容量/可用；du=目录/文件实际占用；各给 1 场景（如 df 看满盘，du 找大目录）  
D2：至少 2 原因：inode 耗尽（df -i）、权限/只读挂载（mount/findmnt）、配额/保留空间、btrfs 元数据满等（命令与判读）  
D3：说清树：disk→part→md→lvm→fs→mount；举 TNAS 示例链路  
D4：位图记录被写过的区域；异常后只同步“被标记区域”而非全盘  
D5：State、Raid Devices/Total Devices、Failed Devices、Active/Working Devices、成员表 State、/proc/mdstat 的 [U] 等  
D6：sudo -l 看权限；account not available 是 shell 登录限制/默认 shell；uid=0 表示权限已是 root（需按实际环境解释）  
D7：mount/findmnt→SOURCE；lsblk/lsblk -f 看层级；mdadm --detail 看成员；lsblk -no PKNAME 追到物理盘  
D8：盘已挂载在用；应 umount/swapoff；只读查看用 p/q；避免 w 写入

## E. 场景题评分要点（每题 10 分）
E1：先 df -i；再检查只读挂载/权限；再 du 定位；必要时查 btrfs 元数据；给出清理思路  
E2：du -h --max-depth=1 /Volume1 | sort -hr | head；解释 du/sort/head  
E3：find /Volume1 -type f -size +5G -print0 | xargs -0 ls -lh | sort -hk5；解释 print0/xargs -0/排序列  
E4：cat /proc/mdstat；mdadm --detail /dev/mdX；lsblk；判读 [x/y][U]、degraded、resync/recovery、Failed Devices  
E5：findmnt /Volume1；lvs -a -o +devices；mdadm --detail；lsblk -f；最终落到 sdXn  
E6：lsblk -f；mount | grep '@usb|@iscsi'；blkid /dev/sda1 /dev/sdb1

## F. 实操题评分要点（每题 10 分）
F1：至少 4 条命令：findmnt、lvs(+devices)、mdadm --detail、lsblk/PKNAME；每条说明目的  
F2：df -i 判读；find 统计小文件；清理方案强调“删临时/日志/业务缓存、避免系统目录”  
F3：mdadm --detail 看 Raid Devices/成员数；/proc/mdstat 看 [UU]/[_U]；指出“Raid Level=raid1≠一定双盘”  
F4：lvs -a -o lv_name,vg_name,lv_size,devices；pvs/vgs；mdadm --detail /dev/md0；lsblk 追到 sdza4 等

---

## 你可以怎么用这份问卷
- 第一次：不看答案独立作答，记录总分  
- 第二次：针对错题，把相关命令的“表头翻译 + 关键字段判读”再背一遍  
- 第三次：把 E/F 场景题写成自己的“排查 SOP”备忘
