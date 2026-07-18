# xv6-riscv Virtio 磁盘驱动与中断处理

## 什么是 Virtio

Virtio 是虚拟化 I/O 的**标准协议**。与传统模拟真实硬件（IDE/SATA）不同，virtio 设备知道自己是被虚拟的，因此协议极简化——不需要模拟硬件怪癖（定时、DMA 引擎、命令队列管理等）。

```
真实硬件模拟 (IDE/SATA)           Virtio
─────────────────────────      ─────────────────────
QEMU 模拟完整的 IDE 控制器      QEMU 提供 virtio 接口
寄存器时序、DMA、中断            共享内存 + 简洁通知机制
慢，代码量大                     快，xv6 驱动仅 326 行
```

## QEMU 中的设备接入

```makefile
# Makefile
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

xv6 通过 **MMIO**（Memory-Mapped I/O）访问设备，寄存器基址：`VIRTIO0 = 0x10001000`。

## 三大核心数据结构（Virtqueue）

驱动和设备通过三块**共享内存**通信：

```
┌──────────────────────────────────────┐
│          Descriptor Table (desc)     │  ← 驱动描述请求（每个 16 字节）
│  [0]   addr | len | flags | next     │
│  [1]   ...                           │
│  [NUM-1]                             │  NUM = 8
├──────────────────────────────────────┤
│        Available Ring (avail)        │  ← 驱动放入完成准备的描述符链头
│  flags | idx | ring[NUM] | unused    │
├──────────────────────────────────────┤
│          Used Ring (used)            │  ← 设备放入已完成的描述符链头
│  flags | idx | ring[NUM]             │
└──────────────────────────────────────┘
```

三者都是 `kalloc()` 分配的物理页，物理地址写入 MMIO 寄存器告知设备。

### 描述符结构

```c
struct virtq_desc {
    uint64 addr;   // 数据物理地址
    uint32 len;    // 数据长度
    uint16 flags;  // F_NEXT(1)=链式, F_WRITE(2)=设备写入此内存
    uint16 next;   // 链中下一个描述符的索引
};
```

### 磁盘请求头

```c
struct virtio_blk_req {
    uint32 type;      // VIRTIO_BLK_T_IN(0)=读, VIRTIO_BLK_T_OUT(1)=写
    uint32 reserved;
    uint64 sector;    // 扇区号 (512 字节为单位)
};
```

## 初始化流程 `virtio_disk_init()`

严格遵循 virtio spec 的状态机：

```
1. 验证设备 (MMIO 寄存器)
   MAGIC_VALUE == 0x74726976 ("virt")
   VERSION == 2, DEVICE_ID == 2 (block device)
   VENDOR_ID == 0x554d4551

2. 协商特性 (feature bits)
   驱动不支持的特性全部关掉:
   RO, SCSI, WCE, MQ, ANY_LAYOUT, EVENT_IDX, INDIRECT_DESC
   → 最简功能: 纯读/写

3. 分配并清零描述符表、avail 环、used 环
   将物理地址写入 MMIO 寄存器 (DESC_LOW/HIGH 等)

4. QUEUE_READY = 1
   STATUS |= DRIVER_OK

5. 标记所有 8 个描述符为 free
```

## 一次磁盘读写：三描述符链

每次磁盘 I/O 请求由三个描述符组成链：

```
┌─ desc[idx0]: 请求头 ────────────────┐
│  addr  = &disk.ops[idx0] (virtio_blk_req)
│  len   = 16 字节
│  flags = F_NEXT
│  next  = idx1
└──────────────┬──────────────────────┘
               │ next
               ▼
┌─ desc[idx1]: 数据缓冲区 ────────────┐
│  addr  = b->data (1024 字节)
│  len   = BSIZE
│  flags = F_NEXT | (读? F_WRITE : 0)
│  next  = idx2
└──────────────┬──────────────────────┘
               │ next
               ▼
