# TNAS / TOS：Volume1 删除/卸载失败（target is busy）排查与处理笔记

> 目标：解决 `umount /Volume1` 报错 `target is busy`，并为后续 UI 删除 Volume 做准备。  
> 环境特征：`/Volume1` 为 **btrfs 子卷**（`/dev/mapper/vg0-lv0[/@]`），且卷内存在 **@iscsi / @usb 子挂载** 与 **@apps 应用进程**占用。

---

## 1. 现象

```bash
umount -f /Volume1
# => umount: /Volume1: target is busy.
```

---

## 2. 先看磁盘/挂载全貌（确认 Volume1 的设备与子挂载）

### 2.1 查看块设备（确认 Volume1 对应的 LV/RAID/LVM）
```bash
lsblk
```

关键结论示例：
- `/Volume1` ← `/dev/mapper/vg0-lv0`（btrfs）
- Volume1 目录下还可能挂载：`@iscsi`、`@usb`

### 2.2 查看 /Volume1 及其子挂载（**核心**）
```bash
findmnt -R /Volume1
# 或
mount | grep /Volume1
```

示例输出（你现场看到的结构）：
- `/Volume1`  ← `/dev/mapper/vg0-lv0[/@]`（btrfs）
- `/Volume1/@iscsi/...` ← `/dev/sdf1`（fuseblk）
- `/Volume1/@usb/...`   ← `/dev/sdh2`（ext4）
- `/Volume1/@usb/...`   ← `/dev/sdh3`（ext4）

---

## 3. 正确卸载顺序：先子挂载，再主挂载（从里到外）

> 只要子挂载存在，`/Volume1` 一定 busy。

### 3.1 确保当前 shell 不在 /Volume1 里
```bash
cd /
pwd
```

### 3.2 先卸载子挂载
```bash
umount /Volume1/@iscsi/26204102447-0
umount /Volume1/@usb/usb_generic_3
umount /Volume1/@usb/usb_generic_4
```

### 3.3 再卸载主卷
```bash
umount /Volume1
```

如果仍 `target is busy` → 进入第 4 步定位占用者。

---

## 4. 定位是谁占用 /Volume1（进程级）

### 4.1 fuser 快速看占用（可初筛）
```bash
# 单点查看
fuser -vm /Volume1
```

也可以批量查看（排查子挂载是否还残留）：
```bash
for m in \
  /Volume1/@iscsi/26204102447-0 \
  /Volume1/@usb/usb_generic_3 \
  /Volume1/@usb/usb_generic_4 \
  /Volume1
do
  echo "========== $m =========="
  fuser -vm "$m"
  echo
done
```

### 4.2 查看同一设备是否挂载在多个目录（btrfs 常见）
```bash
findmnt -S /dev/mapper/vg0-lv0
```

你现场的关键发现：
- `/Volume1`                 ← `/dev/mapper/vg0-lv0[/@]`
- `/var/subvols/xxxxxxxxxxxx`← `/dev/mapper/vg0-lv0[/@]`
- `/home`                    ← `/dev/mapper/vg0-lv0`（subvol=/）

> 说明：同一块设备存在多个挂载点，卸载策略要谨慎，避免误影响系统目录（尤其 `/home`）。

### 4.3 没有 lsof 时，用 /proc 扫描定位占用（**最有效**）

#### 4.3.1 扫 cwd：哪个进程工作目录在 /Volume1 下
```bash
for p in /proc/[0-9]*; do
  pid=${p#/proc/}
  cwd=$(readlink -f "$p/cwd" 2>/dev/null) || continue
  case "$cwd" in
    /Volume1* ) echo "PID=$pid  CWD=$cwd  CMD=$(tr '\0' ' ' < "$p/cmdline")" ;;
  esac
done
```

#### 4.3.2 扫 fd：哪个进程打开了 /Volume1 下的文件句柄
```bash
for p in /proc/[0-9]*; do
  pid=${p#/proc/}
  for fd in "$p"/fd/*; do
    t=$(readlink -f "$fd" 2>/dev/null) || continue
    case "$t" in
      /Volume1* )
        echo "PID=$pid  FD=$(basename "$fd")  TARGET=$t  CMD=$(tr '\0' ' ' < "$p/cmdline")"
        break
      ;;
    esac
  done
done
```

你现场定位到的占用者（示例）：
- `../scripts/iscsimanager` 占用 `/Volume1/@apps/iSCSIManager/...`
- `usbcopy` 占用 `/Volume1/@apps/USBCopy/...`

---

## 5. 处理占用：停服务 + 清理残留进程

### 5.1 先停常见服务（按需，报错可忽略）
```bash
systemctl stop smbd nmbd 2>/dev/null || true
systemctl stop nfs-server nfs-kernel-server 2>/dev/null || true
systemctl stop docker containerd 2>/dev/null || true
systemctl stop tgtd tgt iscsitarget target 2>/dev/null || true
```

