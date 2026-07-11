---
title: "QEMU 整体架构框架"
date: 2026-07-08
tags: [qemu, architecture, overview, framework]
status: notes
---

## 背景

正式开始阅读 QEMU 源码前，先建立一个整体框架。知道"什么在什么地方"、"数据怎么流"，后续看具体代码时能快速定位。

## 一、QEMU 是什么 — 一句话

QEMU = **TCG（JIT 翻译 Guest 指令）+ 设备模型（MemoryRegion + 回调）+ 事件循环**

三个组件在一个进程里协同工作：
- TCG 把 Guest 的 RISC-V 指令翻译成 x86 指令执行
- 当 Guest 访问外设寄存器时，触发 MemoryRegion 回调
- 事件循环处理定时器、IO、QMP 命令等

---

## 二、进程的三种"身份"

| 视角 | 做什么 | 对应目录 |
|------|--------|---------|
| 虚拟 CPU | 取指、翻译、执行 Guest 指令 | `target/` + `tcg/` |
| 虚拟硬件 | 响应 Guest 的 MMIO 访问、触发中断 | `hw/` |
| 管理程序 | 解析命令行、创建虚拟机、响应 QMP、渲染画面 | `system/` + `monitor/` + `ui/` |

这三者跑在**同一个宿主机进程**里，通过 BQL（Big QEMU Lock）协调。

---

## 三、总览：从 main() 到 Guest 第一条指令

```
main()                                      // system/main.c
 │
 └─ qemu_init(argc, argv)                  // system/vl.c — 初始化一切
      │
      ├─ 1. 解析命令行（-M spike、-kernel、-m 1G...）
      │
      ├─ 2. 创建 Machine
      │     object_new("spike")             // 从全局 TypeInfo 哈希表查
      │       ├─ instance_init()             // 创建子对象（CPU、SoC）
      │       └─ realize()
      │           └─ machine->init()        // = spike_board_init()
      │               ├─ 创建 HART（CPU 核）
      │               ├─ 创建 CLINT（定时器 + 软中断）
      │               ├─ memory_region_add_subregion(mem, 0x80000000, ram)
      │               ├─ memory_region_init_rom(mask_rom, ...)
      │               ├─ riscv_load_firmware()   // 加载 OpenSBI 到 DRAM
      │               ├─ create_fdt()             // 生成设备树
      │               └─ riscv_setup_rom_reset_vec() // 写跳转指令到 ROM
      │
      ├─ 3. 初始化加速器（TCG/KVM）
      │     accel_init_interfaces()
      │
      ├─ 4. 加载 Guest 内核/固件
      │     └─ riscv_load_kernel()
      │
      ├─ 5. 创建 monitor/chardev/network/block 后端
      │
      └─ 6. CPU 复位（PC → resetvec，通常是 0x1000）
            └─ cpu_reset()
```

```
main()                                      // system/main.c
 │
 └─ qemu_main_loop()                       // 事件循环（永不退出）
      │
      while (!shutdown_requested) {
      │
      ├─ 等所有 VCPU 线程把时间片用完（或遇到 IO/停机）
      │
      ├─ 处理定时器事件（Timer）
      │
      ├─ 处理 IO 事件（chardev 输入、网络包、块设备完成）
      │
      └─ 再次启动 VCPU 线程
      }
```

**关键认知：QEMU 的事件循环 ≠ VCPU 执行。VCPU 在独立线程中跑，每次执行一小段时间片后回到事件循环。**

---

## 四、三大核心子系统

### 4.1 QOM — C 对象系统

```
程序启动（main 之前）
  type_init(xxx) → __attribute__((constructor))
  → 所有 TypeInfo 注册到全局哈希表

运行时
  object_new("spike")
    → 查哈希表 → 分配 instance_size → 递归调 instance_init
    → qdev_realize() → DeviceClass->realize()
```

| 关键文件 | 内容 |
|---------|------|
| `include/qom/object.h` | Object/TypeInfo/Class 定义 + API |
| `qom/object.c` | type_register、object_new、属性系统 |
| `hw/core/qdev.c` | DeviceState（设备的基类） |

