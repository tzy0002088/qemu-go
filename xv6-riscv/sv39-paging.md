---
title: "Sv39 三级页表深入"
date: 2026-07-19
tags: [xv6, riscv, sv39, mmu, page-table, virtual-address]
status: notes
---

## 背景

xv6-riscv 使用 RISC-V 的 **Sv39** 分页方案。在阅读 `walk()` 函数和 `PX` 宏时，对"三级页表"的具体位布局、各级的命名、以及 xv6 是否使用大页产生了疑问。

## 提问

1. Sv39 用了几级页表？虚拟地址的 bit 布局是怎样的？
2. `PX(2, va)`、`PX(1, va)`、`PX(0, va)` 分别取的是 VA 的哪些位？落到哪一级页表？
3. kernel_pagetable 是 Level-2，为什么编号最大的反而是根？"一级"还是"三级"？
4. 中间级页表（Level-1、Level-0）在哪里分配的？
5. xv6 用大页（2MB megapage / 1GB gigapage）了吗？

## 理解

### Sv39 虚拟地址格式

来自 RISC-V 特权规范 v20211203，Section 4.4，Figure 4.19：

```
 38        30 29        21 20        12 11         0
┌────────────┬────────────┬────────────┬────────────┐
│  VPN[2]    │  VPN[1]    │  VPN[0]    │ page offset│
│   9 bits   │   9 bits   │   9 bits   │  12 bits   │
└────────────┴────────────┴────────────┴────────────┘
     │            │            │            │
     ▼            ▼            ▼            ▼
  索引 L2      索引 L1      索引 L0      页内字节偏移
  根页表       中间页表     叶子页表     (不参与翻译)
```

- **bits 63~39**：必须全部等于 bit 38（符号扩展），否则触发 page fault。xv6 将 bit 38 固定为 0，因此只用低 256 GB 虚拟地址空间（`MAXVA = 1L << 38`）。
- **VPN[2:0]**：共 27 位虚拟页号，每 9 位索引一级页表（2^9 = 512 个 PTE）
- **offset**：12 位页内偏移，直接拼到物理地址末尾

### Sv39 物理地址格式（Figure 4.20）

```
 55            30 29        21 20        12 11         0
┌────────────────┬────────────┬────────────┬────────────┐
│    PPN[2]      │  PPN[1]    │  PPN[0]    │ page offset│
│    26 bits     │  9 bits    │  9 bits    │  12 bits   │
└────────────────┴────────────┴────────────┴────────────┘
```

物理地址共 56 位（44 位 PPN + 12 位 offset），支持最大 64 PiB 物理内存。

### PX 宏和位布局

```c
// kernel/riscv.h
#define PGSHIFT         12
#define PXMASK          0x1FF        // 9 bits
#define PXSHIFT(level)  (PGSHIFT + (9 * (level)))
#define PX(level, va)   ((((uint64)(va)) >> PXSHIFT(level)) & PXMASK)
```

展开计算：

| 宏 | 右移位数 | 取 VA 的哪些位 | 用途 |
|---|---|---|---|
| `PX(2, va)` | 12 + 18 = **30** | **bits 38..30** | 索引 Level-2（根页表） |
| `PX(1, va)` | 12 + 9 = **21** | **bits 29..21** | 索引 Level-1（中间页表） |
| `PX(0, va)` | 12 + 0 = **12** | **bits 20..12** | 索引 Level-0（叶子页表） |
| — | — | **bits 11..0** | offset，不参与翻译 |

### 页表各级 entry 的覆盖范围

既然每级页表都有 512 个 entry，而 VA 被三级 VPN 逐级切分，每个 entry 能覆盖的地址范围可以用"向下乘"计算。

#### Level-0 entry：4KB（1 个物理页）

```
1 个 Level-0 PTE → 指向 1 个 4KB 物理页
                   offset = VA[11:0] = 12 bits = 4096 字节
```

这是翻译的终点：PTE 里存储物理页号，拼上 offset 得到最终物理地址。

#### Level-1 entry：2MB（512 个 4KB 页）

```
1 个 Level-1 PTE
  └─ 指向 1 个 Level-0 页表
       └─ 512 个 Level-0 PTE，每个映射 4KB

= 1 × 512 × 4KB
= 2MB
```

如果 Level-1 的 PTE 直接设为 leaf PTE（设置 R/W/X 权限，而非仅 `PTE_V` 指向下级），这个 PTE 本身就可以映射一个 **2MB megapage**——物理地址由 Level-1 PTE 的 PPN[1] + PPN[0]（共 18 bits）拼上 VA[20:0]（21 位 offset）构成。

#### Level-2 entry：1GB（512 个 2MB 区域）

```
1 个 Level-2 PTE（根页表 entry）
  └─ 指向 1 个 Level-1 页表
       └─ 512 个 Level-1 PTE，每个指向 1 个 Level-0 页表
            └─ 512 个 Level-0 PTE，每个映射 4KB

= 1 × 512 × 512 × 4KB
= 262144 × 4KB
= 1GB
```

