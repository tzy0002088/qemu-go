---
title: "QEMU 深度剖析 — 从类型注册到指令分发"
date: 2026-07-02
tags: [qemu, deep-dive, qom, memoryregion, sysbus, internals]
status: notes
---

## 目标

理解 QEMU 底层机制到能够独立排查问题的程度：
- 类型注册到底做了什么？
- `sysbus_realize` 为什么调用之前对象不能用？
- Guest 的一条 `store` 指令，怎么最终调用到设备 `write` 回调？
- 出问题时从哪入手排查？

---

## 第一部分：QOM — QEMU 的 C 对象系统

QEMU 用纯 C 实现了一套对象系统。核心是三个概念：**TypeInfo**（类型的描述）、**ObjectClass**（属于类型，全局唯一）、**Object**（属于实例，每个设备一个）。

### 1.1 TypeInfo — "类型注册表"

```c
// 来自 include/qom/object.h:435
struct TypeInfo {
    const char *name;             // 类型名，如 "sys-bus-device"
    const char *parent;           // 父类型名，如 "device"
    size_t instance_size;         // 实例 struct 的大小 (sizeof 你的 State)
    size_t instance_align;        // 内存对齐
    void (*instance_init)(Object *obj);       // 构造第一步
    void (*instance_post_init)(Object *obj);  // 构造最后一步（罕见）
    void (*instance_finalize)(Object *obj);   // 析构
    bool abstract;                // true = 不能直接创建实例（类似 C++ 纯虚类）
    size_t class_size;            // Class struct 的大小
    void (*class_init)(ObjectClass *klass, const void *data);  // 类初始化
    InterfaceInfo *interfaces;    // 实现的接口
};
```

**type_init 宏展开**：

你用 `type_init(my_register)` 注册时，宏展开为：

```c
static void __attribute__((constructor)) my_register(void);
//                              ^^^^^^^^^^^^^^^^
// GCC 扩展：main() 之前自动调用这个函数
```

所以 **所有类型在 main() 之前就已经注册到全局类型表里了**。`spike` 这个 machine type、`tmp105` 这个 sensor type、`sys-bus-device` 这个 bus type——都是程序启动时就注册好的，和 main 无关。

### 1.2 Object — "每个实例"

```c
// include/qom/object.h:154
struct Object {
    ObjectClass *class;    // 指向类型的 Class（多态的根基）
    ObjectFree *free;      // 释放函数
    GHashTable *properties; // 属性散列表
    uint32_t ref;          // 引用计数
    Object *parent;        // 在对象树中的父节点
};
```

**关键设计**：`Object` 必须是任何实例 struct 的第一个字段。C 保证 struct 的第一个字段在偏移 0 处，所以：

```c
struct MyDeviceState {
    SysBusDevice parent;   // → DeviceState → Object (全部在偏移 0)
    // ... 自有字段
};
MyDeviceState *s = ...;
Object *obj = (Object *)s;   // 无条件安全转换，无开销
```

### 1.3 两阶段构造：init vs realize

这是最容易困惑的地方。QEMU 把构造分成两个阶段：

```
Phase 1: init (instance_init)
  - 创建子对象 (object_initialize_child)
  - 设置默认属性值
  - 不能分配宿主机资源（内存、线程、fd）
  - 不能失败
  
Phase 2: realize (DeviceClass->realize)  
  - 分配宿主机资源
  - 映射 MMIO 到地址空间
  - 连接 IRQ 线
  - 可以失败（返回 Error）
```

**为什么这样设计？** 这来自 QEMU 的设备内省（introspection）需求。QEMU 允许你在命令行指定 `-device` 之前先查询设备有什么属性，这个查询会创建临时对象、读取属性、然后立刻销毁。如果 init 阶段就分配了资源，每次查询都会泄漏资源。

**DeviceState 源码直接说明**（`include/hw/qdev-core.h:13`）：
> Devices are constructed in two stages:
> 1) object instantiation via object_initialize()
> 2) device realization via the #DeviceState.realized property
> The former may not fail, and the latter may return error information.

**实际代码对比**（以 Aspeed AST2600 为例）：