### 4.2 MemoryRegion + AddressSpace — 地址空间

```
Guest 物理地址空间（system_memory）
  │
  ├── [0x00001000-0x00001FFF] mask_rom  (ROM)
  │     └── riscv_setup_rom_reset_vec 写入的跳转指令
  │
  ├── [0x10013000-0x10013FFF] UART0    (IO → sifive_uart_read/write)
  │     └── CPU 写这个范围 → 调用回调 → 往 serial chardev 输出字符
  │
  ├── [0x02000000-0x0200FFFF] CLINT    (IO → riscv_aclint_*)
  │
  └── [0x80000000-0xFFFFFFFF] DRAM     (RAM → mmap 出的宿主机内存)
```

每当添加/删除 MemoryRegion，QEMU 重建 **FlatView**（地址空间的扁平线性数组），供 TCG/KVM 快速查找。

| 关键文件 | 内容 |
|---------|------|
| `include/exec/memory.h` | MemoryRegion/MemoryRegionOps/AddressSpace API |
| `system/memory.c` | MemoryRegion 树管理、FlatView 构建 |
| `system/physmem.c` | 物理内存分配、MMIO dispatch |
| `accel/tcg/cputlb.c` | TLB（SoftMMU）查找 + MMIO 路径 |
| `accel/tcg/translate-all.c` | TCG 翻译初始化 |

### 4.3 SysBus — SoC 总线抽象

```
SoC init 函数做的三件事（以 sifive_e 为例）:

1. sysbus_create_var("sifive_uart", base_addr, irq)
     = object_new + sysbus_realize + sysbus_mmio_map + sysbus_connect_irq

2. sysbus_mmio_map(dev, 0, base_addr)
     = memory_region_add_subregion(system_memory, base_addr, dev->mmio[0].memory)

3. sysbus_connect_irq(dev, 0, plic_irq)
     = 设备.irq[0] → PLIC.gpio_in[n]
```

| 关键文件 | 内容 |
|---------|------|
| `include/hw/sysbus.h` | SysBusDevice 结构定义 |
| `hw/core/sysbus.c` | sysbus_mmio_map、sysbus_connect_irq 实现 |

---

## 五、Guest 指令执行的完整路径（TCG 路径）

这是 QEMU 最核心的流程：

```
                    ┌─────────────────────────────┐
                    │       Guest RISC-V ELF      │
                    │      (加载到 DRAM 中)        │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │   TCG 翻译阶段 (translate.c) │
                    │                              │
                    │   Guest 指令:                 │
                    │     sw a0, 0(t0)             │
                    │        ↓                      │
                    │   TCG IR (中间表示):           │
                    │     tcg_gen_qemu_st_i32()     │
                    │        ↓                      │
                    │   Host 代码 (x86 JIT 生成):    │
                    │     mov [rbx+rax], ecx        │
                    │     ; 或调用 helper 函数       │
                    └──────────┬──────────────────┘
                               │
                    ┌──────────▼──────────────────┐
                    │   TCG 执行阶段 (cpu-exec.c)  │
                    │                              │
                    │   执行 JIT 生成的 Host 代码:   │
                    │                              │
                    │   ┌─ RAM 访问:               │
                    │   │  SoftMMU(TLB) hit →      │
                    │   │  直接读写宿主机 mmap 内存  │
                    │   │  (极快，~几条 Host 指令)   │
                    │   │                          │
                    │   └─ MMIO 访问 (外设寄存器):   │
                    │      SoftMMU(TLB) miss →     │
                    │      → 查 FlatView           │
                    │      → 找到 IO MemoryRegion  │
                    │      → mr->ops->write()      │
                    │      → sifive_uart_write()   │
                    │      → chardev 输出字符       │
                    └──────────────────────────────┘
```

**RAM 和 MMIO 的分岔点**：在 SoftMMU TLB 查找时决定。TLB 条目记录了这段地址是 RAM（直接指针）还是 IO（MemoryRegion + 回调）。