从 VA 位布局也能直接验证：VPN[2] 有 9 bits，区分 512 个区域，而总 VA 空间是 512GB，所以 `512GB ÷ 512 = 1GB`，每个根页表 entry 正好管辖 1GB。

同理，如果 Level-2 的 PTE 直接设为 leaf PTE，可以映射一个 **1GB gigapage**（当然 xv6 没用）。

#### 一张图总结

```
VA (39 bits)
┌──── VPN[2] ────┬──── VPN[1] ────┬──── VPN[0] ────┬─── offset ───┐
│   9 bits        │   9 bits        │   9 bits        │  12 bits      │
│   512 entries   │   512 entries   │   512 entries   │  4096 bytes   │
└───────┬─────────┴───────┬─────────┴───────┬─────────┴──────┬───────┘
        │                 │                 │                │
        ▼                 ▼                 ▼                ▼
  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Level 2  │     │ Level 1  │     │ Level 0  │     │ 4KB page │
  │ 根页表   │────▶│ 中间页表 │────▶│ 叶子页表 │────▶│ physical │
  │          │     │          │     │          │     │  page    │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘
  1 entry covers   1 entry covers   1 entry covers
     1GB               2MB               4KB
  (gigapage)       (megapage)        (普通页)
```

### 根页表实际覆盖了哪些地址？

根页表 `kernel_pagetable` 共 512 个 entry，每个 entry 的管辖范围由 VPN[2]（即 VA[38:30]）决定。xv6 将 `MAXVA` 定义为 `1L << 38`（即 256 GB），只使用 VPN[2] = 0x000 ~ 0x0FF（bit 38 = 0 的低半区）：

```
VPN[2] = VA[38:30]  →  对应的 1GB VA 范围              xv6 用途
──────────────────────────────────────────────────────────────────
  0x000 (0)    →  0x0000000000 - 0x003FFFFFFF     用户进程空间,
                                                    UART0 (0x10000000),
                                                    VIRTIO0 (0x10001000),
                                                    PLIC (0x0C000000)
  0x001 (1)    →  0x0040000000 - 0x007FFFFFFF
  0x002 (2)    →  0x0080000000 - 0x00BFFFFFFF     KERNBASE (0x80000000)
                                                    内核代码+数据+RAM (128MB)
  0x003 (3)    →  0x00C0000000 - 0x00FFFFFFFF     PHYSTOP (0x88000000) 在此范围内
  ...
  0x0FF (255)  →  0x3FC0000000 - 0x3FFFFFFFFF     TRAMPOLINE (0x3FFFFFFFF000),
                                                    TRAPFRAME, KSTACK
                                                    MAXVA=0x4000000000 是上界
──────────────────────────────────────────────────────────────────
  0x100 ~ 0x1FF  →  >= MAXVA (0x4000000000)         xv6 不使用（超出 MAXVA）
```

注意 UART0（`0x10000000`）、PLIC（`0x0C000000`）都在 VPN[2]=0（0~1GB 区域），和用户空间在同一个 1GB entry 内——但**内核页表和用户页表是两张独立的表**，互不冲突。内核页表中这个 VA 映射到设备寄存器，用户页表中这个 VA 范围可能是用户内存或未映射。

xv6 只用了其中少数几个 entry。**大部分 entry 是空的**——对应的 PTE 的 V 位为 0，其下的 Level-1 和 Level-0 页表根本不存在。这正是三级页表的稀疏性优势：只为实际用到的地址区域分配下级页表，不用的不占内存。

拿 PLIC（`0x0C000000`）举例：
- `VPN[2] = 0x0C000000 >> 30 = 0x00` → 根页表第 0 个 entry
- 内核在 `kvmmap` PLIC 时，`walk()` 发现这个 entry 为空 → `kalloc()` 一个 Level-1 页表
- 继续 walk，发现 Level-1 的对应 entry 为空 → `kalloc()` 一个 Level-0 页表
- 在 Level-0 的对应 PTE 填入 PLIC 的物理地址和权限

PLIC 的 64MB 映射到底需要分配多少页表页？具体计算：

```
64MB ÷ 4KB = 16384 个页

1 个 Level-0 页表 = 512 个 PTE → 覆盖 2MB
16384 页 ÷ 512 = 32 个 Level-0 页表

1 个 Level-1 页表 = 512 个 PTE → 覆盖 1GB
这 32 个 Level-0 只需要 32 个 Level-1 entry
→ 都在同一个 Level-1 页表内

合计：1 个 Level-1 + 32 个 Level-0 = 33 页 ≈ 132KB
```

对比：如果这 1GB 区域全部用 4KB 页映射满，需要 1 + 512 = 513 页 ≈ 2MB。实际只用 33 页，其余 960MB 的地址空间没有任何页表分配——对应的 PTE 全是空的（V=0）。

更关键的是，根页表 512 个 entry 中绝大多数**连 Level-1 页表都不存在**。`walk()` 走到 V=0 的 entry 时立即返回，底下没有任何内存开销。这就是三级页表的稀疏性优势：只为实际使用的地址区间分配页表。

### 为什么 Level-2 是根？"一级"和"三级"哪个说法对？

