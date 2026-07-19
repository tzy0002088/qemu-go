---
title: "内核页表初始化 (kvmmake)"
date: 2026-07-19
tags: [xv6, riscv, virtual-memory, page-table, kernel]
status: notes
---

## 背景

阅读 `kernel/vm.c` 的 `kvmmake()` 时产生困惑：为什么要专门给内核分配一个页表？内核不就是跑在物理地址上吗？

## 提问

1. `kvmmake()` 里的 `kalloc()` 分配的是什么？为什么要申请一个 pagetable？
2. 内核页表是给 MMU 用的吗？
3. 为什么内核使用直接映射（VA == PA），唯独 trampoline 例外？
4. 每个映射分别是干什么的？

## 理解

### 内核为什么需要页表

RISC-V CPU 一旦开启分页（`satp` 寄存器非零），**所有内存访问**都要经过 MMU 翻译——内核代码也不例外。所以内核必须为自己建立一张页表，这张页表就是 `kernel_pagetable`（`vm.c:14`）。

`kvminit()` 在启动早期调用 `kvmmake()` 创建这张表，然后 `kvminithart()` 通过 `w_satp(MAKE_SATP(kernel_pagetable))` 把它写入 `satp` 寄存器，MMU 就开始工作了。

### kalloc() 分配的是什么

```c
kpgtbl = (pagetable_t)kalloc();    // 分配一页物理内存（4096 字节）
memset(kpgtbl, 0, PGSIZE);         // 清零 → 空的根页表（Level-2）
```

`kalloc()` 分配的是**根页表**（RISC-V Sv39 的 Level-2 页表）。一页正好 512 个 8 字节 PTE，共 4096 字节。这只是"根"——Level-1 和 Level-0 的页表页由后续 `mappages()` → `walk()` 按需动态分配。

### 内核页表的映射内容

`kvmmake()` 通过 `kvmmap()` 建立以下映射：

| kvmmap 调用 | 虚拟地址 | 物理地址 | 大小 | 权限 | 说明 |
|---|---|---|---|---|---|
| `UART0` | `0x10000000` | `0x10000000` | 4KB | R\|W | UART 寄存器（MMIO） |
| `VIRTIO0` | `0x10001000` | `0x10001000` | 4KB | R\|W | virtio 磁盘 MMIO |
| `PLIC` | `0x0C000000` | `0x0C000000` | 64MB | R\|W | 平台级中断控制器 |
| `KERNBASE..etext` | `0x80000000` | `0x80000000` | 到 etext | R\|X | 内核代码段（只读可执行） |
| `etext..PHYSTOP` | etext | etext | 到 PHYSTOP | R\|W | 内核数据 + 可用 RAM |
| `TRAMPOLINE` | `MAXVA-PGSIZE` | `(uint64)trampoline` | 4KB | R\|X | trampoline 页（**非恒等映射**） |
| 内核栈 (per proc) | `KSTACK(p)` | kalloc 分配 | 4KB | R\|W | 每个进程的内核栈 |

### 直接映射（VA == PA）的设计意图

外设寄存器（UART、VIRTIO、PLIC）和物理内存（KERNBASE..PHYSTOP）都使用直接映射。这样做的好处：
- 内核可以用物理地址直接操作设备，不需要额外的地址转换逻辑
- 分页开启前后代码无感知——同样的地址，经过页表翻译后落到同一物理地址
- 简化了物理内存分配器（kalloc/free）的实现

### trampoline 为什么是例外

```c
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
//             ^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^
//             极高的 VA    实际物理地址在
//             (接近 MAXVA)  内核 .text 段中
```

trampoline 页被映射到 `MAXVA - PGSIZE`（即 `0x3FFFFFFFF000`，256 GB 虚拟地址空间的最后一页），这是用户态和内核态**共享的虚拟地址**。当 `satp` 从内核页表切换到用户页表（或反过来）时，trampoline 代码的 VA 不变——切换发生在 trampoline 代码内部，因此不会出现"切页表后 PC 指向的地址突然无效"的问题。

### 内核页表 vs 用户页表

| | 内核页表 (`kernel_pagetable`) | 用户进程页表 |
|---|---|---|
| 数量 | 全局唯一，所有 CPU 共享 | 每个进程一个 |
| 创建 | `kvmmake()` @ 启动 | `proc_pagetable()` @ kexec/kfork |
| 映射模式 | 恒等映射（VA=PA）为主 | VA 从 0x0 起 |
| TRAMPOLINE | 有（同一物理页） | 有（同一物理页，同一 VA） |
| TRAPFRAME | 无 | 有 |
| satp.MODE | Sv39 (8) | Sv39 (8) |

## 要点

- `kvmmake()` 中的 `kalloc()` 分配的是 Sv39 **根页表（Level-2）**，不是整个页表树
- 内核页表 = MMU 翻译的"地图"，写入 `satp` 后 MMU 开始工作
- 直接映射（VA=PA）简化内核设计，只有 trampoline 例外
- 中间级页表（Level-1, Level-0）由 `walk()` 按需分配，不在 `kvmmake` 中显式创建

## 参考

- `kernel/vm.c`: kvmmake, kvminit, kvminithart, walk, mappages
- `kernel/memlayout.h`: 物理内存布局常量
- RISC-V privileged spec v20211203, Section 4.4 (Sv39)