┌─ desc[idx2]: 状态字节 ──────────────┐
│  addr  = &disk.info[idx0].status (1 字节)
│  len   = 1
│  flags = F_WRITE  (设备写 0 = 成功)
│  next  = 0 (链结束)
└─────────────────────────────────────┘
```

### `virtio_disk_rw()` 完整流程

```c
void virtio_disk_rw(struct buf *b, int write) {
    // 1. 块号 → 扇区号（每块 = 2 扇区，因为 BSIZE=1024, 扇区=512）
    sector = b->blockno * (BSIZE / 512);

    // 2. 分配 3 个空闲描述符（不够则 sleep 等待）
    alloc3_desc(idx);

    // 3. 填充请求头（类型、扇区号）
    buf0->type   = write ? VIRTIO_BLK_T_OUT : VIRTIO_BLK_T_IN;
    buf0->sector = sector;

    // 4. 串联三个描述符（如上图）
    disk.desc[idx[0]].addr  = (uint64)buf0;
    disk.desc[idx[0]].flags = VRING_DESC_F_NEXT;
    disk.desc[idx[0]].next  = idx[1];
    // ...

    // 5. 记录 buf，供中断处理使用
    b->disk = 1;
    disk.info[idx[0]].b = b;

    // 6. 把链头放入 avail 环
    disk.avail->ring[disk.avail->idx % NUM] = idx[0];
    disk.avail->idx += 1;  // 通知设备

    // 7. 写 MMIO 寄存器通知设备
    *R(VIRTIO_MMIO_QUEUE_NOTIFY) = 0;

    // 8. 睡眠等待中断（让出 CPU）
    while (b->disk == 1) {
        sleep(b, &disk.vdisk_lock);
    }

    // 9. 被中断唤醒后释放描述符链
    free_chain(idx[0]);
}
```

## 中断处理

### 为什么用中断而不是轮询？

磁盘 I/O 是毫秒级的，CPU 每纳秒执行一条指令——轮询浪费百万条指令。中断让 I/O 等待期间 CPU 能运行其他进程。

### 完整中断路径（4 层）

```
virtio 磁盘设备（QEMU 模拟）
    │  拉高 IRQ 线 #1 (VIRTIO0_IRQ)
    ▼
┌─ PLIC (Platform-Level Interrupt Controller) ──────┐
│  汇聚所有外设中断，路由到各 HART (CPU)            │
│  UART = IRQ 10,  VIRTIO DISK = IRQ 1              │
│                                                    │
│  plicinit():    设置各 IRQ 优先级 = 1              │
│  plicinithart(): 为本 HART 使能 IRQ 1 和 10       │
└──────────────────────┬─────────────────────────────┘
                       │  外部中断信号
                       ▼
┌─ RISC-V CPU ──────────────────────────────────────┐
│  scause = 0x8000000000000009                       │
│  (S-mode external interrupt)                       │
│  CPU 硬件: sepc←PC, sstatus.SPP←当前特权级,         │
│            跳转到 stvec                            │
└──────────────────────┬─────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           │                       │
    在用户态发生               在内核态发生
           │                       │
           ▼                       ▼
     uservec                  kernelvec
  (trampoline.S)            (kernelvec.S)
    保存用户寄存器             保存内核寄存器
    切到内核页表               (同一内核栈)
           │                       │
           ▼                       ▼
      usertrap()               kerneltrap()
           └───────────┬───────────┘
                       ▼
                   devintr()
                   检查 scause:
         0x8000000000000009 → plic_claim()
                              返回 irq
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
    irq == VIRTIO0_IRQ (1)   irq == UART0_IRQ (10)
           │                       │
           ▼                       ▼
   virtio_disk_intr()          uartintr()
           │
    plic_complete(irq)
