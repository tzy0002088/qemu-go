---
title: "Aspeed BMC 设备模拟全览"
date: 2026-07-02
tags: [qemu, aspeed, bmc, device-emulation, openbmc, i2c]
status: notes
---

## 背景

在了解 QEMU 传感器模拟时发现了 Aspeed 平台。想知道 QEMU 为 Aspeed 做了哪些硬件模拟、为什么这么全面。

## 提问

1. Aspeed 是什么？为什么 QEMU 投入这么多精力模拟它？
2. QEMU 模拟了 Aspeed 的哪些硬件 IP？
3. 哪些是完整实现，哪些还是 stub？

## 理解

### Aspeed 的角色

Aspeed 是一家做 **BMC（Baseboard Management Controller）** 芯片的公司。BMC 是服务器主板上独立于主 CPU 的小型管理芯片，负责：

- 远程开关机 / 重启
- 监控温度 / 电压 / 风扇
- 提供 SOL（Serial over LAN）串口重定向
- 挂载虚拟光驱（USB）
- 固件升级

即使服务器 CPU 完全挂了，BMC 仍然可以工作（独立供电）。

### QEMU 模拟的 SoC 家族

| SoC | CPU 核 | 代际 | 代表机型 |
|-----|--------|------|----------|
| AST2400 | ARM926EJ-S | 第1代 | Palmetto、Supermicro X11 |
| AST2500 | ARM1176 | 第2代 | Romulus、Witherspoon、YosemiteV2（最多机型） |
| AST2600 | Cortex-A7 | 第3代 | Rainier（IBM）、Fuji/Bletchley/Catalina（Facebook）、GB200NVL（NVIDIA） |
| AST2700 | Cortex-A35 | 第4代（最新） | EVB A0/A1 评估板 |
| AST1030 | Cortex-M4 | 精简版 | MiniBMC |

### 芯片内部 IP 实现完整度

#### 完整实现

| 类别 | 器件 | 源码位置 | 作用 |
|------|------|----------|------|
| 系统控制 | SCU | `hw/misc/aspeed_scu.c` | 时钟源 / 复位 / 引脚复用 / 芯片 ID |
| 内存 | SDMC | `hw/misc/aspeed_sdmc.c` | DDR SDRAM 控制器 |
| 中断 | VIC / INTC / GICv3 | `hw/intc/aspeed_*.c` | 中断控制器（不同代不同 IP） |
| SPI Flash | FMC + SPI×3 | `hw/ssi/aspeed_smc.c` | 固件存储（BMC 从这里启动） |
| SD/eMMC | SDHCI + eMMC | `hw/sd/aspeed_sdhci.c` | SD 卡 / eMMC |
| 以太网 | FTGMAC100×4 | 通用 MAC 驱动 | BMC 管理网口 |
| I2C | 多组 I2C 控制器 | `hw/i2c/aspeed_i2c.c` | 挂传感器/EEPROM/LED/风扇芯片 |
| I3C | I3C 控制器 | `hw/misc/aspeed_i3c.c` | 下一代 I2C（更高带宽） |
| PCIe | RC×3 | `hw/pci-host/aspeed_pcie.c` | PCIe 根端口 |
| PECI | PECI 控制器 | `hw/misc/aspeed_peci.c` | 读 Intel/AMD CPU 温度（单线协议） |
| FSI | FSI×2 | `hw/fsi/aspeed_apb2opb.c` | IBM POWER CPU 管理总线 |
| LPC | LPC 控制器 | `hw/misc/aspeed_lpc.c` | 与传统主机通信（KCS/BT 协议） |
| 定时器 | Timer×8 | `hw/timer/aspeed_timer.c` | 通用定时器 |
| 看门狗 | WDT×8 | `hw/watchdog/wdt_aspeed.c` | 看门狗定时器 |
| RTC | 实时时钟 | `hw/rtc/aspeed_rtc.c` | 实时时钟 |
| ADC | 模数转换器 | `hw/adc/aspeed_adc.c` | 读模拟电压（主板电源轨监控） |
| 加密 | HACE | `hw/misc/aspeed_hace.c` | 硬件哈希/加密引擎 |
| 安全启动 | SBC | `hw/misc/aspeed_sbc.c` | 安全启动控制器 |
| OTP | eFuse | `hw/nvram/aspeed_otp.c` | 一次性可编程存储 |
| GPIO | GPIO×2 | `hw/gpio/aspeed_gpio.c` | 通用 IO（控制 LED/电源/复位） |
| 串口 | UART×13 | 通用 16550 | 调试串口（**最重要的调试接口**） |
| USB | EHCI×4 | 通用 EHCI | USB 2.0（虚拟光驱/USB 网卡） |
| DMA | XDMA | `hw/misc/aspeed_xdma.c` | DMA 引擎 |
| SLI | SLI/SLIIO | `hw/misc/aspeed_sli.c` | LTPI 接口 |
| 协处理器 | FC/SSP/TSP | `aspeed_ast27x0-*.c` | AST2700 安全协处理器 |

