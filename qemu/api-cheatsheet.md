---
title: "芯片建模核心 API 速查"
date: 2026-07-09
tags: [qemu, api, reference, device-emulation, soc, machine]
status: notes
---

## 背景

芯片原厂工程师用 QEMU 建模自己的 SoC + Board，不需要理解 TCG/SoftMMU/FlatView 底层。只需要掌握三类 API：**外设建模**、**SoC 组装**、**Machine 创建**。共 15 个核心 API。

---

## 第一类：外设建模（4 个 API）

每个自研 IP 写一个 `.c` 文件，核心结构：

```c
// 你的设备状态
typedef struct {
    SysBusDevice parent;     // 固定，继承 SysBusDevice
    MemoryRegion iomem;      // 寄存器空间
    qemu_irq irq;            // 中断输出引脚
    uint32_t reg0;           // 你的第一个寄存器
    uint32_t reg1;           // 你的第二个寄存器
} MyIPState;

// CPU 读寄存器时调用
static uint64_t my_ip_read(void *opaque, hwaddr addr, unsigned size) {
    MyIPState *s = opaque;
    switch (addr) {
    case 0x00: return s->reg0;
    case 0x04: return s->reg1;
    default: return 0;
    }
}

// CPU 写寄存器时调用
static void my_ip_write(void *opaque, hwaddr addr, uint64_t val, unsigned size) {
    MyIPState *s = opaque;
    switch (addr) {
    case 0x00: s->reg0 = val; break;
    case 0x04: s->reg1 = val; break;
    }
}

static const MemoryRegionOps my_ip_ops = {
    .read  = my_ip_read,
    .write = my_ip_write,
    .endianness = DEVICE_NATIVE_ENDIAN,
    .valid = { .min_access_size = 4, .max_access_size = 4 },
};
```

### API 1: `memory_region_init_io`

```c
// 在 instance_init 里调用，创建 IO 类型 MemoryRegion
memory_region_init_io(&s->iomem, obj, &my_ip_ops, s, "my-ip", 0x1000);
//                       MR     所属对象 回调函数表 opaque 调试名  大小
```

### API 2: `sysbus_init_mmio`

```c
// 在 instance_init 里调用，把 MR 导出给 SysBus，SoC 后续才能 map
sysbus_init_mmio(SYS_BUS_DEVICE(obj), &s->iomem);
```

### API 3: `sysbus_init_irq`

```c
// 在 instance_init 里调用，声明"我有 1 个中断输出引脚"
sysbus_init_irq(SYS_BUS_DEVICE(obj), &s->irq);
```

### API 4: `qemu_set_irq`

```c
// 在 read/write 回调里调用，触发中断
qemu_set_irq(s->irq, 1);  // 拉高中断
qemu_set_irq(s->irq, 0);  // 拉低中断
```

**外加一个模板**：QOM 注册（每次照抄改名字）：
```c
static void my_ip_class_init(ObjectClass *oc, const void *data) {
    DeviceClass *dc = DEVICE_CLASS(oc);
    dc->realize = my_ip_realize;
}

static const TypeInfo my_ip_info = {
    .name          = TYPE_MY_IP,
    .parent        = TYPE_SYS_BUS_DEVICE,
    .instance_size = sizeof(MyIPState),
    .instance_init = my_ip_init,
    .class_init    = my_ip_class_init,
};
type_init(my_ip_register_types)
```

---

## 第二类：SoC 组装（5 个 API）

在 SoC 的 `realize` 函数里，按硬件手册的 memory map 组装所有 IP。

### API 5: `sysbus_create_simple`

```c
// 一行创建外设 + 映射地址 + 连接 IRQ（最常用）
DeviceState *dev = sysbus_create_simple("my-ip", 0x10000000,
                                        qdev_get_gpio_in(plic, IRQ_N));
//                                      type       基地址      PLIC 的某个中断输入
```

### API 6: `sysbus_mmio_map`

```c
// 手动映射（当设备已创建但还没映射时用）
sysbus_mmio_map(SYS_BUS_DEVICE(dev), 0, 0x10000000);
//                                 ^区域编号   基地址
```

