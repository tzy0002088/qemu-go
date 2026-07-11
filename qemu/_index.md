# QEMU

QEMU 整体架构与核心子系统学习记录。

## 子方向

- [内存模型](../qemu-memory/_index.md) — MemoryRegion、AddressSpace、IOMMU
- （更多子方向按需添加）

## 记录列表

| 日期 | 记录 | 状态 | 说明 |
|------|------|------|------|
| 2026-07-09 | [device-modeling-guide.md](./device-modeling-guide.md) | notes | 外设建模分级指南：pinctrl/clock/PHY/DMA/GPIO 等特殊外设的建模方式 |
| 2026-07-02 | [deep-dive-foundations.md](./deep-dive-foundations.md) | notes | 深度剖析：QOM 类型系统、MemoryRegion 内部机制、Guest 指令到设备回调全路径 |
| 2026-07-02 | [learning-roadmap.md](./learning-roadmap.md) | notes | QEMU 学习路线图：嵌入式工程师视角，8 阶段从零到建模 RISC-V 芯片 |
| 2026-07-02 | [aspeed-bmc.md](./aspeed-bmc.md) | notes | Aspeed BMC 设备模拟全览：4 代 SoC、23 款机型、全部外设 |
| 2026-07-02 | [sensor-emulation.md](./sensor-emulation.md) | notes | QEMU 传感器模拟：温度/气压/PMBus 等 I2C 传感器 |
| 2026-07-08 | [build-compile.md](./build-compile.md) | notes | QEMU 编译指南：RISC-V 模拟器编译步骤与构建系统分析 |
| 2026-07-08 | [architecture-overview.md](./architecture-overview.md) | notes | QEMU 整体架构框架：从 main() 到 Guest 指令执行的完整路径 |
| 2026-07-08 | [qemu-init.md](./qemu-init.md) | notes | QEMU 初始化流程：从 main() 到 qemu_main_loop() 的关键步骤 |
| 2026-07-08 | [virt-analysis.md](./virt-analysis.md) | notes | virt.c 源码分析：RISC-V VirtIO Board 的设计思想与结构 |
| 2026-07-09 | [api-cheatsheet.md](./api-cheatsheet.md) | notes | 芯片建模核心 API 速查：3 类 15 个 API，外设/SoC/Machine |
| 2026-07-08 | [type-init-mechanism.md](./type-init-mechanism.md) | notes | type_init 自动注册机制：constructor → .init_array → register_module_init |