#### 尚未实现（stub / UnimplementedDevice）

| 器件 | 标记 | 影响 |
|------|------|------|
| Video Engine | `UnimplementedDeviceState video` | 无法视频压缩（远程 KVM 的关键功能缺失） |
| PWM | `UnimplementedDeviceState pwm` | 无法模拟风扇调速 |
| eSPI | `UnimplementedDeviceState espi` | 不能用 eSPI 与主机通信（只能用 LPC） |
| DPMCU | `UnimplementedDeviceState dpmcu` | 电源管理 MCU 不可用 |
| UDC | `UnimplementedDeviceState udc` | USB Device 模式不可用 |
| SGPIOM | `UnimplementedDeviceState sgpiom` | 串行 GPIO 扩展不可用 |

### I2C 总线上挂载的外设（板级）

每款机型在 I2C 初始化函数中注册设备：

```c
// 例：ast2600_evb_i2c_init()
i2c_slave_create_simple(aspeed_i2c_get_bus(&soc->i2c, 1), "tmp105", 0x4c);
i2c_slave_create_simple(aspeed_i2c_get_bus(&soc->i2c, 1), "tmp105", 0x4e);
```

常用板级 I2C 器件：

| 器件 | 类型 | 用途 |
|------|------|------|
| TMP105 / TMP421 | 温度传感器 | 监控主板多位置温度 |
| EEPROM (24c64 等) | 存储 | FRU 数据（序列号、型号、厂商） |
| PCA9552 | LED 驱动 | 面板指示灯 |
| PCA9546 | I2C Mux | 一路扩多路 I2C |

### 为什么 QEMU 投入这么大精力？

1. **OpenBMC 开发需要** — OpenBMC 是 Linux 基金会项目（IBM/Facebook/Intel/Google 参与），QEMU 是主要开发/CI 平台
2. **无需真实硬件** — 固件开发者可以完整运行和调试 BMC 固件
3. **CI 自动化** — 每次提交自动启动 BMC 虚拟机跑测试
4. **传感器测试** — 通过 QMP 动态注入温度值，验证风扇控制策略
5. **安全测试** — 验证安全启动、加密引擎行为

## 要点

- QEMU Aspeed 模拟是 **BMC 固件开发的基石**，不是玩具
- Aspeed 模拟覆盖了 **4 代 SoC + 23 款真实服务器机型**
- 大多数硬件 IP 已完整实现，主要缺口是 Video Engine（远程 KVM 画面）和 PWM（风扇调速）
- 所有外设通过 I2C 总线挂载，模式统一：`i2c_slave_create_simple()`
- 启动命令范例：`qemu-system-arm -M ast2600-evb -drive file=obmc-phosphor-image.ast2600-evb.static.mtd,format=raw,if=mtd`

## 参考

- SoC 定义：`include/hw/arm/aspeed_soc.h`
- AST2600 实现：`hw/arm/aspeed_ast2600.c`
- 机型定义：`hw/arm/aspeed.c`
- OpenBMC 项目：https://github.com/openbmc
