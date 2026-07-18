# 用户态程序的编译、链接与加载执行

## 编译方式：静态链接

xv6 的用户程序采用**完全静态链接**。每个 `user/_X` 在编译时就把所有依赖的代码链接进去了。

### 链接公式

```makefile
ULIB = user/ulib.o user/usys.o user/printf.o user/umalloc.o

_%: %.o $(ULIB) user/user.ld
    $(LD) $(LDFLAGS) -T user/user.ld -o $@ $< $(ULIB)
```

以 `_sh` 为例：

```
user/_sh:
  ├── sh.o        ← sh.c 的代码
  ├── ulib.o      ← C 库函数 (strcpy, memset, memmove, strlen, start 入口...)
  ├── usys.o      ← 系统调用桩 (fork, read, write, ..., 通过 ecall 进入内核)
  ├── printf.o    ← printf / fprintf 实现
  └── umalloc.o   ← malloc / free 实现 (基于 sbrk)
```

每个用户程序的 ELF 自包含所有代码，体积约 30~60 KB（含调试信息）。

### 用户链接脚本 (user.ld)

```
OUTPUT_ARCH("riscv")
SECTIONS {
    . = 0x0;                          # 虚拟地址从 0 开始
    .text   : { *(.text .text.*) }    # 代码段
    .rodata : { ... }                 # 只读数据 (16 字节对齐)
    .eh_frame : { ... }               # DWARF 异常处理表
    .data   : { ... }                 # 可读写数据 (页对齐)
    .bss    : { ... }                 # 未初始化数据
    PROVIDE(end = .);                 # 堆起始位置
}
```

没有 `ENTRY` 指令——入口地址由 ELF Header 的 `entry` 字段指定，`kexec()` 从这里读取。

## ELF 格式：标准但极简

### xv6 ELF 结构

```
user/_sh: ELF 64-bit LSB executable, UCB RISC-V, RVC, double-float ABI,
          version 1 (SYSV), statically linked, with debug_info, not stripped
```

ELF Header 关键字段：

| 字段 | 值 | 说明 |
|------|-----|------|
| Type | `EXEC` (2) | 固定地址可执行文件 |
| Machine | `EM_RISCV` (243) | RISC-V 64 位 |
| Entry | `0x98e` | ulib.c:start 的地址 |
| Program Headers | 2 个 | 仅两个 PT_LOAD 段 |

### Program Headers（加载段）

```
  Type   VirtAddr   MemSiz   Flags            内容
  LOAD   0x0000     0x1339   R E     .text + .rodata (代码 + 只读数据)
  LOAD   0x2000     0x0098   RW      .data + .bss    (可读写 + 未初始化)
```

### 与 Linux 用户程序的对比

| | xv6 `_sh` | Linux `/bin/ls` |
|---|---|---|
| ELF 类型 | `EXEC`（固定地址） | `DYN`（PIE，位置无关） |
| 链接方式 | **静态链接** | **动态链接** |
| Program Headers | **2 个** | **13 个** |
| 动态链接器 | 无 | `/lib64/ld-linux-x86-64.so.2` |
| 段类型 | 只有 PT_LOAD | PHDR, INTERP, DYNAMIC, NOTE, GNU_STACK, GNU_RELRO... |
| 符号表 | 保留（含 .debug_* 段） | stripped（已剥离） |
| 体积 | ~57 KB（含调试信息） | ~140 KB（无调试信息，但有 PLT/GOT） |

**格式本身是同一套 ELF 标准**——file 命令、readelf、objdump 对两者都能正确识别。区别在于使用的 ELF 特性子集不同。

## 加载执行：kexec()

