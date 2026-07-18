# xv6-riscv 启动流程：从加电到 Shell

本文档追踪 xv6-riscv 从上电到出现 shell 提示符的完整路径。

## 启动三阶段（地址空间视角）

### 阶段 1：纯物理地址（M-mode）

```
QEMU 加载内核到物理地址 0x80000000
         ↓
    entry.S @ 0x80000000（物理）
         ↓
    start.c  @ 0x80000000（物理）
         ↓
    w_satp(0)  ← 显式禁用分页
```

此时 satp=0，MMU 关闭，所有地址都是物理地址。

### 阶段 2：恒等映射（S-mode，分页开启）

`main()` 调用 `kvminit()` 建立**恒等映射页表**：

```c
// 虚拟地址 == 物理地址
kvmmap(kpgtbl, KERNBASE, KERNBASE, ...);          // 0x80000000
kvmmap(kpgtbl, UART0, UART0, ...);                 // 0x10000000
kvmmap(kpgtbl, PLIC, PLIC, ...);                   // 0x0C000000
kvmmap(kpgtbl, TRAMPOLINE, (uint64)trampoline, ...); // 唯一的非恒等映射
```

然后 `kvminithart()` 写 satp 寄存器打开分页。分页开启后，代码中的地址变为虚拟地址，但经页表翻译后落回同一物理地址——代码无感知。

### 阶段 3：用户进程独立页表

每个用户进程有独立页表，虚拟地址从 `0x0` 开始：

```
User VA layout:
  0x0 ─── text (代码段)
         data + bss
         stack (固定大小)
         heap (可扩展，通过 sbrk)
         ...
         TRAPFRAME
  MAXVA - PGSIZE ─── TRAMPOLINE (和内核共享同一物理页)
```

对比：

| | 链接基址 | 运行时虚拟地址 | 物理地址 |
|---|---|---|---|
| 内核 | `0x80000000` | `0x80000000` (恒等映射) | `0x80000000` |
| 用户程序 | `0x0` | `0x0` 起 | 内核动态分配 |
| Trampoline | 嵌入内核 .text 中 | `MAXVA - PGSIZE` | 内核中 trampoline 物理页 |

## PMP（物理内存保护）

在 `start.c` 中，M-mode 跳转到 S-mode 之前，需要配置 PMP：

```c
w_pmpaddr0(0x3fffffffffffffull);
w_pmpcfg0(0xf);  // TOR 模式, R+W+X, 不锁定
```

RISC-V 的 PMP 允许 M-mode 控制低特权级能访问哪些物理地址。这两行：
- `pmpcfg0 = 0xf`：地址匹配模式 TOR（Top-of-Range），权限 R+W+X
- `pmpaddr0 = 0x3fffffffffffffull`：地址上界覆盖整个物理地址空间
- TOR 范围：`[0, pmpaddr0)`，对 entry 0 来说下界隐含为 0

**含义：允许 S-mode 访问全部物理内存。** 不配置的话，mret 切换到 S-mode 后第一条指令就会触发访问异常。

## Sv39 MMU

xv6 使用 RISC-V 的 **Sv39** 分页方案（39 位虚拟地址，三级页表，4KB 页面）：

```
63 bits   39 bits                          0 bits
┌─ 全0 ──┬── L2 ──┬── L1 ──┬── L0 ──┬── offset ──┐
  25 bits   9 bits   9 bits   9 bits    12 bits
```

```c
#define SATP_SV39 (8L << 60)
#define MAKE_SATP(pagetable) (SATP_SV39 | (((uint64)pagetable) >> 12))
#define MAXVA (1L << (9 + 9 + 9 + 12 - 1))  // 256 GB
```

- 每级页表 512 个 PTE（9 位索引），每页正好 512×8=4096 字节
- MAXVA = 256 GB，只用 Sv39 的一半空间（最高位固定为 0，避免符号扩展问题）
- PTE 标志：V(有效), R(读), W(写), X(执行), U(用户可访问)

### 三级页表遍历（walk 函数）

```
va → PX(2,va) 索引 L2 根页表 → 得到 L1 页表物理地址
   → PX(1,va) 索引 L1 页表   → 得到 L0 页表物理地址
   → PX(0,va) 索引 L0 页表   → 返回最终 PTE 指针
```

某级页表不存在且 alloc!=0 时，分配新页补上。

## 位置无关代码（PIC/PIE）

xv6 **不使用** PIC/PIE，且明确禁用（`-fno-pie -no-pie`）。原因：
- 内核加载地址固定（QEMU 加载到 `0x80000000`，= 链接地址）
- 用户程序虚拟地址固定（`0x0` 起，每个进程独立页表互不冲突）
- 没有共享库、没有 KASLR，不需要运行时重定位

## 内核到用户态的切换

### 第一个用户进程的创建

```
main() → userinit() → allocproc() → context.ra = forkret, state = RUNNABLE
```

### scheduler() 调度到 forkret

```
scheduler() → 找到 init (RUNNABLE) → swtch(&c->context, &p->context)
  → ret 到 forkret (在进程自己的内核栈上)
```

### forkret 做的事

```c
void forkret(void) {
    // 首次运行:
    fsinit(ROOTDEV);                                       // 挂载文件系统
    p->trapframe->a0 = kexec("/init", ...);                // 加载 /init

    // 每次从内核返回用户态:
    prepare_return();                                      // 设置 sstatus.SPP=User
    uint64 satp = MAKE_SATP(p->pagetable);
    ((void (*)(uint64))(TRAMPOLINE + (userret - trampoline)))(satp);  // 跳入 trampoline
}
```

### prepare_return() 的关键操作

