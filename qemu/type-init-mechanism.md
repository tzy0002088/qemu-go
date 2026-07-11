---
title: "type_init 自动注册机制"
date: 2026-07-08
tags: [qemu, qom, type_init, constructor, init_array, elf]
status: notes
---

## 背景

看到 `virt.c` 最后一行 `type_init(virt_machine_init_register_types)`，好奇它如何在 `main()` 之前自动执行。QEMU 所有的 TypeInfo 注册都靠这个机制。

## 原理

### 宏展开链

```c
// include/qemu/module.h
#define type_init(function)     module_init(function, MODULE_INIT_QOM)
#define block_init(function)    module_init(function, MODULE_INIT_BLOCK)
#define opts_init(function)     module_init(function, MODULE_INIT_OPTS)
// ...

// 核心宏：生成一个 __attribute__((constructor)) 函数
#define module_init(function, type)
static void __attribute__((constructor)) do_qemu_init_ ## function(void)
{
    register_module_init(function, type);
}
```

所以 `type_init(virt_machine_init_register_types)` 展开为：

```c
static void __attribute__((constructor))
do_qemu_init_virt_machine_init_register_types(void)
{
    register_module_init(virt_machine_init_register_types, MODULE_INIT_QOM);
}
```

### ELF 层面：`.init_array` 段

`__attribute__((constructor))` 是 GCC/Clang 扩展，编译器将函数指针放入 `.init_array` 段。

```bash
# 可以自己验证
readelf -S build/qemu-system-riscv64 | grep init_array
objdump -t build/qemu-system-riscv64 | grep do_qemu_init
```

链接器把所有 `.init_array` 条目合并成一张函数指针表。**不需要自定义链接脚本**——这是 ELF 标准机制。

### 执行时机：`_start → __libc_start_main → .init_array → main`

```
_start (crt0.S)
  │
  └── __libc_start_main
        │
        ├── 1. 遍历 .init_array 段，逐条调用 constructor 函数
        │      ├── do_qemu_init_virt_machine_init_register_types()
        │      │     └── register_module_init(..., MODULE_INIT_QOM)
        │      ├── do_qemu_init_spike_...( )
        │      ├── ... 数百个 constructor 依次执行 ...
        │      └── do_qemu_init_tmp105_...( )
        │
        ├── 2. main()
        │      └── qemu_init()
        │            └── module_call_init(MODULE_INIT_QOM)  ← 真正调用注册的函数
        │                  │
        │                  └── 遍历 init_type_list[MODULE_INIT_QOM] 链表
        │                      依次调用每个函数：
        │                        virt_machine_init_register_types()
        │                          → type_register_static(&virt_machine_typeinfo)
        │                        spike_...
        │                        ...所有 TypeInfo 进入全局哈希表
        │
        └── 3. 遍历 .fini_array 段（程序退出时）
```

**关键**：constructor 阶段只是把函数指针记录到链表里，真正调用发生在 `module_call_init()`。

## 数据结构

```c
// util/module.c
typedef struct ModuleEntry {
    void (*init)(void);              // 要调用的函数指针
    QTAILQ_ENTRY(ModuleEntry) node;  // 链表节点
    module_init_type type;           // QOM / BLOCK / OPTS / ...
} ModuleEntry;

// 全局数组，按类型分组，每类一个链表
static ModuleTypeList init_type_list[MODULE_INIT_MAX];

// 防止重复调用
static bool modules_init_done[MODULE_INIT_MAX];
```

### register_module_init — 构造函数中调用，插入链表

```c
void register_module_init(void (*fn)(void), module_init_type type)
{
    ModuleEntry *e;
    e = g_malloc0(sizeof(*e));
    e->init = fn;
    e->type = type;
    QTAILQ_INSERT_TAIL(find_type(type), e, node);  // 尾插，保持注册顺序
}
```

### module_call_init — qemu_init 中调用，遍历链表

```c
void module_call_init(module_init_type type)
{
    if (modules_init_done[type]) return;   // 防重入
    QTAILQ_FOREACH(e, find_type(type), node) {
        e->init();
    }
    modules_init_done[type] = true;
}
```

## 完整时序图

```
程序启动
  │
  │  [.init_array 段执行]         constructor 阶段
  │     │                         只是把函数指针存入链表
  │     └── register_module_init(fn, type)
  │            └── 插入 init_type_list[type] 链表尾部
  │
  └── main()
        └── qemu_init()
              │
              ├── module_call_init(MODULE_INIT_OPTS)       ← 先注册 QemuOpts
              │
              ├── ... 参数解析、machine 选择 ...
              │
              ├── module_call_init(MODULE_INIT_QOM)         ← 再注册 QOM 类型
              │     │
              │     └── 遍历 init_type_list[MODULE_INIT_QOM]
              │           ├── virt_machine_init_register_types()
              │           │     └── type_register_static(&virt_machine_typeinfo)
              │           ├── spike_machine_init_register_types()
              │           └── ... 所有 TypeInfo 进入全局哈希表
              │
              ├── module_call_init(MODULE_INIT_BLOCK)       ← 块设备驱动
              └── ...
```

## 设计思想

1. **去中心化注册**：每个 `.c` 文件只管自己的 `type_init`，不用改任何全局文件来"加一条记录"
2. **两阶段**：constructor（存指针，不能失败）+ `module_call_init`（真正调用，按类型分组），保证初始化顺序可控
3. **纯 C 实现**：不依赖 C++ 全局对象构造、不依赖链接脚本、不依赖 OS 特殊机制
4. **类型分组**：`MODULE_INIT_QOM`、`MODULE_INIT_BLOCK`、`MODULE_INIT_OPTS` 等，每种类型在不同时机初始化，确保依赖顺序

## 参考

- `include/qemu/module.h` — 宏定义
- `util/module.c` — register_module_init / module_call_init 实现
- `include/qom/object.h` — type_register_static / TypeInfo
- GCC 文档: `__attribute__((constructor))` — https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html
