---
title: "QEMU 传感器模拟"
date: 2026-07-02
tags: [qemu, sensor, i2c, device-emulation]
status: notes
---

## 背景

想了解 QEMU 是否能模拟板卡上挂载的传感器（温度、气压、电压等），以及如何实现。

## 提问

1. QEMU 能模拟板卡上的 sensor 吗？
2. 支持哪些类型的传感器？
3. 传感器怎么挂到板卡上？Guest 怎么访问？

## 理解

### 架构

QEMU 有专门的 `hw/sensor/` 目录，所有传感器都作为 **I2C 从设备** 实现。Guest OS 通过 I2C 总线驱动读写传感器寄存器，与真实硬件无异。

### 支持的传感器

| 型号 | 类型 | 接口 |
|------|------|------|
| TMP105/75 | 温度传感器 | I2C |
| TMP421/422/423 | 多通道温度传感器 | I2C |
| EMC1412/13/14 | 多通道温度传感器 | I2C |
| DPS310 | 气压传感器 | I2C |
| LSM303DLHC | 磁力计 | I2C |
| ADM1266 | PMBus 电源监控 | I2C/PMBus |
| ADM1272 | PMBus 热插拔控制器 | I2C/PMBus |
| MAX31785 | PMBus 风扇控制器 | I2C/PMBus |
| MAX34451 | PMBus 电源监控 | I2C/PMBus |
| ISL68220/21/24 | PMBus 电压调节器 | I2C/PMBus |

### 挂载方式

板卡初始化时，一行代码将传感器注册到 I2C 总线：

```c
i2c_slave_create_simple(aspeed_i2c_get_bus(&soc->i2c, 1),
                        TYPE_TMP105, 0x4c);
//                      总线       传感器型号   I2C从机地址
```

使用这些传感器的板卡（示例）：
- **Aspeed BMC** (`hw/arm/aspeed.c`)：使用了 TMP105、TMP421
- **Nuvoton NPCM7xx/8xx** (`hw/arm/npcm7xx_boards.c`)：温度传感器

### 温度传感器工作流程（以 TMP105 为例）

1. Guest 驱动通过 I2C 读写 TMP105 的寄存器（温度、配置、阈值）
2. TMP105 内部维护 `s->temperature` 值
3. 当温度超过阈值时，触发中断引脚 `qemu_set_irq(s->pin, ...)`
4. 温度值可通过 QMP/qom-set 从外部修改（用于测试）

## 要点

- 所有传感器都是 I2C 从设备，放在 `hw/sensor/` 目录
- 挂载方式统一：`i2c_slave_create_simple(bus, TYPE, addr)`
- 模拟的是 **寄存器级** 行为，Guest 驱动无需修改
- PMBus 传感器有专门的基类 `hw/i2c/pmbus_device.h`，提供 PMBus 协议框架
- 温度值可通过 QMP 动态修改（适合自动化测试）

## 参考

- 源码目录: `hw/sensor/`
- 头文件: `include/hw/sensor/`
- 使用示例: `hw/arm/aspeed.c`（搜索 `TYPE_TMP105`）
- I2C 框架: `hw/i2c/core.c`