```c
// 1. sstatus.SPP = 0 → sret 后将进入 User mode（而非 Supervisor）
x &= ~SSTATUS_SPP;
x |= SSTATUS_SPIE;     // 进入用户态后开中断

// 2. sepc = 用户程序入口
w_sepc(p->trapframe->epc);  // = elf.entry = ulib.c 的 start()

// 3. stvec = uservec（为下次用户→内核 trap 做准备）
w_stvec(TRAMPOLINE + (uservec - trampoline));

// 4. 在 trapframe 中保存内核信息，供下次 uservec 使用
p->trapframe->kernel_satp = r_satp();
p->trapframe->kernel_sp = p->kstack + PGSIZE;
p->trapframe->kernel_trap = (uint64)usertrap;
```

### trampoline.S 的 userret

```asm
userret:
    csrw satp, a0          # 切换到用户页表
    sfence.vma

    # 从 TRAPFRAME 恢复所有用户寄存器
    ld ra, 40(a0)
    ld sp, 48(a0)
    ...                     # gp, tp, t0-t6, s0-s11, a1-a7
    ld a0, 112(a0)         # 最后恢复 a0

    sret                   # CPU: SPP=User → 切到 U-mode, PC=sepc
```

`sret` 执行后，CPU 在 **User mode**，PC = sepc = `start()` 的地址。

### 用户态启动

```c
// ulib.c
void start(int argc, char **argv) {
    extern int main(int argc, char **argv);
    r = main(argc, argv);  // 调用用户程序的 main()
    exit(r);
}
```

### 用户↔内核往返机制

之后每次系统调用/中断/缺页都走 trampoline：

```
用户态                      内核态
  │                          ▲
  │ ecall                    │ sret
  ▼                          │
uservec ──保存寄存器──▶ usertrap()
(trampoline.S)   切页表   (trap.c)
                           │
                           ├─ syscall() / devintr() / vmfault()
                           │
                           ▼
                        prepare_return() → userret → sret → 用户态
```

trampoline 页在用户页表和内核页表中映射到**同一虚拟地址** (`TRAMPOLINE = MAXVA - PGSIZE`)，因此切换 satp 时 trampoline 代码本身不受影响。

## 文件系统加载与 Shell 启动

### 内核初始化链路

```c
main() {
    consoleinit();         // UART 控制台
    kinit();               // 物理页分配器
    kvminit();             // 内核页表
    kvminithart();         // 打开分页
    procinit();            // 进程表
    trapinit()/trapinithart(); // 陷阱向量
    plicinit()/plicinithart(); // 中断控制器
    binit();               // 缓冲区缓存
    iinit();               // inode 表
    fileinit();            // 文件描述符表
    virtio_disk_init();    // 磁盘驱动
    userinit();            // 第一个用户进程
    scheduler();           // 开始调度
}
```

### 磁盘 I/O 栈

```
exec/read/write 系统调用
    ↓
namei/readi/writei       [fs.c — inode 层]
    ↓
bread/bwrite              [bio.c — 缓冲区缓存]
    ↓
virtio_disk_rw            [virtio_disk.c — 磁盘驱动]
    ↓
QEMU virtio-blk 设备 → fs.img
```

### forkret 中的文件系统挂载

```c
fsinit(ROOTDEV) {
    readsb(dev, &sb);   // bread(dev, 1) 读第 1 块 = 超级块
    if (sb.magic != FSMAGIC) panic("invalid file system");
    initlog(dev, &sb);
    ireclaim(dev);
}
```

超级块在 fs.img 布局：
```
块 0: boot block (不用)
块 1: superblock (魔数、大小、各区域起始位置)
块 2~31: log blocks
块 32~: inodes → bitmap → data blocks
```

### kexec("/init") 加载 ELF

```c
kexec("/init", argv) {
    ip = namei("/init");                // 路径 → inode
    readi(ip, ..., &elf, sizeof(elf));  // 读 ELF 头
    pagetable = proc_pagetable(p);      // 新页表

    for (each program header) {
        uvmalloc(pagetable, ..., vaddr + memsz); // 分配内存
        loadseg(pagetable, vaddr, ip, off, filesz); // 读入段
    }

    // 分配用户栈，拷入 argv
    // 设入口
    p->trapframe->epc = elf.entry;  // = ulib.c:start
    p->trapframe->sp = sp;
}
```

### /init 启动 shell

```c
// user/init.c
main() {
    open("console", O_RDWR);  // fd 0
    dup(0);                    // fd 1 (stdout)
    dup(0);                    // fd 2 (stderr)

    for (;;) {
        pid = fork();
        if (pid == 0) {
            exec("sh", argv);  // 子进程变成 shell
        }
        wait(0);               // 父进程等 sh 退出后重启
    }
}
```

## 完整启动流程（一张图）

```
QEMU 加载 kernel @ 0x80000000
    ↓
entry.S (M-mode, 物理地址)
    ↓
start.c: 配置 PMP, 设 MPP=S, mepc=main, mret
    ↓
main() (S-mode, 物理地址): 初始化各子系统
    ↓
kvminit() + kvminithart(): 建立恒等映射页表, 打开分页
    ↓ (此时地址变为虚拟地址，但恒等映射所以无感)
virtio_disk_init(): 磁盘驱动就绪
    ↓
userinit() → allocproc() → context.ra = forkret
    ↓
scheduler() → swtch() → forkret()
    ↓
fsinit(ROOTDEV): 读超级块, 挂载文件系统
kexec("/init"): 从 fs.img 加载 ELF
    ↓
prepare_return() + userret + sret → User mode
    ↓
ulib.c:start() → /init:main()
    ↓
fork + exec("sh") → $ shell 提示符
```
