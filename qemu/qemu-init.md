---
title: "QEMU 初始化流程（qemu-init）"
date: 2026-07-08
tags: [qemu, init, startup, main, boot-flow]
status: notes
---

## 背景

来自格维开源社区 QEMU 训练营 2026 的教程内容（`qemu.gevico.online/tutorial/2026/ch1/qemu-init/`），讲解 QEMU 从 `main()` 到进入事件循环的完整初始化过程。

## 整体流程

```
main()                              // system/main.c
  │
  └─ qemu_init(argc, argv)          // system/vl.c — 初始化所有子系统
       │
       ├─ 1. select_machine()       选择 MachineClass（-M spike/pc/virt）
       │
       ├─ 2. cpu_exec_init_all()    CPU 执行环境初始化
       │
       ├─ 3. memory_map_init()      创建 system_memory + system_io 顶级地址空间
       │
       ├─ 4. page_size_init()       页大小初始化
       │
       ├─ 5. configure_accelerator() 配置加速器（TCG/KVM）
       │
       ├─ 6. accel_init_machine()    加速器初始化（如 tcg_init）
       │
       ├─ 7. machine_run_board_init()  ← ★ 板级初始化入口
       │     └─ MachineClass->init(machine)
       │          = spike_board_init()   (RISC-V)
       │          = pc_init1()           (x86)
       │            ├─ 创建 CPU
       │            ├─ 分配 RAM
       │            ├─ 创建总线（PCI/I2C）
       │            └─ 创建所有板载外设
       │
       └─ 8. 设备 realize、loadvm/migration 处理
```

```
main()
  │
  └─ qemu_main_loop()              // 进入事件循环，永不出
       │
       while (!shutdown) {
         ├─ VCPU 线程执行 Guest 指令
         ├─ 处理定时器事件
         └─ 处理 IO 事件（chardev、网络、块设备）
       }
```

## 关键函数

| 函数 | 做了什么 | 对应文件 |
|------|---------|---------|
| `select_machine()` | 根据 `-M` 参数查 MachineClass（每种机器 type_init 注册的） | `system/vl.c` |
| `memory_map_init()` | 创建 `system_memory` 和 `system_io` 两个顶级 MemoryRegion 容器 | `system/memory.c` |
| `configure_accelerator()` | 决定用 TCG 还是 KVM | `system/vl.c` |
| `machine_run_board_init()` | 调用具体 Machine 的 init 函数创建所有设备 | `hw/core/machine.c` |
| `cpu_create()` | object_new + qdev_realize → 创建 VCPU 线程 | `hw/core/cpu-common.c` |

## CPU 创建链（以 RISC-V 为例）

```
cpu_create(TYPE_RISCV_CPU)
  ├─ object_new()          // QOM: 分配 CPUState，调 instance_init
  └─ qdev_realize()        // QOM: 调 realize
       └─ riscv_cpu_realize()
            └─ qemu_init_vcpu()
                 └─ cpus_accel->create_vcpu_thread()
                      └─ TCG: 创建 VCPU 线程 → 进入 cpu_exec() 循环
```

## GDB 断点建议

```bash
gdb --args qemu-system-riscv64 -M spike
(gdb) b select_machine
(gdb) b memory_map_init
(gdb) b machine_run_board_init
(gdb) b spike_board_init
```

## 参考

- QEMU 训练营 2026: https://qemu.gevico.online/tutorial/2026/ch1/qemu-init/
- Martins3 QEMU Init: https://martins3.github.io/qemu/init.html
- QEMU Wiki (Paolo Bonzini): https://wiki.qemu.org/User:Paolo_Bonzini/Machine_init_sequence
