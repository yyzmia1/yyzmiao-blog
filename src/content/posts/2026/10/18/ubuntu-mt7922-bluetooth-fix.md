---
title: "不升级内核解决 Ubuntu 6.8 下联发科 MT7922 (13d3:3585) 蓝牙超时 (-110) 无法开启问题"
published: 2026-10-18T16:00:00+08:00
description: "记录在 Ubuntu 6.8 内核下通过二进制 Patch btusb 驱动硬件 ID 表并剔除失效数字签名，快速解决联发科 MT7922 (13d3:3585) 蓝牙 -110 超时故障的无痛修复方案与底层原理解析。"
category: "技术"
subcategory: "硬件与系统"
tags: ["Linux", "Ubuntu", "Bluetooth", "MT7922", "故障排查"]
draft: false
lang: "zh_CN"
---

如果你在 Ubuntu（特别是 6.8 内核，如 Ubuntu 24.04 LTS 或 22.04 HWE）下遇到联发科 MT7922（RZ616）无线网卡的 Wi-Fi 正常，但**蓝牙完全搜不到、打不开**，同时因为机器上跑着深度学习环境或依赖特定版本的 NVIDIA 显卡驱动、CUDA 而不敢轻易升级内核，这篇文章可以帮你在保留现有内核的前提下，5 分钟内直接修复蓝牙。

---

## 一、 先确认：你的问题和本文一样吗？

动手操作前，建议先打开终端跑两条命令，确认硬件型号和症状一致再继续：

1. **查看蓝牙硬件 ID**：
   ```bash
   lsusb
   ```
   如果输出中包含 `ID 13d3:3585 IMC Networks`（这是华硕天选/幻系列、联想 ThinkBook/拯救者等主板和笔记本常见的 AzureWave 代工版 MT7922），说明硬件型号一致。

2. **查看内核报错日志**：
   ```bash
   sudo dmesg | grep -i bluetooth
   ```
   如果看到类似下面的超时报错（错误码 `-110`）：
   ```text
   Bluetooth: hci0: Opcode 0x0c03 failed: -110
   ```
   同时执行 `bluetoothctl list` 没有任何控制器输出，并且在 Windows 下蓝牙功能完全正常（排除硬件物理损坏），那么本文的方案正好适用。

> [!IMPORTANT]
> **适用前提**：请确保主板 BIOS 中已关闭 **Secure Boot（安全启动）**。因为我们需要修改系统的蓝牙驱动文件，开启安全启动会导致系统强制校验厂商签名，从而拦截修改后的第三方驱动加载。

---

## 二、 为什么会出现这个问题？

### 2.1 说人话版

联发科 MT7922 的蓝牙芯片比较特殊，它本身不自带固化程序，每次开机通电时芯片内部都是“空白”的，必须靠 Linux 系统的蓝牙驱动（`btusb`）先往芯片里刷入一份微码固件（位于 `/lib/firmware/mediatek/` 下），它才能正常开始工作。

问题出在**驱动的“设备名单”漏掉了型号**：

Ubuntu 6.8 自带的蓝牙驱动里登记了同款芯片的老批次型号（例如 `13d3:3568`），但偏偏漏掉了较新的批次 `13d3:3585`。

结果就是：开机时系统识别到 `13d3:3585`，没认出它是联发科芯片，就把它当成普通的通用 USB 蓝牙适配器，直接跳过了“刷入固件”这一步，开始发送 HCI 复位指令。还没加载固件的裸芯片听不懂任何指令，系统干等 2 秒后超时报错 `-110`（Connection timed out）并彻底放弃初始化。

### 2.2 进阶极客版：Patch 的本质是什么？

在 Linux 内核驱动 `drivers/bluetooth/btusb.c` 中，支持的设备通过 `struct usb_device_id` 静态数组注册：

```c
/* 内核源码中的设备匹配结构 */
{ USB_DEVICE(0x13d3, 0x3568), .driver_info = BTUSB_MEDIATEK },
```

编译成二进制内核模块 `btusb.ko` 后：
* `0x0003`（匹配设备标志位 `USB_DEVICE_ID_MATCH_DEVICE`）在 Little-Endian 下为 `\x03\x00`
* `0x13d3`（Vendor ID）在小端序下存储为 `\xd3\x13`
* `0x3568`（旧款 Product ID）在小端序下存储为 `\x68\x35`

