# 特权级切换机制

本节将深入介绍批处理系统中最核心的机制：**特权级切换**。这是实现用户态和内核态隔离的基础。

## 为什么需要特权级切换

在第一章中，我们的裸机应用运行在机器模式（M 模式）或监督模式（S 模式）下，拥有对硬件的完全控制权。但在批处理系统中，我们希望：

- **隔离**：用户程序运行在用户态（U 模式），无法直接访问硬件或内核内存
- **保护**：即使用户程序出错或恶意，也不能破坏操作系统
- **服务**：用户程序通过系统调用请求内核提供服务

这需要在用户态和内核态之间频繁切换：

1. **用户 → 内核**：系统调用、异常、中断
2. **内核 → 用户**：系统调用返回、启动程序

## RISC-V 特权级切换的硬件支持

RISC-V 提供了三种特权级：

- **M 模式**（Machine）：机器模式，最高权限，通常用于固件（如 RustSBI）
- **S 模式**（Supervisor）：监督模式，操作系统内核运行在此模式
- **U 模式**（User）：用户模式，应用程序运行在此模式

特权级切换涉及的 CSR（控制状态寄存器）：

| CSR 名称 | 功能描述 |
|---------|---------|
| `sstatus` | 保存处理器状态，包括 SPP（上一特权级）、SPIE（中断使能）等 |
| `sepc` | 保存触发 Trap 的指令地址（或下一条指令地址） |
| `scause` | 保存 Trap 的原因（系统调用、异常类型等） |
| `stval` | 保存 Trap 的附加信息（如错误的内存地址） |
| `stvec` | 设置 Trap 处理程序的入口地址 |
| `sscratch` | 临时寄存器，用于保存/交换数据 |

### 硬件自动完成的工作

当 CPU 从用户态陷入（Trap）到内核态时，硬件自动：

1. 将当前特权级（U）保存到 `sstatus.SPP`
2. 将下一条指令地址保存到 `sepc`
3. 将 Trap 原因保存到 `scause`，附加信息保存到 `stval`
4. 跳转到 `stvec` 指向的地址，并切换到 S 模式

### sret 指令：返回用户态

内核使用 `sret` 指令返回用户态，硬件自动：

1. 根据 `sstatus.SPP` 恢复特权级（切换回 U 模式）
2. 跳转到 `sepc` 指向的地址继续执行

## kernel-context 组件：封装上下文切换

`kernel-context` 组件提供了一个优雅的抽象，封装了特权级切换的底层细节。

### LocalContext 结构

`kernel-context/src/lib.rs` 定义了线程上下文：

```rust
#[repr(C)]
pub struct LocalContext {
    sctx: usize,            // 调度上下文（栈指针）
    x: [usize; 31],         // 通用寄存器 x1-x31（x0 硬编码为 0）
    sepc: usize,            // 程序计数器
    supervisor: bool,       // 是否为内核线程
    interrupt: bool,        // 是否开启中断
}
```

这个结构保存了执行线程所需的全部上下文：

- `x[31]`：保存所有通用寄存器（x1-x31，x0 不需要保存）
- `sepc`：程序计数器（PC），指向下一条要执行的指令
- `supervisor`：指示目标特权级（false = 用户态，true = 内核态）
- `interrupt`：是否在目标模式下开启中断

### 创建用户态上下文

```rust
pub const fn user(pc: usize) -> Self {
    Self {
        sctx: 0,
        x: [0; 31],
        supervisor: false,    // 用户态
        interrupt: true,      // 在用户态时，内核中断是开启的
        sepc: pc,             // 设置程序入口
    }
}
```

在 `ch2/src/main.rs` 中使用：

```rust
let mut ctx = LocalContext::user(app_base);
*ctx.sp_mut() = user_stack_top;  // 设置用户栈指针
```

### 访问寄存器的便捷方法

`LocalContext` 提供了访问寄存器的方法：

```rust
pub fn a(&self, n: usize) -> usize {
    self.x(n + 10)  // a0-a7 对应 x10-x17
}

pub fn a_mut(&mut self, n: usize) -> &mut usize {
    self.x_mut(n + 10)
}

pub fn sp(&self) -> usize {
    self.x(2)  // sp 是 x2
}

pub fn move_next(&mut self) {
    self.sepc = self.sepc.wrapping_add(4);  // 跳过当前指令（假设是 4 字节）
}
```

这些方法让我们可以方便地读写系统调用参数、栈指针等。

## 上下文切换的核心：execute 方法

`execute` 方法是上下文切换的核心，它完成从内核态到用户态的切换，执行用户程序，并在 Trap 发生时切换回内核态。

```rust
#[inline(never)]
pub unsafe fn execute(&mut self) -> usize {
    let mut sstatus = build_sstatus(self.supervisor, self.interrupt);
    let ctx_ptr = self as *mut Self;
    let mut sepc = self.sepc;
    let old_sscratch: usize;
    
    core::arch::asm!(
        // 1. 保存上下文指针到 sscratch，保存旧值
        "   csrrw {old_ss}, sscratch, {ctx}
            csrw  sepc    , {sepc}
            csrw  sstatus , {sstatus}
        // 2. 保存返回地址并调用汇编函数
            addi  sp, sp, -8
            sd    ra, (sp)
            call  {execute_naked}
            ld    ra, (sp)
            addi  sp, sp,  8
        // 3. 恢复 sscratch 并读取返回值
            csrw  sscratch, {old_ss}
            csrr  {sepc}   , sepc
            csrr  {sstatus}, sstatus
        ",
        ctx           = in       (reg) ctx_ptr,
        old_ss        = out      (reg) old_sscratch,
        sepc          = inlateout(reg) sepc,
        sstatus       = inlateout(reg) sstatus,
        execute_naked = sym execute_naked,
    );
    
    (*ctx_ptr).sepc = sepc;
    sstatus
}
```