```c
// init: 只创建对象、什么都不分配
static void aspeed_soc_ast2600_init(Object *obj) {
    object_initialize_child(obj, "scu", &s->scu, "aspeed.scu-ast2600-a3");
    object_initialize_child(obj, "i2c", &s->i2c, "aspeed.i2c-ast2600-a3");
    // ... 纯对象树的构建，全部不可失败
}

// realize: 这才真正干活
static void aspeed_soc_ast2600_realize(DeviceState *dev, Error **errp) {
    sysbus_realize(SYS_BUS_DEVICE(&s->scu), errp);  // 可以失败
    sysbus_mmio_map(SYS_BUS_DEVICE(&s->scu), 0, addr);
    sysbus_connect_irq(SYS_BUS_DEVICE(&s->rtc), 0, irq);
    // ... 真正分配资源
}
```

### 1.4 从 TypeInfo 到实际对象：全景路径

```
程序启动（main 之前）
  │
  ├── type_init(xxx) → __attribute__((constructor)) 注册 TypeInfo 到全局哈希表
  │
  └── main()
        │
        └── qemu_init()
              │
              └── select_machine()
                    │
                    └── object_new("spike")   // 从全局表查 TypeInfo，分配内存
                          │
                          ├── 1. 分配 sizeof(SpikeState) 的内存
                          ├── 2. 递归调用各层 instance_init:
                          │      object_init → DeviceClass->instance_init → ...
                          │      → spike_machine_instance_init() (空)
                          │      → 所有 object_initialize_child() 在这阶段执行
                          │
                          └── 3. qdev_realize()
                                │
                                └── DeviceClass->realize() 链:
                                      MachineClass->init = spike_board_init()
                                        │
                                        ├── object_initialize_child("soc0", ...)
                                        ├── sysbus_realize(soc)  // SoC 的 realize
                                        │     └── sifive_e_soc_realize()
                                        │           创建 UART、PLIC、CLINT...
                                        └── memory_region_add_subregion(...)
```

---

## 第二部分：MemoryRegion — QEMU 的心脏

### 2.1 数据结构

```c
// include/system/memory.h:823
struct MemoryRegion {
    Object parent_obj;                         // QOM 基类

    /* 分类标志 */
    bool ram;           // true = RAM/ROM 类，直接读写宿主机内存
    bool rom_device;    // true = ROM 设备（可读如 RAM，写触发回调）
    bool readonly;
    bool is_iommu;      // true = IOMMU 区域（地址翻译）
    bool subpage;        // 内部使用：小于一页的 MMIO 分片

    /* 如果是 RAM/ROM 类型 */
    RAMBlock *ram_block;   // 指向实际的宿主机内存块
    Object *owner;         // 谁创建了这个 MR（用于调试）

    /* 如果是 IO(MMIO) 类型 */
    const MemoryRegionOps *ops;   // 读写回调函数表
    void *opaque;                 // 传给回调的指针（通常指向设备 State）

    /* 地址空间组织 */
    MemoryRegion *container;      // 我在哪个 container 里？
    MemoryRegion *alias;          // 我是谁的别名？
    hwaddr alias_offset;          // 别名偏移
    Int128 size;                  // 我有多大
    hwaddr addr;                  // 我在 container 中的偏移
    int32_t priority;             // 重叠区域时的优先级
    bool terminates;              // true = 搜索终止于此（不再往下找子区域）

    /* 子区域链表 */
    QTAILQ_HEAD(, MemoryRegion) subregions;  // 子区域链表头
    QTAILQ_ENTRY(MemoryRegion) subregions_link; // 链表节点

    const char *name;             // 调试用名字
    DeviceState *dev;             // 所属设备（re-entrancy guard 用）
};
```

### 2.2 三种 MR 类型及其内部机制

#### RAM 类型：`memory_region_init_ram()`

```c
// spike.c:262
memory_region_add_subregion(system_memory, 0x80000000, machine->ram);
//                                                      ^^^^^^^^^^^^
// machine->ram 是在 machine 初始化时创建的 RAM MemoryRegion
// 内部：分配了 sizeof(guest_ram) 的宿主机虚拟内存
```

