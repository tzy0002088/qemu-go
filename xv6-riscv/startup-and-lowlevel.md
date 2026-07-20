---
title: "内核启动与底层细节问答"
date: 2026-07-21
tags: [xv6, riscv, linker, startup, memory-model, spinlock]
status: notes
---

## 背景

逐行阅读 xv6-riscv 内核代码（`kernel.ld`、`entry.S`、`start.c`、`main.c`、`kalloc.c`、`uart.c`、`spinlock.c` 等），对底层细节产生一系列提问。

## 提问与理解

### 1. 链接脚本 — 为什么基址是 0x80000000？

QEMU RISC-V `virt` 板卡的物理内存布局中，`0x80000000` 是 RAM 的起始地址。以下是 IO 设备区间：

| 地址 | 用途 |
|---|---|
| `0x00001000` | boot ROM |
| `0x02000000` | CLINT |
| `0x0C000000` | PLIC |
| `0x10000000` | UART0 |
| `0x10001000` | virtio 磁盘 |
| **`0x80000000`** | **内核加载地址（RAM 起始）** |

QEMU `-kernel` 把 ELF 加载到 `0x80000000`，然后让各 CPU hart 跳到这里执行。链接脚本 `. = 0x80000000` 保证所有符号地址与加载位置一致。

### 2. `.srodata` 注释 — "do not need to distinguish this from .rodata"

`.srodata` = Short Read-Only Data。RISC-V 编译器为利用 `gp` 全局指针寄存器做小数据优化（±2KB 内单指令访问），把 ≤某阈值的数据放进 `.sdata`/`.srodata`。xv6 不开启此优化，所以把 `.srodata` 合并到 `.rodata` 一视同仁。

### 3. `la` 伪指令 → `auipc` + `addi`

RISC-V 指令 32 位定长，装不下完整地址。`la` 展开为：
- `auipc rd, imm20`：rd = PC + (imm20 << 12)，构造 PC-相对地址的高 20 位
- `addi rd, rd, imm12`：补上低 12 位

汇编器自动处理符号扩展边界：若低 12 位偏移 > 2047（超 12 位有符号范围），则 auipc 向上借 1（多 0x1000），addi 用负偏移补回。如 `stack0 = 0x80007870`，PC = `0x80000000`，偏移 `0x7870`：
- `auipc sp, 0x8` → sp = `0x80008000`
- `addi sp, sp, -1936` → sp = `0x80007870` ✓

### 4. `-mcmodel=medany` 为什么必须？

内核链接在 `0x80000000`，bit 31 = 1。

`medlow`：用 `lui` + `addi` 绝对寻址，符号必须在 [0, 2GB)。`lui` 在 RV64 中对 bit 31 = 1 的地址会符号扩展到高 32 位全 1（`0xFFFFFFFF_80000xxx`），地址错误。

`medany`：用 `auipc` + `addi` 做 PC 相对寻址。虽然也是 ±2GB 范围，但窗口跟着 PC 走，内核全部代码在这窗口内。

medlow 窗口钉在地址 0，medany 窗口跟着 PC 走。

### 5. `ENTRY(_entry)` — 链接器入口点

`ENTRY` 指令将 `_entry` 的地址写入 ELF 头的 `e_entry` 字段。QEMU `-kernel` 读取此字段，让各 CPU hart 跳到该地址。

`_entry` 定义于 `entry.S`，由 `.text : { kernel/entry.o(_entry) ... }` 确保放在 `.text` 段最开头（`0x80000000`）。

### 6. `w_medeleg(0xffff)` / `w_mideleg(0xffff)` / `w_sie(...)` — 委托中断给 S 模式

RISC-V 默认所有 trap 上报 M 模式。xv6 内核跑在 S 模式，需要在 `start()` 中把中断/异常委托下去：

- **`medeleg`**：Machine Exception Delegation。`0xffff` = 低 16 种异常（非法指令、缺页、ecall 等）全部交给 S 模式。
- **`mideleg`**：Machine Interrupt Delegation。`0xffff` = 软件中断(bit 1)、定时器中断(bit 5)、外部中断(bit 9) 交给 S 模式。
- **`sie |= SEIE | STIE`**：在 S 模式下开启外部中断和定时器中断。

然后 `mret` 切换到 S 模式执行 `main()`。

### 7. `MENVCFG_ADUE` — 硬件自动更新页表 A/D 位

RISC-V PTE 的第 6 位是 A (Accessed)、第 7 位是 D (Dirty)。Svadu 扩展允许硬件自动置位。`menvcfg` bit 61 (ADUE) 启用此功能。设置后页表 A/D 位全由硬件管理，xv6 无需在 trap handler 中处理 A/D page fault。必须在 M 模式设置，S 模式无权访问 `menvcfg`。