执行流程：

1. 构建 `sstatus` 寄存器的值（设置特权级和中断状态）
2. 将上下文指针保存到 `sscratch`（用于 Trap 时恢复）
3. 调用 `execute_naked` 切换到用户态
4. 用户程序执行，直到发生 Trap
5. Trap 处理程序将控制权交还给 `execute_naked`
6. `execute_naked` 返回，恢复内核状态

## execute_naked：汇编级上下文切换

`execute_naked` 是一个裸函数（naked function），完全由汇编实现，不经过 Rust 的函数调用约定。它完成实际的寄存器保存/恢复和特权级切换。

```rust
#[unsafe(naked)]
unsafe extern "C" fn execute_naked() {
    core::arch::naked_asm!(
        // 定义保存/加载寄存器的宏
        r"  .altmacro
            .macro SAVE n
                sd x\n, \n*8(sp)
            .endm
            .macro LOAD n
                ld x\n, \n*8(sp)
            .endm
            // ... 省略 SAVE_ALL 和 LOAD_ALL 的定义
        ",
        
        // === 阶段 1：保存调度上下文（内核栈） ===
        "   addi sp, sp, -32*8      # 在栈上分配空间
            SAVE_ALL                 # 保存所有寄存器
        ",
        
        // === 阶段 2：设置 Trap 处理入口 ===
        "   la   t0, 1f              # 加载标签 1 的地址
            csrw stvec, t0           # 设置 stvec 为陷入处理入口
        ",
        
        // === 阶段 3：切换到用户上下文 ===
        "   csrr t0, sscratch        # 读取上下文指针（之前保存在 sscratch）
            sd   sp, (t0)            # 保存调度栈指针到上下文
            mv   sp, t0              # 切换 sp 到用户上下文
        ",
        
        // === 阶段 4：恢复用户寄存器并执行 ===
        "   LOAD_ALL                 # 恢复所有用户寄存器
            ld   sp, 2*8(sp)         # 恢复用户栈指针（x2 = sp）
            sret                     # 返回用户态！
        ",
        
        // === 阶段 5：Trap 入口（标签 1） ===
        "   .align 2                 # 对齐到 4 字节
        1:  csrrw sp, sscratch, sp   # 交换 sp 和 sscratch
        ",
        
        // === 阶段 6：保存用户上下文 ===
        "   SAVE_ALL                 # 保存用户寄存器
            csrrw t0, sscratch, sp   # 恢复用户 sp 到 t0
            sd    t0, 2*8(sp)        # 将用户 sp 保存到上下文
        ",
        
        // === 阶段 7：恢复调度上下文并返回 ===
        "   ld sp, (sp)              # 恢复调度栈指针
            LOAD_ALL                 # 恢复调度寄存器
            addi sp, sp, 32*8        # 释放栈空间
            ret                      # 返回到 execute 函数
        ",
    )
}
```

### 执行流程图解

```
[内核态]
   ↓ execute()
保存上下文指针到 sscratch
   ↓ call execute_naked

[execute_naked 开始]
保存内核寄存器到内核栈 ← 阶段 1
   ↓
设置 stvec = 标签 1     ← 阶段 2（Trap 入口）
   ↓
从 sscratch 读取用户上下文
保存内核 sp 到上下文
切换 sp 到用户上下文    ← 阶段 3
   ↓
恢复用户寄存器
sret ────────────────→ [用户态执行]
                          ↓
                       系统调用/异常
                          ↓
                       硬件自动：
                       - SPP = U
                       - sepc = PC
                       - PC = stvec
                          ↓
[标签 1：Trap 入口]   ← 阶段 5
交换 sp 和 sscratch   ← 现在 sp 指向用户上下文
   ↓
保存用户寄存器        ← 阶段 6
   ↓
恢复内核 sp
恢复内核寄存器        ← 阶段 7
   ↓
ret ─────────────────→ [返回 execute()]

[内核态]
处理 Trap
```

## 关键设计要点

### 1. 双栈设计

- **内核栈**：保存内核的调度上下文（execute 函数的栈帧）
- **用户上下文**：LocalContext 结构本身，保存用户寄存器

### 2. sscratch 寄存器的妙用

- 在执行用户程序时，`sscratch` 保存用户上下文的指针
- 发生 Trap 时，通过 `csrrw sp, sscratch, sp` 一条指令同时：
  - 将用户 sp 保存到 sscratch
  - 将上下文指针加载到 sp
- 这样就能立即访问上下文结构，保存寄存器

### 3. stvec 的动态设置

- 在 `execute_naked` 中，我们将 `stvec` 设置为标签 `1` 的地址
- 这是一个**局部的 Trap 处理入口**，专门用于从用户态返回
- 相比全局的 Trap 处理函数，这种方式更高效，因为我们知道：
  - 上下文已经在 `sscratch` 中
  - 只需要保存寄存器并返回

## 小结

本节介绍了批处理系统中特权级切换的核心机制：

- **硬件层面**：RISC-V 提供了 CSR 和 `sret` 指令支持特权级切换
- **抽象层面**：`kernel-context` 组件的 `LocalContext` 封装了上下文
- **实现层面**：`execute` 和 `execute_naked` 完成寄存器保存/恢复和模式切换

通过这套机制，我们实现了：

- 用户程序在 U 模式运行，无法破坏内核
- 系统调用和异常能够安全地切换到 S 模式处理
- 内核可以无缝地启动和恢复用户程序

下一节将介绍 `syscall` 组件，它基于上下文切换机制，提供了模块化的系统调用框架。
