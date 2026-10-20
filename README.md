# 小米 10 (umi) 官方 HyperOS / MIUI 旁路供电内核项目

本项目为小米 10（代号：`umi`，高通骁龙 865 / SM8250 平台）在官方 HyperOS / MIUI 14（Android 13 / Linux 4.19）底包下，实现**硬件级旁路供电（电源直供 / 电池闲置模式）**并完美兼顾闭源原厂驱动模块的内核修补与自动化构建方案。

---

## 一、硬件级旁路供电原理

### 1. 硬件拓扑架构（NVDC 动态电源路径管理）
小米 10 搭载高通 **PM8150B + SMB1390** 电源管理芯片架构，硬件底层原生支持 NVDC（Narrow Voltage DC）路径管理：
```text
   外部充电器 (USB 5V/9V/QC)
              │
              ▼
    [ Buck 降压变换器 ]
              │
              ├────────────────────────► [ 系统总线 VSYS ] (直供 SoC、屏幕、射频模组)
              │                               ▲
              ▼                               │
       [ BATFET 电子开关 ]                    │ (放电隔离)
              │                               │
              ▼                               │
         [ 电池电芯 ] ─────────────────────────┘
```
- **旁路状态**：通过向驱动层投票 `chg_disable_votable`，PMIC 硬件开关（BATFET）断开通向电池电芯的充电回路，同时保持外部 USB 输入转换器全功率运作。
- **运行表现**：外部电源直接供给手机主板（VSYS），电池电流归零（0mA），不充也不放，高负载/长时间游戏发热骤减，彻底消除电池循环损耗。

---

## 二、核心补丁与实现方案

### 1. 真实原厂配置提取与重构 (`hyperos_umi_defconfig`)
- 从官方工作镜像逆向提取全套官方内核符号配置，精准保留小米私有服务（MIHW、MIGT、MILLET 等）支持。

### 2. 电源驱动补丁 (`drivers/power/supply/qcom/`)
- 在 `battery.c`、`qpnp-smb5.c` 和 `smb5-lib.c` 中打通 `chg_disable_votable`；
- 开放标准 sysfs 交互节点：
  - `/sys/class/power_supply/battery/charging_enabled`
  - `/sys/class/qcom-battery/bypass_charging_enable`
- 精准状态反馈：处于旁路模式时向用户空间上报 `POWER_SUPPLY_STATUS_NOT_CHARGING`，让 ACC / ACCA 能够正确识别并进入 `Idle / Discharging: 0mA` 电池闲置状态。

### 3. 闭源驱动兼容与签名放行
- 修正 `Makefile` 内核版本标识为官方格式 `4.19.157-perf-g8e1dce01cadc`；
- 禁用 `CONFIG_MODULE_SIG`、`CONFIG_MODVERSIONS` 并启用 `CONFIG_MODULE_FORCE_LOAD`，彻底兼容 `/vendor/lib/modules/` 下的原厂闭源驱动（Wi-Fi、音频、电荷泵等）。

---

## 三、构建与适配关键踩坑指南（经验归档）

在适配与编译过程中，总结出以下 5 个关键技术痛点及根治方案：

### 1. 官方原生 DTB 保留机制（防止开机跳 Recovery）
- **现象**：若将源码编译出的 `kona*.dtb` 打包进 AnyKernel3，刷入后开机必定直接崩溃重启进 Recovery。
- **原因**：源码自带的设备树是针对 AOSP 系统的，与官方 HyperOS 的 `dtbo.img` / `vendor.img` 存在严重的屏幕面板和硬件节点冲突。
- **方案**：**AnyKernel3 刷机包中绝对不放任何 `dtb` 文件**。AnyKernel3 在本地刷机时会通过 `dump_boot` 实时解包手机当前运行的原厂 `boot.img`，自动提取并原封不动地保留官方原生 DTB，实现硬件树 100% 原生咬合。

### 2. 为什么严禁乱刷第三方的 `dtbo.img`？
- 小米官方 HyperOS 的 `mi_display` 面板驱动、色彩调优、FOD 屏下指纹高度绑定原厂 `dtbo` 分区。
- 本项目保持手机内部的 `dtbo` 分区原版不动，仅替换内核可执行文件 `Image`，确保屏幕、指纹、高刷功能完全正常。

### 3. 触控驱动符号重复冲突 (`duplicate symbol`)
- **现象**：链接 `vmlinux` 时报错 `ld.lld: error: duplicate symbol: requestCompensationData / calculateCRC8 / checkEcho ...`。
- **原因**：defconfig 中同时开启了旧版 `CONFIG_TOUCHSCREEN_ST=y` 与主流版 `CONFIG_TOUCHSCREEN_ST_FTS_V521=y`，两者共用一套未声明为 `static` 的 `fts_lib` 库导致重名。
- **方案**：关闭旧版 `CONFIG_TOUCHSCREEN_ST`，仅保留 `CONFIG_TOUCHSCREEN_ST_FTS_V521=y`。

### 4. 专有硬件未定义符号缺失 (`undefined symbol: ps5169_cfg_usb`)
- **现象**：链接最后报 `undefined symbol: ps5169_cfg_usb`（由 `dwc3-msm.c` 引用）。
- **原因**：小米 10 硬件搭载专有的 USB 3.0 Redriver（信号中继芯片），源码 USB 驱动显式调用了该芯片配置。
- **方案**：开启 `CONFIG_PS5169=y` 编译该驱动。

### 5. AnyKernel3 报错 `unable to determine partition aborting`
- **现象**：TWRP / OrangeFox 刷入时中断，提示无法识别分区。
- **原因**：`anykernel.sh` 中变量名误用了小写 `block=` 和 `is_slot_device=`，AnyKernel3 核心解析脚本仅读取大写变量 `$BLOCK`，导致读取为空。
- **方案**：配置全大写变量 `BLOCK=boot;` 和 `IS_SLOT_DEVICE=auto;`，AnyKernel3 会自动按优先级智能嗅探不同 Recovery 下真实的 boot 物理分区路径。

---

## 四、刷入与使用说明

### 1. 刷入方式
1. 前往本仓库的 **Releases** 页面下载最新构建生成的 `AnyKernel3-Mi10-BypassCharging-A13-umi-*.zip`；
2. 手机进入 TWRP / OrangeFox Recovery；
3. 直接刷入该 zip 文件（无需清除数据）；
4. 点击重启系统（Reboot System）。

### 2. 开启旁路供电
刷入成功后，可以通过以下任意方式控制旁路供电：

- **通过 ACC / ACCA 控制**：
  在控制选项中选择充电开关控制文件为：
  `/sys/class/power_supply/battery/charging_enabled`
  停充策略选择 `0`，即可完美进入 `Battery Idle` 电池闲置模式。

- **通过终端 Shell 手动控制**：
  ```bash
  # 开启旁路供电（切断电池充电，充电器直供主板）
  su -c "echo 0 > /sys/class/power_supply/battery/charging_enabled"

  # 恢复常规充电
  su -c "echo 1 > /sys/class/power_supply/battery/charging_enabled"
  ```