### API 7: `sysbus_connect_irq`

```c
// 手动连中断（当设备已创建但中断还没连时用）
sysbus_connect_irq(SYS_BUS_DEVICE(dev), 0, plic_irq);
//                                    ^输出引脚  ^目的中断线
```

### API 8: `qdev_get_gpio_in`

```c
// 获取中断控制器的某个输入引脚
qemu_irq irq = qdev_get_gpio_in(plic, IRQ_N);
```

### API 9: `create_unimplemented_device`

```c
// 还没写的 IP 先占坑，Guest 访问不崩溃，打印 "unimplemented" 日志
create_unimplemented_device("my-dma", 0x20000000, 0x1000);
```

---

## 第三类：Machine 创建（5 个 API）

在 `machine_init` 函数里，拼板级器件。

### API 10: `memory_region_add_subregion`

```c
// 把 RAM/ROM/外设的 MR 塞进全局地址空间
memory_region_add_subregion(get_system_memory(), 0x80000000, machine->ram);
```

### API 11: `memory_region_init_rom`

```c
// 创建 Boot ROM
MemoryRegion *mask_rom = g_new(MemoryRegion, 1);
memory_region_init_rom(mask_rom, NULL, "board.mrom", 0xf000, &error_fatal);
memory_region_add_subregion(get_system_memory(), 0x1000, mask_rom);
```

### API 12: `riscv_setup_rom_reset_vec`

```c
// 写复位向量：CPU 上电 PC=rom_base，执行 ROM 里小跳板，跳到 start_addr
riscv_setup_rom_reset_vec(machine, &s->soc, start_addr,
                          rom_base, rom_size, kernel_entry, fdt_addr);
```

### API 13: `serial_mm_init`

```c
// 快速创建 16550a UART（一行搞定）
serial_mm_init(get_system_memory(), 0x10000000, 0,
               NULL, 399193, serial_hd(0), DEVICE_LITTLE_ENDIAN);
//    地址空间     基地址  regshift IRQ   波特率    chardev      字节序
```

### API 14: `object_initialize_child` + `sysbus_realize`

```c
// 创建 SoC 对象（CPU）
object_initialize_child(OBJECT(machine), "soc", &s->soc,
                        TYPE_RISCV_HART_ARRAY);
object_property_set_str(OBJECT(&s->soc), "cpu-type", machine->cpu_type, ...);
object_property_set_int(OBJECT(&s->soc), "num-harts", machine->smp.cpus, ...);
sysbus_realize(SYS_BUS_DEVICE(&s->soc), &error_fatal);
```

### API 15: `riscv_find_and_load_firmware` + `riscv_load_kernel` + `riscv_load_fdt`

```c
// 加载固件、内核、设备树（virt_machine_done 里用）
riscv_find_and_load_firmware(machine, &boot_info, "opensbi", &start_addr, NULL);
riscv_load_kernel(machine, &boot_info, kernel_start_addr, true, NULL);
riscv_load_fdt(fdt_addr, machine->fdt);
```

---

## 总结

```
外设建模 (5)            SoC 组装 (5)           Machine (5)
─────────────────      ─────────────────      ─────────────────
memory_region_init_io  sysbus_create_simple   memory_region_add_subregion
sysbus_init_mmio       sysbus_mmio_map        memory_region_init_rom
sysbus_init_irq        sysbus_connect_irq     riscv_setup_rom_reset_vec
qemu_set_irq           qdev_get_gpio_in       serial_mm_init
QOM 注册模板            create_unimplemented   object_initialize_child
                                               + sysbus_realize
                                              riscv_find_and_load_firmware
                                              riscv_load_kernel
```

**典型工作流**：

```
硬件手册 → Memory Map 表 → SoC realize() 里的 sysbus_create_simple() 调用列表
硬件手册 → 寄存器表     → MyIPState struct + read/write 两个函数
硬件手册 → IRQ 路由表   → sysbus_connect_irq() 调用列表
```

下一步实战：给 my_board 加一个自研 GPIO 设备，会用到第一、二类全部 API。