**内部机制**：
- 创建时调用 `qemu_ram_alloc()` → `mmap(NULL, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0)`
- Guest 访问 `0x80000000` → 地址翻译 → 直接读写这块 mmap 内存
- 速度极快（无回调函数调用）

#### ROM 类型：`memory_region_init_rom()`

```c
// spike.c:266
memory_region_init_rom(mask_rom, NULL, "riscv.spike.mrom", 0xf000, &error_fatal);
memory_region_add_subregion(system_memory, 0x1000, mask_rom);
```

**内部机制**：和 RAM 类似，但写操作被忽略（readonly 标志）。初始化后可以往里面写（通过 `rom_add_blob_fixed_as`），写完后设置为只读。

#### IO(MMIO) 类型：`memory_region_init_io()`

```c
// 外设的寄存器空间
memory_region_init_io(&s->iomem, obj, &my_device_ops, s, "my-device", 0x1000);
//                                    ^^^^^^^^^^^^^^  ^
//                                    read/write回调   opaque → 传给回调
```

**内部机制**：
- **不分配**宿主机内存
- Guest 访问这段地址 → 地址翻译到达这个 MR → 调用 `ops->read()` 或 `ops->write()`
- 这是外设模拟的基础

### 2.3 从 Guest 指令到设备回调：完整的 dispatch 路径

这是理解 QEMU 最核心的问题。以 RISC-V guest 执行 `sw a0, 0(t0)` (store word) 为例，假设 `t0 = 0x10013000`（UART 的 MMIO 地址）：

```
1. TCG 翻译阶段（translate.c）
   ┌──────────────────────────────────────────┐
   │ sw a0, 0(t0)                             │
   │   ↓ translate.c 解码                     │
   │ 生成 TCG ops:                            │
   │   tcg_gen_qemu_st_i32(data, addr, ...)   │
   │   ↓                                      │
   │ 最终生成 host x86 代码（JIT 编译）:        │
   │   helper_le_stl_mmu(env, addr, data, ...) │
   └──────────────────────────────────────────┘

2. SoftMMU 查找（accel/tcg/cputlb.c）
   ┌──────────────────────────────────────────┐
   │ helper_le_stl_mmu()                       │
   │   ↓                                      │
   │ 查 TLB: addr=0x10013000                  │
   │   ├── TLB hit → 直接拿到 host 内存指针，写入（RAM 的情况）
   │   └── TLB miss →                        │
   │       ↓                                  │
   │   io_prepare() → address_space_translate()│
   │       ↓                                  │
   │   FlatView 查找（二分搜索）:               │
   │     0x00001000 - 0x00001FFF → mask_rom    │
   │     0x10000000 - 0x10007FFF → AON         │
   │     0x10013000 - 0x10013FFF → uart0 ←命中  │
   └──────────────────────────────────────────┘

3. MemoryRegion dispatch（softmmu/physmem.c → system/memory.c）
   ┌──────────────────────────────────────────┐
   │ memory_region_dispatch_write(mr, 0x00, data, 4) │
   │   ↓                                      │
   │ 找到的 mr 是 IO 类型，有 ops              │
   │   ↓                                      │
   │ mr->ops->write(mr->opaque, 0x00, data, 4) │
   │   = sifive_uart_write(s, 0x00, 'A', 4)   │
   └──────────────────────────────────────────┘

4. 设备回调（hw/char/sifive_uart.c）
   ┌──────────────────────────────────────────┐
   │ sifive_uart_write(s, addr, value, size)    │
   │   switch (addr) {                        │
   │   case 0x00: /* TXDATA */                │
   │       chardev_putchar(s->chr, value);    │
   │       break;                             │
   │   }                                      │
   └──────────────────────────────────────────┘
```

**为什么 TLB 查表是高效的**：
- 第一次访问一个 MMIO 地址 → TLB miss → 查 FlatView → 填充 TLB
- 后续访问同一地址 → TLB hit → 直接跳转到回调（~几十纳秒）
- FlatView 是地址空间的**扁平化视图**，去掉了 container 的层次，方便二分搜索