整个匹配头部特征就是连续的 6 个字节：`03 00 d3 13 68 35`。

我们不需要重新拉取内核源码编译，只需直接把这 6 个字节就地替换为新批次的特征 `03 00 d3 13 85 35`（即 `0x13d3:0x3585`），并抹除旧模块自带的数字签名尾缀，内核就会乖乖将 `13d3:3585` 识别为需要走 `BTUSB_MEDIATEK` 固件加载通道的网卡！

---

## 三、 完整修复步骤（手动执行）

### 第 1 步：备份原版驱动

修改系统模块前先留好退路，把原版驱动备份到主目录：

```bash
# 复制出当前内核正在使用的原版 btusb 模块
cp $(modinfo -n btusb) ~/btusb.ko.bak
cp ~/btusb.ko.bak /tmp/btusb_orig.ko
chmod 666 /tmp/btusb_orig.ko
```

### 第 2 步：替换硬件 ID 并移除失效签名

使用 Python 将驱动中的 `13d3:3568` 字节替换为 `13d3:3585`，并截断文件末尾原厂的签名附录（因为修改文件内容后原厂签名必定哈希校验失败）：

```bash
python3 -c '
d = open("/tmp/btusb_orig.ko", "rb").read()
# 替换设备匹配 ID: 13d3:3568 -> 13d3:3585
d = d.replace(b"\x03\x00\xd3\x13\x68\x35", b"\x03\x00\xd3\x13\x85\x35")
# 剥离末尾内核签名标记
s = b"~Module signature appended~\n"
d = d[:d.rfind(s)-28] if s in d else d
open("/tmp/btusb_clean.ko", "wb").write(d)
print("设备 ID 替换及签名清理完成！")
'

# 清理残留的调试信息段与 Linux 签名附注
sudo objcopy --remove-section=.note.Linux /tmp/btusb_clean.ko 2>/dev/null || strip --strip-debug /tmp/btusb_clean.ko
```

### 第 3 步：覆盖系统驱动并重新加载

将打好补丁的驱动复制回原系统目录，更新模块依赖索引，然后热重载蓝牙驱动：

```bash
sudo cp /tmp/btusb_clean.ko $(modinfo -n btusb)
sudo depmod -a
sudo modprobe -r btusb && sudo modprobe btusb
```

### 第 4 步：写入开机自动绑定别名

为了确保系统每次开机或接入 USB 设备时都能自动命中修改后的 `btusb` 驱动，添加一条 modprobe 别名规则并刷新 initramfs：

```bash
echo "alias usb:v13D3p3585d*dc*dsc*dp*ic*isc*ip*in* btusb" | sudo tee /etc/modprobe.d/btusb-3585.conf
sudo update-initramfs -u
```

---

## 四、 极简一键修复脚本（可选）

如果你有多台设备或者不想分步粘贴命令，也可以直接把以下脚本保存为 `fix_mt7922_bt.sh` 并运行：

```bash
#!/usr/bin/env bash
set -e

echo "=== 开始修复联发科 MT7922 (13d3:3585) 蓝牙驱动 ==="

TARGET_KO=$(modinfo -n btusb 2>/dev/null)
if [ -z "$TARGET_KO" ]; then
    echo "错误：未找到系统内置的 btusb 驱动模块！"
    exit 1
fi

echo "1. 备份原版驱动到 ~/btusb.ko.bak..."
cp "$TARGET_KO" ~/btusb.ko.bak
cp "$TARGET_KO" /tmp/btusb_orig.ko

echo "2. 修改设备 ID 并剥离失效数字签名..."
python3 -c '
d = open("/tmp/btusb_orig.ko", "rb").read()
d = d.replace(b"\x03\x00\xd3\x13\x68\x35", b"\x03\x00\xd3\x13\x85\x35")
s = b"~Module signature appended~\n"
d = d[:d.rfind(s)-28] if s in d else d
open("/tmp/btusb_clean.ko", "wb").write(d)
'
objcopy --remove-section=.note.Linux /tmp/btusb_clean.ko 2>/dev/null || strip --strip-debug /tmp/btusb_clean.ko

echo "3. 安装新驱动并刷新内核模块关系..."
sudo cp /tmp/btusb_clean.ko "$TARGET_KO"
sudo depmod -a
sudo modprobe -r btusb 2>/dev/null || true
sudo modprobe btusb

echo "4. 固化 modprobe 规则并更新 initramfs..."
echo "alias usb:v13D3p3585d*dc*dsc*dp*ic*isc*ip*in* btusb" | sudo tee /etc/modprobe.d/btusb-3585.conf >/dev/null
sudo update-initramfs -u

echo "=== 修复完成！请查看右上角蓝牙图标或运行 bluetoothctl list 验证 ==="
```

