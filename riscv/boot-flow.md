---
title: "RISC-V 启动流程"
date: 2026-07-02
tags: [riscv, boot, firmware, opensbi]
status: draft
---

## 背景

在分析 QEMU RISC-V virt 机器启动过程时，对从复位向量到内核入口的完整路径有困惑。

## 提问

1. RISC-V 上电后 PC 从哪里开始执行？
2. OpenSBI 在启动过程中扮演什么角色？
3. QEMU 中 RISC-V 的复位向量是如何设置的？

## 理解

（在此记录学到的内容）

## 要点

- QEMU RISC-V virt 机器的复位向量由 `resetvec` 属性决定
- 典型的启动链：ZSBL → FSBL(OpenSBI) → Kernel
- OpenSBI 运行在 M-mode，为内核提供 SBI 接口

## 参考

- QEMU 源码: `hw/riscv/virt.c`
- OpenSBI 文档: https://github.com/riscv-software-src/opensbi