```

### devintr() — 中断分发

```c
int devintr() {
    uint64 scause = r_scause();

    if (scause == 0x8000000000000009L) {
        // S-mode 外部中断 → 问 PLIC 是哪个设备
        int irq = plic_claim();

        if (irq == UART0_IRQ) {
            uartintr();
        } else if (irq == VIRTIO0_IRQ) {
            virtio_disk_intr();
        }

        plic_complete(irq);  // 告知 PLIC 中断处理完毕
        return 1;
    } else if (scause == 0x8000000000000005L) {
        // S-mode 定时器中断
        clockintr();
        return 2;
    }
    return 0;
}
```

### virtio_disk_intr() — 唤醒等待者

```c
void virtio_disk_intr() {
    // 1. ACK 中断
    *R(VIRTIO_MMIO_INTERRUPT_ACK) = *R(VIRTIO_MMIO_INTERRUPT_STATUS) & 0x3;

    // 2. 遍历 used ring 中设备已完成的所有请求
    while (disk.used_idx != disk.used->idx) {
        id = disk.used->ring[disk.used_idx % NUM].id;

        // 验证 status == 0 (成功)
        if (disk.info[id].status != 0)
            panic("virtio_disk_intr status");

        // 唤醒等待此 buf 的进程
        struct buf *b = disk.info[id].b;
        b->disk = 0;
        wakeup(b);

        disk.used_idx++;
    }
}
```

## 读写链路（从文件名到磁盘）

```
用户态: read()/write() 系统调用
    ↓
sys_read()/sys_write()         [sysfile.c]
    ↓
fileread()/filewrite()         [file.c — 文件描述符层]
    ↓
readi()/writei()               [fs.c — inode 层]
    ↓
bread()/bwrite()               [bio.c — 缓冲区缓存]
    ↓
virtio_disk_rw()               [virtio_disk.c — 磁盘驱动]
    构造 3 描述符链 → avail ring → QUEUE_NOTIFY → sleep
    ↓
    ... 磁盘工作时 CPU 运行其他进程 ...
    ↓
virtio_disk_intr()             [中断 → PLIC → trap → devintr]
    wakeup(b)                   [唤醒等待进程]
    ↓
数据在 b->data 中，返回上层
```

## PLIC (Platform-Level Interrupt Controller)

```c
// memlayout.h
#define PLIC 0x0c000000L

// 初始化
plicinit():     // 全局: 设各 IRQ 优先级
  *(PLIC + UART0_IRQ * 4) = 1;
  *(PLIC + VIRTIO0_IRQ * 4) = 1;

plicinithart(): // 每 CPU: 使能本 hart 的中断源
  *(PLIC_SENABLE(hart)) = (1 << UART0_IRQ) | (1 << VIRTIO0_IRQ);
  *(PLIC_SPRIORITY(hart)) = 0;  // 优先级阈值 = 0 (接收所有中断)

// 运行时
plic_claim():    // 读 PLIC_SCLAIM → 返回待处理的 IRQ 号
plic_complete(): // 写 PLIC_SCLAIM → 告知 PLIC 中断已处理
```

## MMIO 寄存器偏移（virtio spec）

```c
#define VIRTIO_MMIO_MAGIC_VALUE      0x000  // "virt"
#define VIRTIO_MMIO_VERSION          0x004  // 应为 2
#define VIRTIO_MMIO_DEVICE_ID        0x008  // 2 = block device
#define VIRTIO_MMIO_VENDOR_ID        0x00c
#define VIRTIO_MMIO_DEVICE_FEATURES  0x010
#define VIRTIO_MMIO_DRIVER_FEATURES  0x020
#define VIRTIO_MMIO_QUEUE_SEL        0x030
#define VIRTIO_MMIO_QUEUE_NUM_MAX    0x034
#define VIRTIO_MMIO_QUEUE_NUM        0x038
#define VIRTIO_MMIO_QUEUE_READY      0x044
#define VIRTIO_MMIO_QUEUE_NOTIFY     0x050
#define VIRTIO_MMIO_INTERRUPT_STATUS 0x060
#define VIRTIO_MMIO_INTERRUPT_ACK    0x064
#define VIRTIO_MMIO_STATUS           0x070
#define VIRTIO_MMIO_QUEUE_DESC_LOW   0x080
// ...
```

寄存器访问宏：

```c
#define R(r) ((volatile uint32 *)(VIRTIO0 + (r)))
// 用法: *R(VIRTIO_MMIO_MAGIC_VALUE) 读, *R(VIRTIO_MMIO_STATUS) = status 写
```