### 2.4 FlatView — 地址空间的"展开版"

`system_memory` 是一棵 MemoryRegion 树（subregions 嵌套 subregions）。但每次 CPU 访存都遍历这棵树太慢了。

**FlatView** 把树展开成线性数组，按地址排序：

```
树状（逻辑）:
system_memory
├── [0x00001000-0x00001FFF] mask_rom
├── [0x10000000-0x10007FFF] AON
│   └── [0x10000000-0x100000FF] AON_WDT
│       ...嵌套在 AON 里
├── [0x10012000-0x10012FFF] GPIO
├── [0x10013000-0x10013FFF] UART0
└── [0x80000000-0xFFFFFFFF] DRAM

展开后（FlatView）:
[0x00001000-0x00001FFF] → mask_rom (ROM)
[0x10000000-0x100000FF] → AON_WDT (IO)    ← 子区域优先（priority 机制）
[0x10000100-0x10007FFF] → AON (IO)
[0x10012000-0x10012FFF] → GPIO (IO)
[0x10013000-0x10013FFF] → UART0 (IO)
[0x80000000-0xFFFFFFFF] → DRAM (RAM)
```

每次添加/删除 MemoryRegion → FlatView 重建 → 重新排序 → 通知 MemoryListener（KVM 等加速器通过这个机制更新页表）。

---

## 第三部分：SysBus — MMIO 映射和 IRQ 连线

### 3.1 SysBusDevice 的数据结构

```c
// include/hw/sysbus.h:56
struct SysBusDevice {
    DeviceState parent_obj;     // 继承 DeviceState

    int num_mmio;               // 注册了几个 MMIO 区域
    struct {
        hwaddr addr;            // 映射到哪个基地址
        MemoryRegion *memory;   // 指向设备的 IO MemoryRegion
    } mmio[QDEV_MAX_MMIO];     // 最多 32 个 MMIO 区域

    int num_pio;                // x86 专有：端口 IO
    uint32_t pio[QDEV_MAX_PIO];
};
```

### 3.2 "MMIO 映射"到底做了什么

```c
sysbus_realize(SYS_BUS_DEVICE(&s->uart0), errp);   // 1. realize 设备
sysbus_mmio_map(SYS_BUS_DEVICE(&s->uart0), 0, 0x10013000);  // 2. 映射
```

`sysbus_mmio_map` 的实现（`hw/core/sysbus.c`）本质就两件事：

```c
void sysbus_mmio_map(SysBusDevice *dev, int n, hwaddr addr) {
    // 1. 记住地址（供后续查询）
    dev->mmio[n].addr = addr;
    // 2. 把 MR 插入 system_memory 的 FlatView
    memory_region_add_subregion(get_system_memory(), addr, dev->mmio[n].memory);
    //                           ^^^^^^^^^^^^^^^^^^^^  ^^^^  ^^^^^^^^^^^^^^^^^
    //                           全局地址空间           基地址  设备的寄存器 MR
}
```

**MMIO 映射 = 把设备的 MemoryRegion 插入全局地址空间的指定位置。**没有更复杂的东西。

### 3.3 "IRQ 连线"到底做了什么

```c
sysbus_connect_irq(SYS_BUS_DEVICE(&s->uart0), 0, irq);
//                                                ^^^
//                                    这个 irq 是 PLIC 的某个 gpio_in 引脚
```

`qemu_irq` 本质是一个函数指针 + opaque：

```c
// include/hw/irq.h
typedef struct IRQState *qemu_irq;

struct IRQState {
    qemu_irq_handler handler;  // 函数指针: 触发中断时调用
    void *opaque;              // 传给 handler
    int n;                     // 这个信号在父对象的第几个引脚
};
```

所以 `sysbus_connect_irq(dev, 0, plic_irq3)` 做的：把设备的第 0 个中断输出连到 PLIC 的第 3 个中断输入。当设备调用 `qemu_set_irq(s->irq, 1)` 时：

