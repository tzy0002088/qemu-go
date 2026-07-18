# xv6-riscv 编译构建系统

## 目录结构

```
kernel/   -- 内核源码和链接脚本
user/     -- 用户态程序和库
mkfs/     -- 构建机上运行的文件系统镜像生成工具
```

整个项目**只有一个 Makefile**（根目录），没有递归 Make。

## 工具链

### 交叉编译器

自动检测优先级（通过探测 `<prefix>objdump -i` 是否识别 `elf64-big`）：

1. `riscv64-unknown-elf-`（首选，基于 newlib 的 bare-metal 工具链）
2. `riscv64-elf-`
3. `riscv64-none-elf-`
4. `riscv64-linux-gnu-`
5. `riscv64-unknown-linux-gnu-`

### 模拟器

`qemu-system-riscv64`，最低版本 7.2。

### 主机工具

- `mkfs/mkfs`：用主机 GCC 编译（运行在构建机上操作文件）
- `perl`：用于 `user/usys.pl` 生成系统调用桩
- `clang-format`：可选，用于 `make fmt`

## 关键编译标志（CFLAGS）

```makefile
-Wall -Werror -O -fno-omit-frame-pointer -ggdb -gdwarf-2
-march=rv64gc -mcmodel=medany
-ffreestanding -nostdlib -fno-common -MD
-fno-builtin-<strncpy|strncmp|memset|memcpy|printf|...>
-Wno-main -fno-stack-protector
```

要点：
- `-march=rv64gc`：目标 ISA = RV64I + M/A/F/D/C 扩展
- `-mcmodel=medany`：内核加载在 `0x80000000`（超出 small 模式的 ±2GB 范围），且内核还会映射到高虚拟地址
- `-ffreestanding -nostdlib`：无宿主标准库的裸机环境
- `-MD`：自动生成 `.d` 依赖文件
- PIE 被条件性禁用（检测编译器 spec 输出，兼容不同工具链）

## 构建目标

| 目标 | 说明 |
|------|------|
| `kernel/kernel`（默认） | 内核 ELF 二进制 |
| `fs.img` | 文件系统镜像 |
| `_%`（模式规则） | 编译单个用户程序 |
| `qemu` | 运行 xv6 |
| `qemu-gdb` | QEMU + GDB 调试 |
| `clean` | 清理构建产物 |
| `fmt` | clang-format 格式化 |

## 内核编译与链接

所有 `kernel/*.c` → `kernel/*.o`，然后：

```makefile
$(LD) $(LDFLAGS) -T kernel/kernel.ld -o kernel/kernel $(OBJS)
```

链接脚本要点（`kernel/kernel.ld`）：
- `ENTRY(_entry)`，加载基址 `0x80000000`
- `.text` 段首必须是 `kernel/entry.o(_entry)`
- **trampoline 页**放在 `trampsec` 段，严格页对齐（ASSERT 确保恰好一页）
- 提供 `etext`、`end` 符号给内核内存分配器

链接后生成：`kernel/kernel.asm`（反汇编）、`kernel/kernel.sym`（符号表）

## 用户程序编译

**系统调用桩生成：** `user/usys.pl` → `user/usys.S`

对每个系统调用生成：
```asm
.global <name>
<name>:
    li a7, SYS_<name>
    ecall
    ret
```

**用户库 (ULIB)：** `user/ulib.o` + `user/usys.o` + `user/printf.o` + `user/umalloc.o`

每个用户程序 `user/X.c` → `user/_X`（前缀下划线避免与主机二进制冲突）

特例：`forktest` 用 `-N -e main -Ttext 0` 链接，且不链接 printf/umalloc，保持体积最小。

## 文件系统镜像

`mkfs/mkfs` 用**主机 GCC** 编译。镜像参数（来自 `kernel/param.h`）：
- 总大小 2000 块，每块 1024 字节
- 200 个 inode，30 个日志块

```
[boot block(1) | superblock(1) | log(30) | inodes | bitmap | data blocks]
```

## QEMU 配置

```makefile
QEMUOPTS = -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic
QEMUOPTS += -global virtio-mmio.force-legacy=false
QEMUOPTS += -drive file=fs.img,if=none,format=raw,id=x0
QEMUOPTS += -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0
```

要点：
- `-bios none`：无固件，内核直接运行
- `-kernel kernel/kernel`：加载 ELF 到 `0x80000000`
- `-m 128M`：128MB 物理内存
- `fs.img` 以 virtio-blk 设备接入

## GDB 调试

```makefile
GDBPORT = $(shell expr `id -u` % 5000 + 25000)  # 每用户独立端口
```

- `.gdbinit.tmpl-riscv` → `.gdbinit`（sed 替换端口号）
- `qemu-gdb`：QEMU 加 `-S`（启动时冻结 CPU）+ GDB stub

## 设计亮点

1. **单 Makefile**：无递归，依赖关系一目了然
2. **双工具链**：内核/用户用交叉编译器，mkfs 用主机 GCC
3. **trampoline 页 ASSERT**：链接时校验恰好一页，防止用户/内核切换机制损坏
4. **PIE 条件禁用**：检查编译器 spec 自适应不同工具链
5. **每用户独立 GDB 端口**：多用户同机调试不冲突
6. **无 initramfs**：直接用 mkfs 预构建原始块设备镜像
