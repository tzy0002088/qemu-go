---
title: "virt.c 源码分析 — RISC-V VirtIO Board"
date: 2026-07-08
tags: [qemu, riscv, virt, machine, soc, analysis]
status: notes
---

## 背景

跟着 [virt.c](../../my_qemu/hw/riscv/virt.c)（1807 行）学习 QEMU 的设计思想。这是 RISC-V 最完整的生产级机器。

## 文件结构（从下往上看）

```
type_init(virt_machine_init_register_types)     // 1806 — 注册入口
  ↓
virt_machine_typeinfo                           // 1786 — TypeInfo
  ├─ .class_init    = virt_machine_class_init    // 1718 — 设置 MachineClass 属性 + mc->init
  ├─ .instance_init = virt_machine_instance_init // 1551 — init 阶段：只创建 flash 对象
  ├─ .instance_finalize                         // 1538 — 清理
  └─ .instance_size = sizeof(RISCVVirtState)    // State 结构体
  ↓
virt_machine_init(MachineState *machine)        // 1302 — ★ 核心
  ↓
virt_machine_done(Notifier *notifier, ...)       // 1203 — 延后阶段
```

## 关键函数索引

| 函数 | 行号 | 作用 |
|------|------|------|
| `virt_machine_init_register_types` | 1801 | type_init 注册函数 |
| `virt_machine_typeinfo` | 1786 | QOM TypeInfo 定义 |
| `virt_machine_class_init` | 1718 | 设置 mc->init = virt_machine_init，注册属性 |
| `virt_machine_instance_init` | 1551 | init 阶段创建 flash 子对象 |
| `virt_machine_instance_finalize` | 1538 | 清理 flash 对象 |
| `virt_machine_init` | 1302 | **核心：Machine 初始化** |
| `virt_machine_done` | 1203 | 延后：FDT 收尾 + 加载固件 |
| `create_fdt` | 1005 | FDT 初始生成（flash/fw_cfg/pmu + 占位 pci 节点） |
| `finalize_fdt` | 979 | FDT 收尾（sockets/virtio/pcie/reset/uart/rtc） |
| `gpex_pcie_init` | 1036 | PCIe 主桥初始化 |
| `create_platform_bus` | 1139 | 动态 sysbus 设备平台总线 |
| `virt_create_plic` | 1118 | Per-socket PLIC 创建 |
| `create_fw_cfg` | 1108 | fw_cfg（固件配置接口） |

## virt_machine_init 五层结构

```
virt_machine_init(MachineState *machine)
  │
  ├─ ① 多 socket 初始化 (1328-1427)
  │     for each socket:
  │       object_initialize_child → RISCV_HART_ARRAY
  │       sysbus_realize(hart_array)          — 创建 CPU
  │       riscv_aclint_swi/mtimer_create()    — 定时器+软中断
  │       virt_create_plic() / riscv_create_aia() — 中断控制器
  │
  ├─ ② 内存映射 (1456-1463)
  │     memory_region_add_subregion(DRAM)
  │     memory_region_init_rom(mask_rom)
  │
  ├─ ③ 设备创建 (1469-1498)
  │     create_fw_cfg()
  │     sifive_test_create()          — 关机/重启寄存器
  │     virtio-mmio ×8               — VirtIO MMIO 传输
  │     gpex_pcie_init()             — PCIe 主桥
  │     create_platform_bus()        — 动态 sysbus 总线
  │     serial_mm_init()             — ns16550a UART
  │     goldfish_rtc                 — RTC
  │     virt_flash_map()             — 两块 PFlash
  │
  ├─ ④ 设备树初始生成 (1501-1509)
  │     create_fdt(s)
  │
  └─ ⑤ 注册延后回调 (1534-1535)
        qemu_add_machine_init_done_notifier(&s->machine_done)
```

## 两阶段初始化设计

virt.c 把初始化拆成两个阶段，这是它的核心设计模式：

```
Phase 1: virt_machine_init()
   所有硬件对象在这阶段创建、MMIO 在这阶段映射
   但此时 kernel/firmware 在哪还不知道

Phase 2: virt_machine_done()  (通过 Notifier 延后执行)
   此时所有选项已处理完，可以做最终决策：
     finalize_fdt()           — 补全 FDT 中依赖运行时信息的节点
     riscv_find_and_load_firmware() — 加载 OpenSBI
     riscv_load_kernel()      — 加载 Linux
     riscv_setup_rom_reset_vec() — 写复位向量
     virt_build_smbios()      — SMBIOS 表
```

**为什么拆成两阶段？** 阶段 1 时还不知道 kernel 会被加载到哪个地址、FDT 放哪里、固件用哪个。等到 `machine_done` 回调时一切已确定，再统一处理。

## Memory Map（86-108 行）