### 5.2 查 PID 属于哪个 systemd unit（便于精准停）
```bash
systemctl status <PID> --no-pager || true
systemctl list-units --type=service --all | egrep -i 'iscsi|usbcopy'
```

你现场发现：
- `USBCopy.service` 已停止，但 `usbcopy` **仍残留运行**（systemd 提示 remains running）

### 5.3 彻底禁止 USBCopy 拉起（disable/mask）
```bash
systemctl stop USBCopy.service 2>/dev/null || true
systemctl disable USBCopy.service 2>/dev/null || true
systemctl mask USBCopy.service 2>/dev/null || true

# 验证
systemctl status USBCopy.service --no-pager 2>/dev/null || true
```

### 5.4 定点结束残留进程（TERM → KILL）
> 使用你现场扫描到的 PID

```bash
kill -TERM <PID1> <PID2> <PID3> 2>/dev/null || true
sleep 2
kill -KILL <PID1> <PID2> <PID3> 2>/dev/null || true
```

（可选）按路径特征一锅端：
```bash
pkill -TERM -f '/Volume1/@apps/USBCopy/sbin/usbcopy' 2>/dev/null || true
sleep 1
pkill -KILL -f '/Volume1/@apps/USBCopy/sbin/usbcopy' 2>/dev/null || true

pkill -TERM -f '/Volume1/@apps/iSCSIManager' 2>/dev/null || true
sleep 1
pkill -KILL -f '/Volume1/@apps/iSCSIManager' 2>/dev/null || true
```

---

## 6. 再次验证是否可卸载（直到只剩 kernel mount）

```bash
# 期望：不再出现任何用户态进程占用
fuser -vm /Volume1

# 也可再次执行 /proc fd 扫描，期望无输出
# （同第 4.3.2）
```

---

## 7. 最终卸载 /Volume1

```bash
umount /Volume1
```

验证：
```bash
mount | grep /Volume1 || echo "OK: /Volume1 已卸载"
findmnt -R /Volume1 || echo "OK: findmnt 无 /Volume1"
```

---

## 8. 兜底方案（慎用）

### 8.1 懒卸载（立刻从目录树摘除，待占用释放后完成卸载）
```bash
umount -l /Volume1
```

### 8.2 强杀占用（高风险，可能影响共享/容器/业务）
```bash
fuser -km /Volume1
```

---

## 9. 关键经验总结（本次 case 的“根因链路”）

1. `/Volume1` 下存在 **子挂载**（`@iscsi`、`@usb`）→ 必须先卸载子挂载  
2. 子挂载卸载后，仍 busy → 通过 `/proc` 扫描定位到 **@apps 应用进程占用**  
3. `USBCopy.service` 停止后仍有残留进程 → `mask` + `kill` 才能彻底释放句柄  
4. `fuser -vm /Volume1` 只剩 `kernel mount` → 卸载成功条件满足  
5. 成功执行 `umount /Volume1`，`mount | grep /Volume1` 为空 → 卸载完成

---

## 10. 注意事项（非常重要）

- `findmnt -S /dev/mapper/vg0-lv0` 显示同一设备还挂了 `/home`、`/var/subvols/...`：  
  - **卸载 /Volume1** 一般只是摘掉一个入口；  
  - 但如果你要“彻底删除 vg0-lv0 / 存储池”，可能影响系统/用户目录（风险极高），不要直接 `lvremove` 等破坏性操作，除非明确要重装/进入维护模式。

---

## 11. 一键复用的最短流程（建议收藏）

```bash
# 1) 看子挂载
findmnt -R /Volume1

# 2) 先卸子挂载
umount /Volume1/@iscsi/* 2>/dev/null || true
umount /Volume1/@usb/*   2>/dev/null || true

# 3) 查占用
fuser -vm /Volume1

# 4) 扫 /proc 找占用者（无 lsof 时）
for p in /proc/[0-9]*; do
  pid=${p#/proc/}
  for fd in "$p"/fd/*; do
    t=$(readlink -f "$fd" 2>/dev/null) || continue
    case "$t" in
      /Volume1* )
        echo "PID=$pid  TARGET=$t  CMD=$(tr '\0' ' ' < "$p/cmdline")"
        break
      ;;
    esac
  done
done

# 5) 停服务 + kill 占用进程（按实际 PID/服务名替换）
systemctl stop USBCopy.service 2>/dev/null || true
systemctl mask USBCopy.service 2>/dev/null || true
pkill -KILL -f '/Volume1/@apps/USBCopy/sbin/usbcopy' 2>/dev/null || true
pkill -KILL -f '/Volume1/@apps/iSCSIManager' 2>/dev/null || true

# 6) 卸载主卷
umount /Volume1
mount | grep /Volume1 || echo "OK: /Volume1 已卸载"
```