| 关键文件 | 内容 |
|---------|------|
| `target/riscv/translate.c` | RISC-V 指令 → TCG IR |
| `tcg/` | TCG IR → Host 指令（JIT 编译器） |
| `accel/tcg/cpu-exec.c` | VCPU 执行循环 |
| `accel/tcg/cputlb.c` | SoftMMU TLB + MMIO dispatch |
| `system/physmem.c` | memory_region_dispatch_read/write |

---

## 六、从 main() 到 spike_board_init() 的调用链

```
system/main.c:69        main()
system/vl.c:2854        qemu_init()
                          ├─ qemu_create_machine()
                          │   └─ object_new("spike")
                          │       ├─ object_initialize() → instance_init 链
                          │       └─ qdev_realize()
                          │           └─ DeviceClass->realize
                          │               = machine_class_init 里设置的:
                          │                 mc->init = spike_board_init
                          │
system/vl.c              └─ qemu_apply_machine_options()
hw/riscv/spike.c:194     spike_board_init()
                          ├─ riscv_aclint_swi_create()
                          ├─ riscv_aclint_mtimer_create()
                          ├─ memory_region_add_subregion() ×N
                          ├─ riscv_load_firmware()
                          ├─ create_fdt()
                          ├─ riscv_load_kernel()
                          └─ riscv_setup_rom_reset_vec()
system/vl.c              └─ accel_setup_post()
                            └─ CPU 复位，PC → resetvec
```

---

## 七、关键数据结构关系图

```
MachineState                            ← "spike" 机器
  ├── CPUState[] × N                    ← 每个 HART 一个
  │     ├── CPUArchState (env)          ← RISC-V 寄存器、CSR、MMU 状态
  │     └── TCGContext                  ← 翻译缓存、JIT 状态
  │
  ├── DeviceState[]                     ← 所有外设
  │     ├── SysBusDevice               ← SoC 外设（UART/CLINT/PLIC...）
  │     │     ├── MemoryRegion mmio[]  ← 寄存器空间
  │     │     └── qemu_irq irq[]       ← 中断输出引脚
  │     └── I2CSlave                   ← I2C 从设备（TMP105 等）
  │
  ├── MemoryRegion "system_memory"      ← 全局物理地址空间
  │     └── subregions[]               ← 所有 RAM/ROM/IO 的 MR
  │                                      → 展开为 FlatView 给 TCG/KVM 用
  │
  └── FlatView                          ← 地址空间扁平化视图（缓存）
        └── FlatRange[]                 ← 按地址排序的线性数组
```

---

## 八、源码阅读路线建议（框架视角）

有了这个框架后，按以下顺序读源码：

```
1. system/main.c         (~100行)  入口 + 事件循环启动
2. system/vl.c           (看 qemu_init 的结构，不深读) 
3. hw/riscv/spike.c      (270行)   一个完整 Machine 怎么构造的
4. hw/riscv/sifive_e.c   (310行)   SoC + Machine 分离模式
5. target/riscv/cpu.c               CPU 对象怎么定义的
6. accel/tcg/cpu-exec.c             VCPU 执行循环
7. system/physmem.c                 MMIO dispatch 怎么找到设备的
8. hw/char/sifive_uart.c            一个具体外设怎么写
```

每一步对照这个框架的对应部分，读完一个再读下一个。不要把 vl.c 从头读到尾（2800+ 行）。

---

## 核心总结

```
QEMU = 对象树（QOM）+ 地址空间树（MemoryRegion）+ 事件循环

初始化：
  main → qemu_init → object_new(Machine) → machine.init()
    → 创建 CPU、RAM、外设 → 把 MR 插入 system_memory
    → CPU 复位 → PC = resetvec

运行时：
  事件循环 ↔ VCPU 线程（TCG 翻译+执行）
    Guest 访存 → SoftMMU(TLB) → RAM(mmap直接读写) 或 IO(回调函数)

建模你的芯片 = 
  把硬件手册的 memory map 翻译成
    sysbus_create(mr, base, irq)
    memory_region_add_subregion()
    这几种调用的排列组合
```
