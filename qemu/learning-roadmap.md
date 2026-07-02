---
title: "QEMU 学习路线图 — 嵌入式工程师视角"
date: 2026-07-02
tags: [qemu, learning, roadmap, riscv, embedded]
status: notes
---

## 目标

能够使用 QEMU 为自研 RISC-V 32/64 芯片建模 SoC、IP、Board。

## 核心前提：QEMU 对嵌入式工程师来说是什么

QEMU 本质是 **指令集模拟器（ISS）+ 寄存器级外设模拟器**。

| 嵌入式概念 | QEMU 等价物 |
|-----------|------------|
| 芯片地址空间 | `MemoryRegion`（一个平坦的地址空间，通过 subregion 填充） |
| 外设寄存器 | `MemoryRegion` + `read()`/`write()` 回调函数 |
| 总线（AHB/APB/AXI） | `SysBus`（标准化 MMIO + IRQ 的连接框架） |
| 外设基地址 | `sysbus_mmio_map(dev, 0, base_addr)` |
| 中断线 | `qemu_irq` + `sysbus_connect_irq()` |
| 设备树 | `create_fdt()` → 生成传给内核的 FDT blob |
| 硬件手册的 memory map 表 | C 数组 `static const hwaddr memmap[]` |

**核心数据结构只有一个：MemoryRegion。** 理解它，就理解了 QEMU 的一半。

---

## 第一阶段：建立心智模型（2 个文件，~600 行代码）

### 第 1 步：读 `qemu/hw/riscv/spike.c`（270 行）
**最简可行的 RISC-V 机器。**

它只有 3 个东西：
- CPU（RISC-V HART）
- 内存（DRAM）
- CLINT（定时器 + 核间中断）
- HTIF（Host-Target Interface，简单的输入输出通道）

**要观察的点：**
1. `spike_memmap[]` — 硬件手册的 memory map 变成 C 数组
2. `spike_board_init()` — Machine 的入口函数，做了什么？
3. `memory_region_add_subregion()` — 把 RAM/ROM 放进地址空间
4. `riscv_aclint_*_create()` — 外设如何创建并映射
5. 最后几行的 `TypeInfo` + `type_init()` — 如何注册 "spike" 这个机器

**关键认知：Machine 就是一个 init 函数，往地址空间里塞 MemoryRegion。**

### 第 2 步：读 `qemu/hw/riscv/sifive_e.c`（310 行）
**第一个真正的嵌入式 SoC。**

比 spike 多了：
- PLIC（平台级中断控制器）— 外设中断怎么接到 CPU
- UART ×2 — 真正的 MMIO 外设
- GPIO、AON（Always-On）— 自研 IP 的例子
- SoC 对象（`sifive_e_soc_realize`）— SoC 和 Machine 的分离

**要观察的点：**
1. `sifive_e_memmap[]` — 比 spike 更多外设，地址更密集
2. `sifive_e_soc_realize()` — SoC 层如何实例化所有外设
3. `sifive_uart_create()` — 通用外设的创建（一行搞定）
4. `create_unimplemented_device()` — 还没实现的 IP 如何占位
5. `sysbus_connect_irq()` — 中断如何从外设连到 PLIC
6. 复位向量的手工构造 — `rom_add_blob_fixed_as()`

**关键认知：SoC realize = 遍历 memmap，逐个外设 create + map + connect_irq。Machine init = 选 SoC + 加板级器件。**

---

## 第二阶段：理解 QOM（QEMU Object Model）

QEMU 用 C 实现了一套面向对象系统。记住以下规律即可：

| QOM 概念 | 对应 C 代码 |
|----------|-----------|
| 类型注册 | `static const TypeInfo xxx_type_info = { .name=..., .parent=..., .instance_size=..., .class_init=... }` |
| 构造函数（init） | `instance_init` → 创建子对象（`object_initialize_child`），不分配资源 |
| 构造函数（realize） | `DeviceClass->realize` → 分配资源、映射 MMIO、连线中断 |
| 创建对象 | `object_new(TYPE)` 或 `object_initialize_child()` |
| 使能对象 | `qdev_realize()` 或 `sysbus_realize()` |
| 属性（property） | `object_property_set_*()` 在 realize 前配置 |

**为什么要分 init 和 realize？** 
- init 阶段：创建对象树（CPU、UART、I2C 等），用户可以配置属性
- realize 阶段：真正分配宿主机资源。一旦 realize 就不能再改属性了

**文件参考：**
- `include/qom/object.h` — QOM 核心 API
- `hw/core/qdev.c` — Device 基类
- `hw/arm/aspeed_ast2600.c` 的 `aspeed_soc_ast2600_init()` vs `aspeed_soc_ast2600_realize()` — 对比学习

