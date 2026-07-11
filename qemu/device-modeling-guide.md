---
title: "外设建模复杂度分级与特殊外设指南"
date: 2026-07-09
tags: [qemu, device-emulation, pinctrl, clock, phy, gpio, dma, reference]
status: notes
---

## 背景

芯片原厂工程师用 QEMU 建模自己 SoC 的外设。绝大多数外设是纯寄存器桩（Guest 写什么存什么、读什么返回什么），但有少数外设需要额外的副作用逻辑。

---

## 外设三档分级

### 第一档：纯寄存器桩（~90% 的外设）

Guest 读写寄存器 → 存值、返回值。**没有副作用需要模拟。**

| 外设 | 代码量 | 注意点 |
|------|--------|--------|
| pinctrl | 30 行 | 完全桩，存值就行 |
| clock ctrl | 40 行 | 复位默认值要对（分频系数、时钟源选择）。算频率是 Guest 驱动自己做的事，不在 QEMU 里算 |
| syscon / SCU | 30 行 | 芯片 ID、版本号等只读寄存器值要对 |
| power / PMU | 20 行 | 纯寄存器，写写读读 |
| reset ctrl | 20 行 | 纯寄存器 |
| OTP / eFuse | 20 行 | 只读区域的值要预先写好（序列号、校准值） |
| watchdog | — | QEMU 有现成框架 `hw/watchdog/`，直接用 |
| I2C 控制器 | 150 行 | QEMU 有 I2C 框架 `hw/i2c/`，接上就行 |
| SPI 控制器 | 150 行 | QEMU 有 SSI 框架 `hw/ssi/`，接上就行 |
| DMA | 80 行 | 驱动用了 DMA 的话，至少做个"写完立刻完成"的桩（见下文） |
| PHY | 20 行 | 嵌在 MAC 里，几个标准寄存器值要对（见下文） |

### 第二档：有回调副作用（~8%）

寄存器的读写**会触发外部行为**——通知 chardev、触发中断、驱动 GPIO。

| 外设 | 额外要做什么 | 代码量 |
|------|-------------|--------|
| GPIO | output 写 → 触发中断；input 读 → 返回外部输入值 | 100 行 |
| PWM | 如果只用占空比 → 桩；如果驱动用 PWM 中断 → 需要定时器回调 | 80 行 |
| ADC | 返回预设模拟值，QEMU 可外部注入 | 80 行 |

### 第三档：必须精确模拟（~2%）

**不模拟就跑不起来**——直接影响 CPU 执行流程。

| 外设 | 为什么必须精确 | 做法 |
|------|---------------|------|
| 中断控制器 (PLIC/APLIC) | 没有中断，系统卡死 | 用 QEMU 已有的，别自己写 |
| 定时器 (CLINT/ACLINT) | OS 调度依赖定时器中断 | 用 QEMU 已有的 |
| 看门狗 | 乱复位或不复位都会导致异常 | QEMU 有框架 |

---

## 特殊外设详解

### 一、pinctrl — 纯桩

**为什么纯桩就够了？** QEMU 没有物理引脚，不需要实际切换连线。驱动写 `MUX[5]=UART` 只是存值，读回来返回存的值，驱动就满意了。

```c
// pinctrl = 寄存器文件
static void pinctrl_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    s->regs[addr / 4] = val;  // 只存值，不做实际路由
}
```

### 二、clock ctrl — 大部分桩，频率值要对

**算频率是 Guest 驱动的事，QEMU 不代劳。** Guest 驱动会读 clock 控制器的分频寄存器，自己代公式算：

```c
// Guest Linux 驱动：
div = readl(CLK_CTRL1);          // 读 QEMU 存的寄存器值
rate = CPU_FREQ / (div + 1);     // 自己算
```

**QEMU 只负责把寄存器存对、读对。** 关键：`realize` 时按硬件手册初始化分频系数等的复位默认值。

### 三、PHY — 嵌在 MAC 里，5 个寄存器值要对

QEMU 有完整的 MII 标准寄存器宏（`include/hw/net/mii.h`），常见 PHY 的 Vendor ID 也都定义好了：

```c
#define DP83848_PHYID1  0x2000  // 你板子上可能用这个
#define DP83848_PHYID2  0x5c90
#define RTL8211E_PHYID1 0x001c
```

#### QEMU 不需要独立的 PHY 设备

PHY 永远通过 MDIO 被 MAC 间接访问——Guest 看不到 PHY 的总线地址，只能写 MAC 的 MDIO 寄存器来读写 PHY。所以 PHY 寄存器直接嵌在 MAC 结构体里：