```c
qemu_set_irq(s->irq, 1)
  → irq->handler(irq->opaque, irq->n, 1)
    → plic_irq_handler(plic, 3, 1)   // PLIC 收到中断信号
      → PLIC 内部逻辑 → 拉高 CPU 的外部中断线
```

---

## 第四部分：spike.c 逐行解剖（board_init 关键路径）

```c
static void spike_board_init(MachineState *machine)        // 行 194
{
    // MemMapEntry = { hwaddr base; hwaddr size; }
    // 来自 spike_memmap[]（行 44-49）:
    //   MROM:  0x00001000, 0xf000     ← Mask ROM（16KB）
    //   HTIF:  0x01000000, 0x1000     ← Host-Target Interface
    //   CLINT: 0x02000000, 0x10000    ← 定时器 + 软中断
    //   DRAM:  0x80000000, size=0     ← 实际大小由 -m 指定
    const MemMapEntry *memmap = spike_memmap;

    // 从 machine 子类 cast 回 SpikeState
    // SPIKE_MACHINE(machine) = OBJECT_CHECK(SpikeState, machine, "spike")
    SpikeState *s = SPIKE_MACHINE(machine);

    // get_system_memory() 返回全局地址空间（"system" MemoryRegion）
    // 所有物理地址最终都 resolve 到这个 MR 的子区域上
    MemoryRegion *system_memory = get_system_memory();

    // g_new() = glib 的 malloc + 清零
    // 这里不用 object_new()，因为 MemoryRegion 不是 Device（轻量级对象）
    MemoryRegion *mask_rom = g_new(MemoryRegion, 1);

    // --- 第一步：初始化 HART（RISC-V CPU 核） ---
    // 用 RISCV_HART_ARRAY 封装（支持多核多 socket）
    for (i = 0; i < riscv_socket_count(machine); i++) {
        // 创建 soc0, soc1... 对象（TYPE_RISCV_HART_ARRAY）
        object_initialize_child(OBJECT(machine), soc_name, &s->soc[i],
                                TYPE_RISCV_HART_ARRAY);
        // 配置属性
        object_property_set_str(OBJECT(&s->soc[i]), "cpu-type",
                                machine->cpu_type, &error_abort);
        object_property_set_int(OBJECT(&s->soc[i]), "hartid-base", base_hartid, ...);
        object_property_set_int(OBJECT(&s->soc[i]), "num-harts", hart_count, ...);
        // realize → CPU 对象真正创建、TCG 初始化
        sysbus_realize(SYS_BUS_DEVICE(&s->soc[i]), &error_fatal);

        // 创建 CLINT（核内定时器 + 核间软件中断）
        riscv_aclint_swi_create(...);    // 软件中断
        riscv_aclint_mtimer_create(...); // 机器定时器
    }

    // --- 第二步：映射 DRAM ---
    // machine->ram 是 MachineState 自动创建的 RAM MemoryRegion
    // 内部已经是 mmap 出来的宿主机内存
    memory_region_add_subregion(system_memory, 0x80000000, machine->ram);

    // --- 第三步：创建 Mask ROM ---
    // mask_rom 是一块 ROM 类型的 MemoryRegion
    memory_region_init_rom(mask_rom, NULL, "riscv.spike.mrom", 0xf000, ...);
    memory_region_add_subregion(system_memory, 0x1000, mask_rom);

    // --- 第四步：加载固件 ---
    // riscv_find_firmware() 查找固件文件（OpenSBI）
    firmware_name = riscv_find_firmware(machine->firmware, ...);
    riscv_load_firmware(firmware_name, &firmware_load_addr, ...);
    // firmware_load_addr 通常是 DRAM 起始地址
    // 固件 ELF 被加载到 DRAM 中

    // --- 第五步：创建设备树 ---
    create_fdt(s, memmap, ...);  // 生成 FDT blob，Guest 内核读取它来发现硬件

    // --- 第六步：加载内核 ---
    riscv_load_kernel(machine, ...);
    // 如果指定了 -kernel，把内核 ELF 加载到 DRAM 中

    // --- 第七步：设置复位向量 ---
    riscv_setup_rom_reset_vec(machine, &s->soc[0], firmware_load_addr,
                              0x1000,     // Mask ROM 基地址
                              0xf000,     // Mask ROM 大小
                              kernel_entry, fdt_load_addr);
    // 实际做的事：写几条跳转指令到 mask_rom 里
    // CPU 上电后 PC=0x1000 → 执行 mask_rom 中的跳转 → 跳到固件入口

    // --- 第八步：HTIF 控制台 ---
    htif_mm_init(system_memory, serial_hd(0), 0x01000000, ...);
    // HTIF = Host-Target Interface, 一个极简单的 IO 通道
    // 在 0x01000000 处创建一个 IO MemoryRegion
}
```