赋予执行权限后运行即可：
```bash
chmod +x fix_mt7922_bt.sh
./fix_mt7922_bt.sh
```

---

## 五、 验证是否修复成功

跑完上述步骤后，通常不需要重启电脑，右上角顶栏的蓝牙图标就会自动亮起。你也可以通过命令行确认：

1. **查看蓝牙控制器列表**：
   ```bash
   bluetoothctl list
   ```
   此时应该能正常列出控制器：
   ```text
   Controller XX:XX:XX:XX:XX:XX [default]
   ```

2. **查看内核固件初始化日志**：
   ```bash
   sudo dmesg | grep -i mediatek
   ```
   如果看到固件被成功加载并初始化：
   ```text
   Bluetooth: hci0: MediaTek: Device setup in ... usecs
   ```
   说明固件已经顺利刷入芯片内部，蓝牙扫描和耳机/外设配对已恢复正常。

---

## 六、 常见踩坑避雷（排查走过的弯路）

* **坑 1：以为是双系统“快速启动”导致的硬件锁死**  
  网上不少帖子提到联发科网卡（MT7921/MT7922）在 Windows + Linux 双系统下常因电源休眠锁死。但如果你在 Windows 中彻底关闭了“快速启动”、甚至拔电长按电源键放电 30 秒后，Linux 下依然报 `-110`，那就不是供电唤醒问题，而是纯粹的驱动设备表漏配。
* **坑 2：盲目下载最新官方固件替换 `/lib/firmware/mediatek/`**  
  驱动根本没把硬件认进联发科流程，连“寻找固件”这一步都还没触发，去折腾固件文件毫无意义。而且在 6.8 内核下强行换入过新的 git 主线固件，反而可能导致 Wi-Fi 驱动报结构体越界警告（`mt7921_load_clc` 报错）。保持 Ubuntu 自带的固件即可。
* **坑 3：改完驱动后用 `insmod` 手动加载报错 `Unknown symbol in module`**  
  `btusb` 依赖 `btrtl`、`btintel`、`btmtk` 等一系列基础模块。用 `insmod` 单独加载裸 `.ko` 时，系统不会自动加载它的上游依赖链。必须放回模块目录执行 `sudo depmod -a`，再用 `sudo modprobe btusb` 加载。
* **坑 4：直接修改驱动后报 `Key was rejected by service`**  
  即使关闭了 Secure Boot，如果驱动文件末尾依然保留着“已被篡改”的原版签名附录，内核加载时仍会校验出错。因此第 2 步中通过 Python 切掉 `~Module signature appended~` 结尾的操作必不可少。
* **坑 5：驱动加载成功但依然搜不到设备**  
  运行 `rfkill list`，检查 Bluetooth 是否处于软屏蔽（`Soft blocked: yes`）。如果是，执行 `sudo rfkill unblock bluetooth` 即可唤醒。

---

## 七、 维护说明：内核更新后的自愈与回滚

### 7.1 后续内核更新会失效吗？

* **内核小版本更新（例如 `6.8.0-40` -> `6.8.0-45`）**：  
  Ubuntu 在通过 `apt upgrade` 安装新的 6.8 小版本内核时，会附带官方全新的 `btusb.ko`，这会覆盖掉我们修改后的驱动。如果更新后蓝牙再次失效，只需重新运行一次上面的步骤（或一键脚本）即可。
* **内核大版本更新（Linux 6.11+）**：  
  在上游 Linux Kernel 6.11 及更新版本中，内核官方已经将 `13d3:3585` 合并到了 `btusb.c` 的支持列表中。如果将来升级到了高版本内核，该补丁就会自然退役。

### 7.2 如何完全回滚恢复原状？

如果需要恢复系统最初的状态，只需用备份文件覆盖回去并清理别名配置：

```bash
sudo cp ~/btusb.ko.bak $(modinfo -n btusb)
sudo rm -f /etc/modprobe.d/btusb-3585.conf
sudo depmod -a
sudo update-initramfs -u
```