---

## 第三阶段：理解 MemoryRegion（QEMU 的心脏）

`MemoryRegion` 代表地址空间中的一块区域。不同类型对应不同行为：

| 类型 | 创建函数 | 行为 | 用途 |
|------|---------|------|------|
| RAM | `memory_region_init_ram()` | 可读写，宿主机分配真实内存 | 模拟 SRAM/DRAM |
| ROM | `memory_region_init_rom()` | 只读 | 模拟 Mask ROM / Flash |
| IO | `memory_region_init_io()` | 读/写触发回调函数 | 外设寄存器 |
| Alias | `memory_region_init_alias()` | 指向另一个 MR 的子区域 | PCIe MMIO 窗口 |
| Container | `memory_region_init()` | 纯容器，包含子 region | 总线地址空间 |

**每个外设本质上就是一个 IO 类型的 MemoryRegion + read/write 回调。**

```c
// 外设 = MemoryRegion + ops
static const MemoryRegionOps my_uart_ops = {
    .read  = my_uart_read,    // CPU 读外设寄存器时调用
    .write = my_uart_write,   // CPU 写外设寄存器时调用
    .endianness = DEVICE_NATIVE_ENDIAN,
};

memory_region_init_io(&s->iomem, obj, &my_uart_ops, s, "my-uart", 0x1000);
//                                                          name     size
// 这个 MR 占用 0x1000 字节地址空间
// CPU 访问这个范围时 → QEMU 调用 my_uart_read/my_uart_write
```

**文件参考：**
- `include/exec/memory.h` — MemoryRegion API（最重要的头文件之一）
- `softmmu/physmem.c` — 物理内存管理实现
- `hw/char/sifive_uart.c` — 看一个真实外设如何用 MemoryRegion

---

## 第四阶段：理解 SysBus

SysBus 是 QEMU 对传统 SoC 总线（AMBA/AHB/APB）的抽象。提供两样东西：
1. **MMIO 映射** — 把外设的寄存器放到地址空间的某个位置
2. **IRQ 连线** — 把外设的中断输出连到中断控制器的输入

**任何 SoC 内部的外设都是 SysBusDevice。**

```
          SysBusDevice
          ├── MemoryRegion (寄存器)  ──map──→  SoC 地址空间
          └── qemu_irq[] (中断输出)  ──connect──→ PLIC/INTC
```

**文件参考：**
- `include/hw/sysbus.h`
- `hw/core/sysbus.c`

---

## 第五阶段：动手写一个 MMIO 设备

**这是最关键的一步。** 写一个最简单的外设，只有两个寄存器：

```c
// my-gpio.c — 最简单的 SysBus MMIO 设备
#define TYPE_MY_GPIO "my-gpio"

struct MyGPIOState {
    SysBusDevice parent;     // 固定：继承 SysBusDevice
    MemoryRegion iomem;      // 寄存器空间
    uint32_t data_out;       // 输出寄存器 @ offset 0x00
    uint32_t data_in;        // 输入寄存器  @ offset 0x04
};

// CPU 读 GPIO 寄存器时调用
static uint64_t my_gpio_read(void *opaque, hwaddr addr, unsigned size) {
    MyGPIOState *s = opaque;
    switch (addr) {
    case 0x00: return s->data_out;
    case 0x04: return s->data_in;
    default:   return 0;
    }
}

// CPU 写 GPIO 寄存器时调用
static void my_gpio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size) {
    MyGPIOState *s = opaque;
    switch (addr) {
    case 0x00: s->data_out = val; break;
    // data_in 是只读的，不处理
    }
}

static const MemoryRegionOps my_gpio_ops = {
    .read  = my_gpio_read,
    .write = my_gpio_write,
    .endianness = DEVICE_NATIVE_ENDIAN,
    .valid = { .min_access_size = 4, .max_access_size = 4 },
};

// init: 创建 MemoryRegion
static void my_gpio_init(Object *obj) {
    MyGPIOState *s = MY_GPIO(obj);
    memory_region_init_io(&s->iomem, obj, &my_gpio_ops, s, TYPE_MY_GPIO, 0x1000);
    sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->iomem);  // 导出给 SoC 用
}

// realize: 初始化状态
static void my_gpio_realize(DeviceState *dev, Error **errp) {
    MyGPIOState *s = MY_GPIO(dev);
    s->data_out = 0;
    s->data_in  = 0xDEADBEEF;  // 模拟输入值
}

static const TypeInfo my_gpio_info = {
    .name          = TYPE_MY_GPIO,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(MyGPIOState),
    .instance_init = my_gpio_init,
    .class_init    = my_gpio_realize,  // 注意: class_init 但赋值给 DeviceClass->realize
};
type_init(my_gpio_register_types);
```