[exec.c](kernel/exec.c#L28-L151) 的 `kexec()` 是 xv6 的 ELF 加载器，约 120 行：

### 加载流程

```
1. namei(path)          → 打开 ELF 文件 inode

2. readi(..., &elf, sizeof(elf))
                        → 读 ELF Header，校验 magic == ELF_MAGIC

3. proc_pagetable(p)    → 创建新的用户页表（替换旧页表）

4. for (each Program Header with type == PT_LOAD):
     uvmalloc(pagetable, ..., vaddr + memsz, flags2perm)
                        → 分配虚拟内存（含 .bss 的 memsz > filesz 部分）
     loadseg(pagetable, vaddr, ip, offset, filesz)
                        → 从文件读段内容到物理页

5. uvmalloc(pagetable, ..., USERSTACK * PGSIZE)
   uvmclear(...)        → 分配用户栈（栈底一个 guard page 不可访问）

6. copyout(pagetable, sp, argv, ...)
                        → 把 argv 字符串拷到用户栈

7. p->trapframe->epc = elf.entry   → 入口设为 ELF entry（即 start()）
   p->trapframe->sp = sp           → 栈指针

8. 提交: p->pagetable = pagetable
```

### 入口点 start()

```c
// ulib.c
void start(int argc, char **argv) {
    extern int main(int argc, char **argv);
    r = main(argc, argv);  // 调用用户程序的 main()
    exit(r);
}
```

ELF 的 entry 指向 `start()` 而非 `main()`——这样即使用户的 `main` 忘记调用 `exit()`，`start` 也会帮你调用。

### .bss 段如何处理

```c
// exec.c 中
uvmalloc(pagetable, ..., ph.vaddr + ph.memsz, flags);
loadseg(pagetable, ph.vaddr, ip, ph.off, ph.filesz);
```

- `uvmalloc` 分配 `memsz` 大小的内存，并用 `memset(p, 0, PGSIZE)` 清零
- `loadseg` 只读入 `filesz` 大小的文件内容
- `.bss` 在文件中不占空间（`filesz < memsz`），但加载后被自动清零

## 为什么不支持动态链接？

### 不是体积问题

实际上每个 xv6 用户程序的"冗余"部分只有 ~9 KB（ulib + printf + umalloc + usys）。fs.img 总共 2 MB，18 个程序装得下，这点冗余可以忽略。

### 真正原因：复杂度和教学定位

实现动态链接需要额外 ~2000+ 行代码：

```
ld.so 的核心组件:
  ├── ELF 符号解析                ~500 行
  ├── R_RISCV_* 重定位处理        ~500 行
  ├── 共享库 mmap 语义            ~300 行
  ├── PLT/GOT 机制                ~400 行
  ├── ld.so 自举                  ~300 行
  └── 惰性绑定 (lazy binding)     ~200 行
  合计: ~2000+ 行
```

而 xv6 整个内核才 ~10000 行。为了省那 9 KB 加 2000 行代码，完全背离了"只保留核心机制"的教学定位。

### 取舍逻辑

```
xv6:
  ✔ 静态链接，每个程序 ~50 KB      ← 简单，exec() ~170 行
  ✔ 文件系统 2 MB，所有程序装得下   ← 够用
  ✔ 关键目标: 让你看懂 ELF 加载全流程

Linux:
  ✔ 动态链接，每个程序 ~16 KB       ← 节省数百 MB 级磁盘/内存
  ✔ 代价: ld.so ~50000+ 行          ← 复杂但工程上值得
  ✔ 关键目标: 生产级性能和安全
```

## 完整链路总结

```
编译时 (Make):
  sh.c -→ sh.o ─┐
  ulib.c -→ ulib.o ─┐
  usys.S -→ usys.o  ├── $(LD) -T user/user.ld -→ user/_sh (ELF)
  printf.c -→ printf.o ─┤
  umalloc.c -→ umalloc.o ─┘

运行时 (exec):
  fork() → kexec("/sh")
    → 读 ELF Header，校验 magic
    → 遍历 PT_LOAD，分配内存，读入段内容
    → 设 trapframe->epc = elf.entry (= start)
    → prepare_return() + sret → User mode
      → start(argc, argv) → main() → Shell 开始运行
```
