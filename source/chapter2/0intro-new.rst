引言
================================

本章导读
---------------------------------

**批处理系统** (Batch System) 是操作系统发展史上的重要里程碑。在计算资源匮乏的年代，批处理系统通过将多个程序打包执行，极大提升了计算机的利用率。

本章将在第一章实现的裸机应用基础上，构建一个具备批处理能力的操作系统。主要目标包括:

- 理解并实现特权级隔离机制，保护操作系统不受错误程序破坏
- 设计模块化的系统调用框架，建立用户程序与内核的接口
- 实现上下文切换机制，支持用户态和内核态之间的无缝切换
- 构建批处理执行流程，自动加载并运行多个用户程序

相比第一章，本章引入了四个关键组件:

- **linker**: 管理内核内存布局和应用程序链接
- **rcore-console**: 提供格式化输出和日志功能
- **kernel-context**: 实现特权级切换的核心机制
- **syscall**: 定义可扩展的系统调用接口

通过这些组件的协同工作，我们将实现一个能够批量运行用户程序的操作系统。

实践体验
---------------------------

在 ``rCore-Tutorial-in-single-workspace`` 项目中运行本章代码：

.. code-block:: console
   
   $ cd rCore-Tutorial-in-single-workspace
   $ LOG=INFO cargo qemu --ch 2

运行后可以看到批处理系统依次执行了多个用户程序:

.. code-block::

   [rustsbi] RustSBI version 0.3.1, adapting to RISC-V SBI v1.0.0
   .______       __    __      _______.___________.  _______..______   __
   |   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
   |  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
   |      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
   |  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
   | _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|
   [rustsbi] Implementation     : RustSBI-QEMU Version 0.2.0-alpha.2
   [rustsbi] Platform Name      : riscv-virtio,qemu
   [rustsbi] Platform SMP       : 1
   [rustsbi] Platform Memory    : 0x80000000..0x84000000
   [rustsbi] Boot HART          : 0
   [rustsbi] Device Tree Region : 0x83e00000..0x83e010f6
   [rustsbi] Firmware Address   : 0x80000000
   [rustsbi] Supervisor Address : 0x80200000
   [rustsbi] pmp01: 0x00000000..0x80000000 (-wr)
   [rustsbi] pmp02: 0x80000000..0x80200000 (---)
   [rustsbi] pmp03: 0x80200000..0x84000000 (xwr)
   [rustsbi] pmp04: 0x84000000..0x00000000 (-wr)

      ______                       __
     / ____/___  ____  _________  / /__
    / /   / __ \/ __ \/ ___/ __ \/ / _ \
   / /___/ /_/ / / / (__  ) /_/ / /  __/
   \____/\____/_/ /_/____/\____/_/\___/
   ====================================
   [ INFO] LOG TEST >> Hello, world!
   [ WARN] LOG TEST >> Hello, world!
   [ERROR] LOG TEST >> Hello, world!

   [ INFO] load app0 to 0x80400000
   [ INFO] load app1 to 0x80400000
   [ INFO] load app2 to 0x80400000
   [ INFO] load app3 to 0x80400000
   [ INFO] load app4 to 0x80400000

   Hello, world from user mode program!
   [ INFO] app0 exit with code 0

   Into Test store_fault, we will insert an invalid store operation...
   Kernel should kill this application!
   [ERROR] app1 was killed because of Exception(StoreFault)

   3^10000=5079(MOD 10007)
   3^20000=8202(MOD 10007)
   3^30000=8824(MOD 10007)
   3^40000=5750(MOD 10007)
   3^50000=3824(MOD 10007)
   Test power OK!
   [ INFO] app2 exit with code 0

   Try to execute privileged instruction in U Mode
   Kernel should kill this application!
   [ERROR] app3 was killed because of Exception(IllegalInstruction)

   Try to access privileged CSR in U Mode
   Kernel should kill this application!
   [ERROR] app4 was killed because of Exception(IllegalInstruction)

可以看到：

- 系统能够依次自动加载并执行多个应用程序
- 某些程序（如 ``app1``、``app3``、``app4``）故意触发异常
- 内核能够正确捕获异常并终止出错程序，继续执行下一个程序
- 正常程序（如 ``app0``、``app2``）能够成功完成并退出

这展示了批处理系统的核心能力：**容错性**和**自动调度**。


本章代码树
-------------------------------------------------

.. code-block::

   ├── ch2（内核实现）
   │   ├── Cargo.toml（配置文件，新增四个组件依赖）
   │   ├── build.rs（构建脚本，生成链接脚本）
   │   └── src
   │       └── main.rs（内核主函数，实现批处理逻辑）
   ├── linker（链接管理组件）
   │   └── src
   │       ├── lib.rs（定义内核内存布局）
   │       └── app.rs（应用程序元数据和加载）
   ├── console（输出组件）
   │   └── src
   │       └── lib.rs（提供 print!/println! 和 log 功能）
   ├── kernel-context（上下文切换组件）
   │   └── src
   │       └── lib.rs（实现用户态/内核态上下文切换）
   ├── syscall（系统调用组件）
   │   └── src
   │       ├── lib.rs（系统调用框架）
   │       ├── syscalls.rs（系统调用号定义）
   │       ├── io.rs（I/O 相关系统调用）
   │       └── kernel/mod.rs（内核端系统调用接口）
   ├── user（用户程序）
   │   ├── cases.toml（测试用例配置）
   │   ├── build.rs（用户程序构建脚本）
   │   └── src
   │       ├── lib.rs（用户库）
   │       └── bin（测试程序）
   │           ├── 00hello_world.rs
   │           ├── 01store_fault.rs
   │           ├── 02power.rs
   │           ├── 03priv_inst.rs
   │           └── 04priv_csr.rs
   └── xtask（构建工具）
       └── src
           ├── main.rs（主构建逻辑）
           └── user.rs（用户程序构建）


本章代码导读
-----------------------------------------------------

相比第一章，本章的主要变化体现在：

**组件化设计**

第二章引入了模块化的组件架构。每个组件负责特定功能，通过清晰的接口协同工作：

- ``linker`` 组件管理内核和应用程序的内存布局
- ``console`` 组件提供统一的输出接口
- ``kernel-context`` 组件封装了特权级切换的底层细节  
- ``syscall`` 组件定义了可扩展的系统调用框架

**批处理执行流程**

``ch2/src/main.rs`` 中的 ``rust_main`` 函数实现了批处理的核心逻辑：

.. code-block:: rust

    for (i, app) in linker::AppMeta::locate().iter().enumerate() {
        let app_base = app.as_ptr() as usize;
        log::info!("load app{i} to {app_base:#x}");
        // 初始化上下文并执行
        let mut ctx = LocalContext::user(app_base);
        // ... 设置栈，执行应用
        loop {
            unsafe { ctx.execute() };
            // 处理 Trap
        }
    }

这个循环遍历所有应用程序，为每个程序创建执行上下文，并处理其执行过程中的 Trap。

**用户程序的构建与链接**

第二章的用户程序与内核一起构建。构建流程包括：

1. ``xtask/src/user.rs`` 读取 ``user/cases.toml`` 中的配置
2. 为每个用户程序编译生成二进制文件
3. 生成 ``app.asm`` 文件，将所有程序链接到内核
4. 内核通过 ``linker::AppMeta`` 访问这些程序

与第三章不同，第二章的所有程序都被加载到**同一地址** ``0x80400000``，每次只能运行一个。这是因为批处理系统采用"加载-执行-清空"的简单模式，不需要多道程序的内存隔离。

接下来的章节将详细介绍各个组件的设计与实现。