**练习方法：** 把这段代码放进 `hw/misc/my-gpio.c`，在 `sifive_e_soc_realize()` 里加上它，编译，用 `-M sifive_e` 启动，通过 GDB 读写 0x10012000 看效果。

---

## 第六阶段：理解 I2C 设备

已经知道 sensor 是 I2C 设备挂上去的。理解 I2C 框架：

```
I2CBus                           ← 控制器（SoC 内部，如 aspeed_i2c）
├── I2CSlave @ 0x48 (TMP105)     ← 温度传感器
├── I2CSlave @ 0x4c (TMP105)
└── I2CSlave @ 0x50 (EEPROM)

Guest 通过 I2C 控制器寄存器发起 I2C 传输
  → QEMU I2C 框架路由到对应地址的 I2CSlave
    → I2CSlave 的 .send()/.recv() 被调用
```

**I2CSlave 不是 SysBusDevice**，它不直接出现在地址空间里。它通过 I2C 总线被间接访问。

**文件参考：**
- `include/hw/i2c/i2c.h` — I2CSlave 接口
- `hw/sensor/tmp105.c` — 看 `tmp105_read()` / `tmp105_write()` 怎么处理 I2C 寄存器
- `hw/i2c/aspeed_i2c.c` — I2C 控制器端

---

## 第七阶段：RISC-V 平台特有的知识

| 组件 | 作用 | QEMU 位置 |
|------|------|----------|
| CLINT | 核内定时器 + 软件中断 | `riscv_aclint_swi_create()` / `riscv_aclint_mtimer_create()` |
| PLIC | 平台级中断控制器 | `sifive_plic_create()` |
| AIA (APLIC+IMSIC) | 新一代中断控制器 | `hw/intc/riscv_aplic.c` / `hw/intc/riscv_imsic.c` |
| HART | RISC-V CPU 核 | `target/riscv/cpu.c`, `hw/riscv/riscv_hart.c` |
| 复位向量 | 上电后 PC 跳到哪里 | `riscv_setup_rom_reset_vec()` |
| DeviceTree | 给内核传递硬件信息 | `create_fdt()` → `riscv_load_fdt()` |
| Firmware | OpenSBI / U-Boot SPL | `riscv_load_firmware()` |

---

## 第八阶段：规划你自己的 SoC + Board

到此你已经理解了全部必要概念。实际建模你自己的芯片时，流程是：

```
1. 硬件手册 → Memory Map 表 → memmap[] 数组
2. 硬件手册 → IRQ 路由表   → irqmap[] 数组  
3. 选 CPU（riscv32 or riscv64）
4. 评估外设：
   - 通用 IP（UART/SPI/I2C）→ QEMU 可能已有
   - 标准 RISC-V IP（CLINT/PLIC）→ QEMU 已有
   - 自研 IP → 需要自己写（模板：第五阶段）
   - 不急着实现的 → UNIMPLEMENTED 占位
5. 写 SoC 的 init + realize
6. 写 Board 的 machine_init
7. 写 Kconfig + meson.build
8. 编译，用 bare metal 程序测试
```

**作为芯片原厂工程师，你的核心竞争力在于：硬件手册到模型映射的准确性和速度。**

---

## 关键源码索引

| 学习内容 | 文件 | 行数 |
|---------|------|------|
| 最简 RISC-V 机器 | `hw/riscv/spike.c` | 385 |
| 嵌入式 SoC 范本 | `hw/riscv/sifive_e.c` | 310 |
| 中大型 SoC 范本 | `hw/arm/aspeed_ast2600.c` | ~600 |
| MemoryRegion API | `include/exec/memory.h` | — |
| SysBus API | `include/hw/sysbus.h` | — |
| QOM 核心 | `include/qom/object.h` | — |
| 简单 MMIO 设备 | `hw/char/sifive_uart.c` | 参考 |
| I2C 传感器 | `hw/sensor/tmp105.c` | 260 |
| 复位向量 | `hw/riscv/boot.c` | — |

## 阅读顺序建议

```
spike.c → sifive_e.c → tmp105.c → sifive_uart.c
    ↓
aspeed_ast2600.c（init vs realize 对比）
    ↓
include/exec/memory.h（浏览 API，不细读）
    ↓
自己写 my-gpio.c → 集成到 sifive_e
    ↓
开始建模你自己的芯片
```