---

## 第五部分：排查问题的调试入口

### 5.1 查看类型注册情况

```c
// 在 QEMU monitor 中:
info qom-tree          // 列出所有对象树（可能非常多）
qom-list /machine      // 列出 machine 的属性
qom-get /machine type  // 获取 machine 类型名
```

### 5.2 查看地址空间布局

```c
// QEMU monitor:
info mtree             // 打印 FlatView → 看到所有 MemoryRegion 映射
// 输出示例:
//  0000000000001000-000000000000ffff (prio 0, rom): riscv.spike.mrom
//  0000000001000000-0000000001000fff (prio 0, i/o): riscv.htif
//  0000000002000000-000000000200ffff (prio 0, i/o): riscv.aclint.swi
//  0000000080000000-0000000087ffffff (prio 0, ram): riscv.spike.ram
```

### 5.3 追踪 MMIO 访问

```bash
# 编译 QEMU 时开启 trace:
./configure --enable-trace-backends=log
make

# 运行时开启 MMIO 访问追踪:
qemu-system-riscv64 -M spike \
  -trace memory_region_ops\* \
  -trace memory_region_subpage\* \
  ...

# 输出:
# memory_region_ops_write: riscv.spike.mrom offset 0x1004 size 4 value 0x...
```

### 5.4 常见错误和排查

| 症状 | 可能原因 | 排查手段 |
|------|---------|---------|
| Guest 启动后立即崩溃 | 复位向量加载到错误地址 | 检查 `riscv_setup_rom_reset_vec` 的参数；`info mtree` 确认 ROM 地址 |
| 串口无输出 | UART 未创建或 Chardev 未连接 | `info qom-tree` 找到 UART；检查 `serial_hd(0)` 是否有前端 |
| 中断不触发 | IRQ 未连接或 IRQ 号写错 | 检查 `sysbus_connect_irq` 调用；在设备 trace 中确认 `qemu_set_irq` 是否被调用 |
| 设备寄存器读写无反应 | 地址映射错误 | `info mtree` 确认设备 MMIO 区域是否在正确地址上 |
| Guest 访问未实现区域 | 地址空间有空隙 | `info mtree` 找空隙；加 `create_unimplemented_device()` 优雅降级 |

---

## 核心总结

```
QOM:
  TypeInfo → type_init → 全局哈希表 → object_new(name) → init → realize
  一个 type 只有一个 Class，但可以有无数个 Object
  init 不可失败（no resource），realize 可失败（allocates resources）

MemoryRegion:
  RAM  → 直接读写宿主机 mmap 内存
  ROM  → 同上但只读
  IO   → 触发 read/write 回调
  Alias → 指向另一个 MR（窗口机制）
  Container → 纯容器（只用来组织结构）

Guest 访存路径:
  TCG JIT → SoftMMU(TLB) → FlatView(二分搜索) → mr->ops->read/write()

SysBus:
  sysbus_mmio_map   = memory_region_add_subregion(system_memory, addr, mr)
  sysbus_connect_irq = 把设备的 qemu_irq 引脚接到中断控制器的 gpio_in

SPIKE:
  地址空间 = memmap[] + memory_region_add_subregion
  CPU    = TYPE_RISCV_HART_ARRAY
  CLINT  = riscv_aclint_swi_create + riscv_aclint_mtimer_create
  ROM    = memory_region_init_rom + riscv_setup_rom_reset_vec
  DRAM   = machine->ram
```