```c
typedef struct {
    SysBusDevice parent;
    // ... MAC 自己的寄存器 ...
    uint16_t phy_regs[32];   // ★ 内部 PHY 寄存器
} MyMACState;
```

#### 关键：reset 时填对值

```c
static void my_mac_reset(MyMACState *s)
{
    s->phy_regs[MII_BMCR]  = MII_BMCR_AUTOEN | MII_BMCR_FD;
    s->phy_regs[MII_BMSR]  = MII_BMSR_100TX_FD | MII_BMSR_AUTONEG |
                              MII_BMSR_LINK_ST | MII_BMSR_AN_COMP;
                              // ★ 永远报告 link up + 自协商完成
    s->phy_regs[MII_PHYID1] = DP83848_PHYID1;  // 你的 PHY 型号
    s->phy_regs[MII_PHYID2] = DP83848_PHYID2;
    s->phy_regs[MII_ANAR]   = MII_ANAR_TXFD | MII_ANAR_TX | ...;
    s->phy_regs[MII_ANLPAR] = MII_ANLPAR_TXFD | MII_ANLPAR_TX | ...;
}
```

#### 为什么 `BMSR_LINK_ST` 必须始终置 1？

Guest 驱动读 BMSR 检查 bit 2 (link status)：
- 0 → 驱动认为"网线没插"，不用这个 MAC
- 1 → 驱动认为链路正常，继续初始化

QEMU 里没有物理网线，所以永远返回 link up。

#### MDIO 读写链路

```
Guest 驱动:
  writel(MAC_MDIO_CMD, READ | phy_addr=1 | phy_reg=2)
       │
       ▼
QEMU MAC write: 检测到读 PHYID1
       │
       ▼
  s->mdio_data = s->phy_regs[2]  → 返回 0x2000
       │
       ▼
Guest 驱动: val = readl(MAC_MDIO_DATA) → "PHY 是 DP83848"
```

### 四、DMA — 不能纯桩，要"立刻完成"

如果 BSP 用了 DMA（UART DMA、SPI DMA 等），驱动会等 DMA 完成中断。不触发中断就会卡死。

```c
// 最简 DMA 桩：写启动寄存器 → 立刻触发完成中断
static void dma_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    switch (addr) {
    case DMA_CTRL:
        if (val & DMA_START) {
            // 假装已经搬完，触发完成中断
            s->int_status |= DMA_INT_DONE;
            qemu_set_irq(s->irq, 1);
            qemu_set_irq(s->irq, 0);  // 脉冲
        }
        break;
    }
}
```

### 五、GPIO — 需要中断联动

```c
static void my_gpio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    switch (addr) {
    case GPIO_DATA_OUT:
        s->data_out = val;           // 存寄存器值
        // 副作用：检查哪些位变化了，触发中断
        if (val & s->int_en & ~s->prev) {  // 上升沿
            s->int_status |= (val & s->int_en & ~s->prev);
            qemu_set_irq(s->irq, 1);
        }
        s->prev = val;
        break;
    }
}
```

### 六、中断控制器和定时器 — 用 QEMU 已有的

别自己写。RISC-V 平台：
- PLIC：`sifive_plic_create(...)`
- APLIC/IMSIC：`riscv_create_aia(...)`
- CLINT：`riscv_aclint_swi_create(...)` + `riscv_aclint_mtimer_create(...)`

---

## 三种"副作用"的本质

```
副作用类型        使用的 API                例子
────────────────  ────────────────────────  ──────────────────
通知 chardev      qemu_chr_fe_write()       UART 输出字符
触发中断          qemu_set_irq(irq, 1/0)    GPIO 引脚变化→PLIC→CPU
操作 GPIO 引脚    gpio_get/set_level()      一个设备读另一个 GPIO
```

都是在 `write` 回调里多调一个 API 的事。纯桩设备只有 `s->reg = val`，有副作用的设备多一行 `qemu_set_irq(...)` 或 `qemu_chr_fe_write(...)`。

---

## 建模优先级建议

```
第一遍：让 BSP 跑起来
  中断控制器（QEMU 已有）→ 定时器（QEMU 已有）→ UART
  → 其余全部 create_unimplemented_device() 占坑

第二遍：按需补桩
  驱动初始化时需要什么外设的什么寄存器返回什么值 →
  按硬件手册补成纯寄存器桩

第三遍：精确行为
  驱动功能验证需要精确行为 → 加中断联动、DMA 完成、PHY link up
```
