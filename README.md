# 小米 10 (umi) 官方 HyperOS / MIUI 旁路充电内核适配指南

## 1. 背景与核心问题
在小米官方 HyperOS / MIUI 14 系统中：
- 官方内核版本为 Linux 4.19.157-perf-g8e1dce01cadc；
- 系统启动高度依赖 /vendor/lib/modules/ 下的闭源驱动模块（包含 Wi-Fi、触屏、声卡、电荷泵、显示等核心驱动）；
- 原版内核在开启普通停充时会挂起整个 USB 供电，导致 ACCA 提示“内核不支持电池闲置模式 (Battery Idle)”。

## 2. 核心补丁方案
1. **真实官方配置同步**：
   从真机工作镜像 magisk_patched-30700_Ujpkw.img 逆向提取真实完整的 1914 项官方内核配置，生成 hyperos_umi_defconfig。
2. **模块强制放行（HyperOS 兼容补丁）**：
   - 关闭 CONFIG_MODULE_SIG（允许无官方私钥加载）；
   - 关闭 CONFIG_MODVERSIONS（跳过符号 CRC 校验）；
   - 修复 kernel/module.c 的 same_magic 校验，允许原厂模块顺利加载。
3. **真实硬件旁路供电补丁**：
   - 在 drivers/power/supply/qcom/qpnp-smb5.c 和 attery.c 中接入 chg_disable_votable；
   - 保持 VBUS 外部 5V/9V 直供主板，切断电池充电通路；
   - 上报 POWER_SUPPLY_STATUS_NOT_CHARGING，让 ACCA 完美进入 Idle / Discharging: 0mA 电池闲置状态。