### 8. `CONSOLE = 1` — 控制台主设备号

xv6 沿用 Unix 设备号约定：`(major, minor)`。`major` 选择驱动（`devsw[major]`），`minor` 区分同类设备实例。`CONSOLE = 1` 是控制台的主设备号。`devsw[0]` 空着，`devsw[1].read = consoleread`，`devsw[1].write = consolewrite`。`init.c` 通过 `mknod("console", CONSOLE, 0)` 创建设备节点。

### 9. `struct cpu` — 每个成员的含义

```c
struct cpu {
    struct proc *proc;      // 此 CPU 上正在运行的进程，NULL = 调度器在跑
    struct context context; // 调度器线程的断点，swtch() 切回来时恢复到此
    int noff;               // push_off() 嵌套深度（支持多层关中断）
    int intena;             // 第一次 push_off() 前中断是否开着（决定 pop_off 到底层时是否恢复）
};
```

### 10. `push_off()` 实际上关了中断 — `csrrc` 的秘密

`push_off()` 表面只记状态，中断开关藏在 `rc_sstatus(SSTATUS_SIE)` 中：

```c
rc_sstatus(uint64 x) {
    __asm__ __volatile__("csrrc %0, sstatus, %1" ...);
    //                    ↑ CSR Read and Clear
    //                    返回旧值的同时，清零 sstatus 中 x 指定的位
}
```

`csrrc` 是原子指令：读 sstatus 旧值到返回值 → 清 SIE 位 → 关中断。一条指令同时完成"读状态"和"关中断"。

`pop_off()` 回到深度 0 且 `intena==1` 时调用 `intr_on()` 恢复。

在 `uartputc_sync` 中调用 `push_off()`，确保轮询写 UART 期间不被 UART 中断处理器抢占，防止寄存器操作交错。

### 11. `PGROUNDUP` — 4KB 页对齐

```c
#define PGROUNDUP(sz)  (((sz) + PGSIZE - 1) & ~(PGSIZE - 1))
```

内核 image 结束地址 `end` 未必是 4KB 对齐。物理页分配器要求每页起始地址对齐 4KB（`kfree` 会 panic 检查）。`freerange` 先把 `end` 向上对齐到下一个 4KB 边界，再逐页 `kfree` 加入空闲链表。

条件 `p + PGSIZE <= pa_end` 保证释放的是完整页，尾部不足一页的丢弃。

### 12. `__atomic_thread_fence(__ATOMIC_SEQ_CST)` — 多核启动内存屏障

CPU 0 初始化全部子系统后写 `started = 1`，其他 CPU 自旋等待 `started == 1` 后开始各自的 per-hart 初始化。

fence 构成"释放-获取"对：

- **CPU 0 侧**（释放 fence）：所有初始化写必须在 fence 处排干，`started = 1` 不可越过 fence 提前对其他 CPU 可见。
- **其他 CPU 侧**（获取 fence）：观察到 `started == 1` 后，后续的读（读页表、读 PLIC 配置等）不可越过 fence 提前到 `started` 检查之前。

`volatile` 只防止编译器优化掉循环读取，管不了 CPU 硬件重排。`fence rw,rw` 刷新 store buffer，保证多核间的写可见性。

## 要点

- `0x80000000` 是 QEMU virt 板 RAM 起始，内核必须链接到此地址
- `.srodata`/`.sdata`/`.sbss` 是 gp 相对寻址的小数据段，xv6 合并到普通段中
- `la` 展开为 `auipc`+`addi`，汇编器自动处理符号扩展边界
- `-mcmodel=medany` 用 PC 相对寻址，避开了 medlow 的 `lui` 符号扩展陷阱
- `medeleg`/`mideleg` 把中断异常从 M 模式委托给 S 模式
- `ADUE` 让硬件自动更新页表 A/D 位，简化 VM 代码
- `push_off` 通过 `csrrc` 原子读并清 SIE 位，实现关中断+记状态
- `PGROUNDUP` 上对齐到 4KB 页边界，保证物理页分配器正确性
- `atomic_thread_fence` 保证多核启动时 CPU 0 的初始化对其它 CPU 全局可见

## 参考

- `kernel/kernel.ld` — 链接脚本
- `kernel/start.c` — M 模式启动、中断委托、ADUE 设置
- `kernel/main.c` — 多核启动
- `kernel/entry.S` — 内核入口汇编
- `kernel/spinlock.c` — push_off/pop_off
- `kernel/kalloc.c` — 物理内存分配器
- `kernel/uart.c` — uartputc_sync
- `kernel/riscv.h` — RISC-V 宏和 CSR 操作
- `kernel/memlayout.h` — 物理内存布局
- `kernel/file.h` — CONSOLE 定义