RISC-V 从**叶子往根数**：Level 号越大，该级 PTE 能映射的页越大：

- Level 0 PTE → 映射 **4KB** 页（一个普通页）
- Level 1 PTE → 可映射 **2MB** 区域（512 × 4KB，megapage）
- Level 2 PTE → 可映射 **1GB** 区域（512 × 2MB，gigapage）

所以 `kernel_pagetable` 作为最顶层：
- **从叶子数**：它是 Level-2（RISC-V 规范命名）
- **从根数**：它是"第一级"（直觉命名）

两种说法都对，描述的是同一张表。`walk()` 的循环方向印证了 RISC-V 的命名：

```c
for (int level = 2; level > 0; level--) {  // 2 → 1 → 0，从根向下走
    pte_t *pte = &pagetable[PX(level, va)];
    ...
}
```

### 中间级页表的按需分配

`kvmmake()` 里只有一个显式的 `kalloc()`——分配根页表。那 Level-1 和 Level-0 的页表页在哪里？

答案在 `walk()` 函数（`kernel/vm.c:98`）：

```c
// walk 遍历三级页表，alloc!=0 时按需分配缺失的页表页
pte_t *
walk(pagetable_t pagetable, uint64 va, int alloc)
{
    for (int level = 2; level > 0; level--) {     // 遍历 Level-2 → Level-1
        pte_t *pte = &pagetable[PX(level, va)];
        if (*pte & PTE_V) {
            pagetable = (pagetable_t)PTE2PA(*pte); // 已存在，继续往下
        } else {
            if (!alloc || (pagetable = (pde_t *)kalloc()) == 0)
                return 0;                           // ← 不存在就分配！
            memset(pagetable, 0, PGSIZE);
            *pte = PA2PTE(pagetable) | PTE_V;      // 写入本级 PTE，指向新页
        }
    }
    return &pagetable[PX(0, va)];  // 返回 Level-0 的 PTE 指针
}
```

调用链：`kvmmap()` → `mappages()` → `walk(pgtbl, va, alloc=1)`

每次 `kvmmap` 一个地址区域时，`walk` 检查从根到叶子的路径上是否每一级页表都存在。不存在的就 `kalloc()` 补上。所以：

- **Level-2**：`kvmmake` 显式分配（根页表）
- **Level-1**：第一次访问对应 1GB 区域时，`walk()` 自动分配
- **Level-0**：同理，按需分配

### PTE 格式（Figure 4.21）

```
63  62 61 60   54 53   28 27   19 18   10 9  8  7  6  5  4  3  2  1  0
┌──┬─────┬─────────┬────────┬────────┬────────┬────┬──┬──┬──┬──┬──┬──┬──┬──┐
│N │PBMT │Reserved │PPN[2]  │PPN[1]  │PPN[0]  │RSW │D │A │G │U │X │W │R │V │
│1 │  2  │   7     │  26    │   9    │   9    │ 2  │1 │1 │1 │1 │1 │1 │1 │1 │
└──┴─────┴─────────┴────────┴────────┴────────┴────┴──┴──┴──┴──┴──┴──┴──┴──┘
```

bits 9-0 各标志位含义：V(有效), R(读), W(写), X(执行), U(用户可访问), G(全局), A(已访问), D(脏), RSW(OS 保留)。

每个 PTE 8 字节，512 个 PTE 正好填满一个 4KB 页表页。

### xv6 不使用大页（superpage）

Sv39 规范允许**任何一级的 PTE 成为叶子 PTE**，从而支持 2MB megapage 和 1GB gigapage。但 xv6 没有使用：

```c
// walk() 永远走满三级，不会中途停下
for (int level = 2; level > 0; level--) {
    ...
}
return &pagetable[PX(0, va)];  // 一定返回 Level-0 的 PTE
```

原因：
- xv6 是教学系统，128MB 物理内存 + 4KB 页已经足够
- 大页增加了分配器和管理复杂度
- 每个 PTE 都指向下一级页表（只设 `PTE_V`），不设 R/W/X 权限——因此不会是叶子 PTE

## 要点

- Sv39 = 三级页表（Sv32=两级, Sv48=四级, Sv57=五级），编号从叶子往根数
- VA bits 38..30 → Level-2, bits 29..21 → Level-1, bits 20..12 → Level-0, bits 11..0 → offset
- 根页表由 `kvmmake` 显式分配，下级页表由 `walk(alloc=1)` 按需动态分配
- xv6 只用 4KB 页，未使用 megapage/gigapage
- 每个页表页 512 个 8 字节 PTE，正好 4KB

## 参考

- `kernel/riscv.h`: PX, PXSHIFT, PXMASK, MAKE_SATP, PTE 标志位
- `kernel/vm.c`: walk, mappages, kvmmake
- RISC-V Privileged Specification v20211203:
  - Section 4.1.11: satp register
  - Section 4.3.2: Virtual Address Translation Process (the Sv32 algorithm, reused by Sv39 with LEVELS=3)
  - Section 4.4: Sv39 — Figure 4.19 (VA), Figure 4.20 (PA), Figure 4.21 (PTE)
