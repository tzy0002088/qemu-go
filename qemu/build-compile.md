---
title: "QEMU 编译指南（RISC-V）"
date: 2026-07-08
tags: [qemu, build, compile, riscv, meson, ninja]
status: notes
---

## 背景

系统学习 QEMU，第一步是能编译出 RISC-V 模拟器并保留调试符号，方便后续 GDB 跟踪源码。

## 编译步骤

### 1. 安装依赖

```bash
# Ubuntu/Debian
sudo apt install ninja-build meson build-essential pkg-config \
  libglib2.0-dev libpixman-1-dev libfdt-dev libslirp-dev
```

### 2. 配置（out-of-tree build）

```bash
cd /home/tzy/inspiration/my_qemu/build
../configure --target-list=riscv64-softmmu --enable-debug --enable-slirp
```

| 选项 | 作用 |
|------|------|
| `--target-list=riscv64-softmmu` | 只编译 riscv64 系统模拟器，大幅缩短编译时间 |
| `--enable-debug` | 开 `-O0` + TCG/graph-lock/mutex 调试断言，保留完整调试符号 |
| `--enable-slirp` | 启用用户态网络（SLIRP），Guest 无需管理员权限即可上网 |

> **按需选择 target-list**：如果还需要 32 位支持，改为 `riscv64-softmmu,riscv32-softmmu`。目前只学 riscv64 的话，单 target 足够。

### 3. 编译

```bash
ninja -j$(nproc)
# 也可以用 make，等价效果。ninja 更快一些
```

### 4. 验证

```bash
# 版本检查
./qemu-system-riscv64 --version

# 确认调试符号未 strip
file qemu-system-riscv64 | grep 'not stripped'
# 输出: ... with debug_info, not stripped
```

实际编译产物：

```
qemu-system-riscv64: ELF 64-bit LSB pie executable, x86-64,
  dynamically linked, with debug_info, not stripped
```

## 构建系统流程

```
../configure  ──→  meson setup  ──→  ninja  ──→  qemu-system-riscv64
 (shell脚本)      (meson.build)     (实际编译)     可执行文件
```

- `../configure` 检查编译环境、解析参数，最终调用 `meson setup` 生成 `build.ninja`
- `ninja` 读取 `build.ninja`，按依赖关系并行编译（比 Make 更快，尤其增量编译时）
- configure 的参数会记录在 `build/config.status` 中，可复现当前配置

## 要点

- **out-of-tree build**：在 `build/` 目录下执行 `../configure`，源码和产物隔离
- QEMU 默认 optimization=2（见 `meson.build` 的 `default_options`），`--enable-debug` 会覆盖为 `-O0`
- `--enable-slirp` 提供用户态网络栈，Guest 可通过 `-netdev user` 上网，无需宿主机配置 tap/bridge
- 单 target 编译很快（~2-3 分钟），如果加了 `--enable-debug` 会稍慢但可调试
- 如需编译所有 target，去掉 `--target-list` 即可（但会耗时很久，不推荐学习阶段用）

## 参考

- [meson.build](../../my_qemu/meson.build) 中的 `default_options` 和 `project()` 声明
- configure 脚本 `--enable-debug` 分支
- `build/config.status` — 记录当前 configure 参数，可随时查看
