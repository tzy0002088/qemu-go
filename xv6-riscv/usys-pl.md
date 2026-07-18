# usys.pl：系统调用桩生成器

## 解决的问题

用户程序是普通 C 代码：

```c
pid = fork();
```

`fork` 声明在 [user.h](user/user.h#L6)，但**没有函数体**——它必须陷入内核才能执行。那谁来提供这个函数体？

答案：`usys.pl` 自动生成的汇编桩（stub）。

## 工作原理

### usys.pl 做什么

[usys.pl](user/usys.pl) 是一个 Perl 脚本，在**构建时**被 Make 调用：

```makefile
user/usys.S : user/usys.pl
    perl user/usys.pl > user/usys.S

user/usys.o : user/usys.S
    $(CC) $(CFLAGS) -c -o user/usys.o user/usys.S
```

它对每个系统调用生成**完全相同的三指令模板**：

```asm
.global <name>
<name>:
    li a7, SYS_<name>    # ① 系统调用号装入 a7 寄存器
    ecall                 # ② 触发陷阱，CPU 切到内核态（S-mode）
    ret                   # ③ 内核处理完后 sret 回来，接着执行
```

所有系统调用的区别仅在于 `a7` 中放的数字不同。

### 生成的实际代码

脚本中维护着系统调用列表（[usys.pl:24-45](user/usys.pl#L24-L45)）：

```perl
entry("fork");
entry("exit");
entry("read");
entry("write");
entry("sbrk");
...
```

生成的 `user/usys.S`（含 `#include "kernel/syscall.h"`，编号来自同一份头文件）：

```asm
#include "kernel/syscall.h"

.global fork
fork:
    li a7, SYS_fork      # SYS_fork = 1 (来自 syscall.h)
    ecall
    ret

.global read
read:
    li a7, SYS_read      # SYS_read = 5
    ecall
    ret
...
```

### sbrk 的特殊处理

```perl
sub entry {
    my $name = shift;
    if ($name eq "sbrk") {
        print ".global sys_$name\n";   # 导出名 = sys_sbrk
        print "sys_$name:\n";
    } else {
        print ".global $name\n";        # 导出名 = fork / read / write ...
        print "$name:\n";
    }
    ...
}
```

因为用户态的 [ulib.c](user/ulib.c#L152-L161) 中有一个 C 封装函数：

```c
char* sbrk(int n) {
    return sys_sbrk(n, SBRK_EAGER);   // C 层加 flag，然后调汇编桩
}
```

`sbrk` 的 C 函数和汇编桩不能同名，所以汇编桩命名为 `sys_sbrk`。

## 完整调用链路

```
用户 C 代码:  fork()
    │           jal 跳转到...
    ▼
usys.S 汇编桩:
    li a7, SYS_fork    ① 装编号
    ecall               ② 陷入内核
    │
    ▼ CPU 硬件切到 S-mode，跳转到 stvec
    │
    ▼ trampoline.S uservec
    │ (保存寄存器，切内核页表)
    ▼
trap.c usertrap()
    │ 读 scause = 8 → 系统调用
    │ 读 trapframe->a7 = SYS_fork
    ▼
syscall.c syscall()
    │ syscalls[1] = sys_fork()
    ▼
sysproc.c sys_fork()
    │ 调用 kfork()
    ▼
proc.c kfork()
    │ 复制进程，设置子进程 a0=0
    │
    ▼ 返回到 trampoline.S userret
    │ sret → User mode
    ▼
回到用户态 fork() 调用点:
    父进程 a0 = 子进程 pid
    子进程 a0 = 0
```

## 三段编号必须一致

三个文件，同一个编号体系：

| 文件 | 角色 | 示例 |
|------|------|------|
| [kernel/syscall.h](kernel/syscall.h) | **权威编号源** | `#define SYS_fork 1` |
| [user/usys.pl](user/usys.pl) | **生成用户侧入口** | `li a7, SYS_fork` → `ecall` |
| [kernel/syscall.c](kernel/syscall.c#L109-L134) | **内核侧分发表** | `[SYS_fork] sys_fork` |

usys.pl 生成的 usys.S 中 `#include "kernel/syscall.h"`，因此编号直接引用内核头文件，保证用户态和内核态用同一套数字。

## 设计意义

1. **消除机械重复**：19 个系统调用，每个 4 行汇编，手写是 76 行完全同构的样板代码。脚本一行 `entry("fork")` 解决。
2. **单一真相源**：编号定义在 `syscall.h` 一处，usys.pl 和 syscall.c 都引用它。不会出现"用户态传编号 5 但内核以为是 6"的 bug。
3. **可维护性**：添加新系统调用只需改 3 处——`syscall.h` 加编号、`usys.pl` 加一行 `entry`、`syscall.c` 加分发表表项（再加一个实现函数）。