```c
static const MemMapEntry virt_memmap[] = {
    [VIRT_DEBUG]        = {        0x0,         0x100 },  // 调试
    [VIRT_MROM]         = {     0x1000,        0xf000 },  // Mask ROM (复位向量)
    [VIRT_TEST]         = {   0x100000,        0x1000 },  // 关机/重启
    [VIRT_RTC]          = {   0x101000,        0x1000 },  // Goldfish RTC
    [VIRT_CLINT]        = {  0x2000000,       0x10000 },  // 定时器
    [VIRT_ACLINT_SSWI]  = {  0x2F00000,        0x4000 },  // S-mode 软中断
    [VIRT_PCIE_PIO]     = {  0x3000000,       0x10000 },  // PCIe PIO
    [VIRT_IOMMU_SYS]    = {  0x3010000,        0x1000 },  // IOMMU
    [VIRT_PLATFORM_BUS] = {  0x4000000,     0x2000000 },  // 动态 sysbus
    [VIRT_PLIC]         = {  0xc000000,    ...        },  // PLIC
    [VIRT_APLIC_M]      = {  0xc000000,    ...        },  // AIA M-level APLIC
    [VIRT_APLIC_S]      = {  0xd000000,    ...        },  // AIA S-level APLIC
    [VIRT_UART0]        = { 0x10000000,         0x100 },  // UART
    [VIRT_VIRTIO]       = { 0x10001000,        0x1000 },  // VirtIO MMIO ×8
    [VIRT_FW_CFG]       = { 0x10100000,          0x18 },  // fw_cfg
    [VIRT_FLASH]        = { 0x20000000,     0x4000000 },  // PFlash ×2
    [VIRT_IMSIC_M]      = { 0x24000000,    ...        },  // AIA M-level IMSIC
    [VIRT_IMSIC_S]      = { 0x28000000,    ...        },  // AIA S-level IMSIC
    [VIRT_PCIE_ECAM]    = { 0x30000000,    0x10000000 },  // PCIe ECAM
    [VIRT_PCIE_MMIO]    = { 0x40000000,    0x40000000 },  // PCIe MMIO
    [VIRT_DRAM]         = { 0x80000000,           0x0 },  // DRAM（起始地址固定，大小由 -m 决定）
};
```

## 外设创建模式（标准三步）

virt.c 中 90% 的外设创建都是这三种模式：

```c
// 模式 1: sysbus_create_simple — 创建+realize+映射+连IRQ，一行搞定
sysbus_create_simple("virtio-mmio",
    base_addr,          // 基地址
    irq);               // 中断线

// 模式 2: qdev_new + 属性 + realize — 需要配置属性
DeviceState *dev = qdev_new(TYPE_GPEX_HOST);
object_property_set_uint(OBJECT(dev), PCI_HOST_ECAM_BASE, base, NULL);
object_property_set_int(OBJECT(dev), PCI_HOST_ECAM_SIZE, size, NULL);
sysbus_realize_and_unref(SYS_BUS_DEVICE(dev), &error_fatal);

// 模式 3: memory_region_add_subregion — 手动插入地址空间
memory_region_add_subregion(system_memory, addr, mr);
```

## FDT 生成（create_fdt → finalize_fdt）

FDT（Flattened Device Tree）分两次生成：

```
create_fdt(s)           // 1005 — 初始生成
  ├─ create_board_device_tree("riscv-virtio,qemu")
  ├─ 占位 pci 节点（热插拔需要）
  ├─ /chosen (rng-seed)
  └─ create_fdt_flash / fw_cfg / pmu
     ...
     (此时 FDT 还不完整，等待 finalize_fdt 补全)

finalize_fdt(s)         // 979 — 收尾生成（machine_done 中调用）
  ├─ create_fdt_sockets()    — CPU/内存节点 + CLINT/PLIC/APLIC 节点
  ├─ create_fdt_virtio()    — 8 个 virtio-mmio 节点
  ├─ create_fdt_pcie()      — PCIe 总线节点
  ├─ create_fdt_reset()     — 关机/重启节点
  ├─ create_fdt_uart()      — UART 节点
  └─ create_fdt_rtc()       — RTC 节点
```

## 中断控制器架构

virt.c 支持两种中断控制器，通过 `-M virt,aia=none|aplic|aplic-imsic` 切换：

```
PLIC 模式 (aia=none):
  PLIC ← 外设中断
         └─→ HART (M/S 外部中断)

AIA APLIC-IMSIC 模式 (aia=aplic-imsic):
  APLIC_M → IMSIC_M → HART (M 外部中断, MSI)
  APLIC_S → IMSIC_S → HART (S 外部中断, MSI)
  
  外设 → APLIC_S (wired 中断)
  外设 → IMSIC_S (MSI, 如 PCIe)
```

## 与 spike.c 对比

| | spike.c (270行) | virt.c (1807行) |
|---|---|---|
| 定位 | 最简 RISC-V 机器 | 完整生产级机器 |
| HART | RISCV_HART_ARRAY | RISCV_HART_ARRAY |
| 定时器 | CLINT (mtimer+swi) | ACLINT 或 CLINT |
| 中断 | 无 PLIC | PLIC 或 APLIC+IMSIC |
| 串口 | HTIF | ns16550a UART |
| 总线 | 无 | PCIe (GPEX) |
| 磁盘 | 无 | VirtIO MMIO ×8 |
| 固件存储 | 无 | PFlash ×2 |
| 设备树 | 简单 create_fdt | 完整 create_fdt + finalize_fdt |
| Socket | 多 socket 支持 | 多 socket NUMA |
| ACPI | 无 | ACPI + SMBIOS |
| IOMMU | 无 | RISC-V IOMMU + VirtIO IOMMU |

## 使用命令示例

```bash
# 基本启动
qemu-system-riscv64 -M virt -m 2G -smp 4 \
  -bios opensbi-fw_jump.bin \
  -kernel Image -append "console=ttyS0 root=/dev/vda"

# 调试：查看 QOM 树
qemu-system-riscv64 -M virt -smp 2 -m 1G -monitor stdio
(qemu) info qom-tree

# 调试：查看地址空间
(qemu) info mtree
```
