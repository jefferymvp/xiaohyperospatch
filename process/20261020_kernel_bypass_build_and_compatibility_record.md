# 小米 10 (umi) 旁路充电内核适配与构建过程归档复盘

- **归档时间**：2026-10-20 15:51
- **目标设备**：Xiaomi 10 (`umi` / 骁龙 865 / SM8250)
- **目标系统**：官方 HyperOS / MIUI 14 (Android 13 / Linux 4.19.157)
- **核心成果**：完成自动化 GitHub Actions 构建、消除所有编译器及链接报错、规范化 AnyKernel3 打包机制，实现硬件级旁路供电与官方原厂闭源驱动 100% 兼容。

---

## 一、问题背景与初期排查

用户在刷入基于 LineageOS 20 源码编译的小米 10 内核后，频繁出现以下三大严重问题：
1. **刷入后开机直接崩溃进 Recovery**；
2. **链接 `vmlinux` 阶段报重复符号（duplicate symbol）与未定义符号（undefined symbol）**；
3. **Recovery 刷入时报 `unable to determine partition, aborting`**。

---

## 二、关键技术问题根因与修复过程

### 1. 致命原因 1：`build-kernel.yml` 覆盖 defconfig 与打包 LineageOS DTB
- **根因分析**：
  - workflow 原脚本在执行 `cp hyperos_umi_defconfig ...` 之后，又执行了 LineageOS 的 `merge_config.sh`，导致官方配置被覆盖并开启了 `CONFIG_MODVERSIONS=y`；
  - workflow 打包了 LineageOS 源码编译出的 `kona*.dtb`，AnyKernel3 刷入时替换了手机 boot.img 内的官方 DTB，导致硬件树与机身官方 `dtbo.img` / `vendor.img` 严重冲突，开机即死。
- **解决方案**：
  - 强制优先加载 `hyperos_umi_defconfig`；
  - 剔除 AnyKernel3 中的 `dtb` 文件，利用 AnyKernel3 的 `dump_boot` 原地保留手机官方原生 DTB。

### 2. 致命原因 2：触控驱动重复符号冲突 (`duplicate symbol`)
- **根因分析**：
  - 在 `hyperos_umi_defconfig` 中同时配置了 `CONFIG_TOUCHSCREEN_ST=y` 与 `CONFIG_TOUCHSCREEN_ST_FTS_V521=y`；
  - 两个驱动均包含未加 `static` 的 `fts_lib` 库（如 `requestCompensationData`、`calculateCRC8`、`checkEcho`），导致链接器报重名错误。
- **解决方案**：
  - 将 `CONFIG_TOUCHSCREEN_ST=y` 禁用，仅保留小米 10 实际使用的主流驱动 `CONFIG_TOUCHSCREEN_ST_FTS_V521=y`。

### 3. 致命原因 3：USB 驱动缺少 `ps5169_cfg_usb` 未定义符号
- **根因分析**：
  - LineageOS 源码中的 `drivers/usb/dwc3/dwc3-msm.c` 依赖了小米 10 的 PS5169 USB Redriver 芯片；
  - defconfig 中误禁用了 `# CONFIG_PS5169 is not set`。
- **解决方案**：
  - 开启 `CONFIG_PS5169=y`，编译该芯片驱动补齐符号。

### 4. 致命原因 4：AnyKernel3 变量小写导致分区无法识别
- **根因分析**：
  - 脚本中使用了小写的 `block=/dev/block/...` 和 `is_slot_device=0`；
  - AnyKernel3 内部仅读取大写变量 `$BLOCK`，小写导致 `$BLOCK` 为空，最终抛出 `Unable to determine partition. Aborting...`。
- **解决方案**：
  - 改为官方标准的通用大写配置：`BLOCK=boot;` 和 `IS_SLOT_DEVICE=auto;`，支持自动多路径智能嗅探。

---

## 三、硬件旁路供电工作原理
小米 10（PM8150B + SMB1390）通过 NVDC 电源路径管理，在驱动层向 `chg_disable_votable` 发送投票切断 BATFET 电池开关，保持外部 Buck 降压变换器直供系统总线 VSYS。电池电流归零（0mA），不充不放，发热骤减，实现硬件级旁路供电。
