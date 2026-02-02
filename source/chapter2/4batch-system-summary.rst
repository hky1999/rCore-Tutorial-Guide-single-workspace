辅助组件与批处理系统总结
=====================================

本节介绍 ``console`` 和 ``linker`` 的其他功能，并总结批处理系统的整体设计。

console 组件：格式化输出
------------------------------------

``rcore-console`` 组件提供了统一的输出接口，支持格式化打印和日志功能。

**Console Trait**

``console/src/lib.rs`` 定义了输出接口：

.. code-block:: rust

    pub trait Console: Sync {
        fn put_char(&self, c: u8);
        
        fn put_str(&self, s: &str) {
            for c in s.bytes() {
                self.put_char(c);
            }
        }
    }

这个 trait 要求实现 ``put_char`` 方法，``put_str`` 有默认实现。

**注册输出实现**

内核需要提供一个实现 ``Console`` 的结构：

.. code-block:: rust

    // ch2/src/main.rs
    
    struct Console;
    
    impl rcore_console::Console for Console {
        fn put_char(&self, c: u8) {
            sbi_rt::legacy::console_putchar(c as usize);
        }
    }

然后在启动时注册：

.. code-block:: rust

    rcore_console::init_console(&Console);
    rcore_console::set_log_level(option_env!("LOG"));
    rcore_console::test_log();

**print! 和 println! 宏**

``console`` 组件通过单例模式保存 ``Console`` 实现，并提供宏：

.. code-block:: rust

    static CONSOLE: Once<&'static dyn Console> = Once::new();

    pub fn init_console(console: &'static dyn Console) {
        CONSOLE.call_once(|| console);
    }

    #[doc(hidden)]
    pub fn _print(args: fmt::Arguments) {
        Logger.write_fmt(args).unwrap();
    }

    #[macro_export]
    macro_rules! println {
        () => ($crate::print!("\n"));
        ($($arg:tt)*) => {{
            $crate::_print(core::format_args!($($arg)*));
            $crate::println!();
        }}
    }

这样内核和用户程序都可以使用 ``print!`` 和 ``println!`` 宏。

**日志功能**

``console`` 还集成了 ``log`` crate，提供分级日志：

.. code-block:: rust

    impl log::Log for Logger {
        fn enabled(&self, _metadata: &Metadata) -> bool {
            true
        }

        fn log(&self, record: &Record) {
            if !self.enabled(record.metadata()) {
                return;
            }
            
            print_in_color(
                format_args!("[{:>5}] {}\n", record.level(), record.args()),
                level_to_color_code(record.level()),
            );
        }

        fn flush(&self) {}
    }

使用示例：

.. code-block:: rust

    log::info!("load app{i} to {app_base:#x}");
    log::error!("app{i} was killed because of {trap:?}");

日志级别可以通过环境变量控制：

.. code-block:: console

    $ LOG=INFO cargo qemu --ch 2


linker 组件：内核内存布局
------------------------------------

``linker`` 组件不仅管理应用程序，还定义了内核的内存布局。

**链接脚本**

``linker/src/lib.rs`` 中定义了链接脚本常量：

.. code-block:: rust

    pub const SCRIPT: &[u8] = b"\
    OUTPUT_ARCH(riscv)
    SECTIONS {
        .text 0x80200000 : {
            __start = .;
            *(.text.entry)
            *(.text .text.*)
        }
        .rodata : ALIGN(4K) {
            __rodata = .;
            *(.rodata .rodata.*)
            *(.srodata .srodata.*)
        }
        .data : ALIGN(4K) {
            __data = .;
            *(.data .data.*)
            *(.sdata .sdata.*)
        }
        .bss : ALIGN(8) {
            __sbss = .;
            *(.bss .bss.*)
            *(.sbss .sbss.*)
            __ebss = .;
        }
        .boot : ALIGN(4K) {
            __boot = .;
            KEEP(*(.boot.stack))
        }
        __end = .;
    }";

这个脚本定义了内核的各个段：

- ``.text``：代码段，起始地址 ``0x80200000``
- ``.rodata``：只读数据段（字符串常量等）
- ``.data``：可写数据段（全局变量等）
- ``.bss``：未初始化数据段
- ``.boot``：启动数据段（启动栈等）

**KernelLayout 结构**

``linker`` 还提供了访问内核布局的接口：

.. code-block:: rust

    pub struct KernelLayout {
        text: usize,
        rodata: usize,
        data: usize,
        sbss: usize,
        ebss: usize,
        boot: usize,
        end: usize,
    }

    impl KernelLayout {
        pub fn locate() -> Self {
            extern "C" {
                fn __start();
                fn __rodata();
                fn __data();
                fn __sbss();
                fn __ebss();
                fn __boot();
                fn __end();
            }
            Self {
                text: __start as usize,
                rodata: __rodata as usize,
                data: __data as usize,
                sbss: __sbss as usize,
                ebss: __ebss as usize,
                boot: __boot as usize,
                end: __end as usize,
            }
        }

        pub unsafe fn zero_bss(&self) {
            core::slice::from_raw_parts_mut(self.sbss as *mut u8, self.ebss - self.sbss)
                .fill(0);
        }
    }

在内核启动时，我们需要清零 ``.bss`` 段：

.. code-block:: rust

    extern "C" fn rust_main() -> ! {
        unsafe { linker::KernelLayout::locate().zero_bss() };
        // ... 其他初始化
    }

**boot0! 宏：定义内核入口**

``linker`` 提供了 ``boot0!`` 宏来定义内核启动栈和入口：

.. code-block:: rust

    #[macro_export]
    macro_rules! boot0 {
        ($entry:ident; stack = $stack:expr) => {
            #[unsafe(naked)]
            #[no_mangle]
            #[link_section = ".text.entry"]
            unsafe extern "C" fn _start() -> ! {
                #[link_section = ".boot.stack"]
                static mut STACK: [u8; $stack] = [0u8; $stack];

                core::arch::naked_asm!(
                    "la sp, __end",
                    "j  {main}",
                    main = sym $entry,
                )
            }
        };
    }

使用：

.. code-block:: rust

    linker::boot0!(rust_main; stack = 4 * 4096);

这会生成一个 ``_start`` 函数，它：

1. 放置在 ``.text.entry`` 段（链接脚本保证它在最前面）
2. 定义一个 16KB 的启动栈（放在 ``.boot.stack`` 段）
3. 设置 ``sp`` 寄存器指向 ``__end``（即启动栈顶）
4. 跳转到 ``rust_main`` 函数


批处理系统的完整执行流程
------------------------------------

总结整个批处理系统从启动到运行的完整流程：

**1. 启动阶段**

.. code-block::

    [QEMU] 加载 rustsbi-qemu.bin 到 0x80000000
       ↓
    [RustSBI] 初始化硬件，设置 M 模式 Trap
       ↓
    [RustSBI] 跳转到 0x80200000（内核入口）
       ↓
    [_start] 设置栈指针，跳转到 rust_main
       ↓
    [rust_main] 
       - 清零 .bss 段
       - 初始化 console
       - 初始化 syscall

**2. 批处理循环**

.. code-block:: rust

    for (i, app) in linker::AppMeta::locate().iter().enumerate() {
        // 应用已被迭代器加载到 0x80400000
        
        // 创建用户上下文
        let mut ctx = LocalContext::user(app.as_ptr() as usize);
        *ctx.sp_mut() = user_stack_top;
        
        loop {
            // 切换到用户态执行
            unsafe { ctx.execute() };
            
            // 处理 Trap
            match scause::read().cause() {
                UserEnvCall => {
                    // 系统调用
                    match handle_syscall(&mut ctx) {
                        Done => continue,
                        Exit(_) | Error(_) => break,
                    }
                }
                _ => {
                    // 异常
                    log::error!("app killed");
                    break;
                }
            }
        }
    }

**3. 系统调用流程**

.. code-block::

    [用户程序] ecall
       ↓
    [硬件] 切换到 S 模式，PC = stvec
       ↓
    [execute_naked] 保存用户寄存器，返回 execute()
       ↓
    [主循环] 读取 scause，调用 handle_syscall()
       ↓
    [handle_syscall] 读取系统调用参数，调用 syscall::handle()
       ↓
    [syscall::handle] 根据系统调用号分发
       ↓
    [SyscallContext] 执行具体功能（如 write）
       ↓
    [handle_syscall] 写入返回值，PC += 4
       ↓
    [主循环] continue，再次 ctx.execute()
       ↓
    [execute_naked] 恢复用户寄存器，sret
       ↓
    [用户程序] 继续执行


关键设计要点回顾
------------------------------------

**1. 组件化架构**

批处理系统通过四个核心组件实现功能分离：

- ``linker``：内存布局和应用链接
- ``console``：统一输出接口
- ``kernel-context``：特权级切换
- ``syscall``：系统调用框架

**2. 单地址加载**

批处理系统的所有应用加载到同一地址（``0x80400000``），采用"加载-执行-清空"模式。
这简化了内存管理，但限制了并发性。

**3. 容错机制**

通过特权级隔离和 Trap 处理，内核能够：

- 捕获用户程序的异常（非法指令、访存错误等）
- 安全地终止出错程序
- 继续执行下一个程序

**4. 可扩展性**

``syscall`` 组件的 trait 设计为后续章节奠定了基础：

- 第三章：添加 ``Scheduling`` trait（yield 系统调用）
- 第五章：扩展 ``Process`` trait（fork、exec）
- 第六章：添加文件系统相关系统调用


与第三章的对比
------------------------------------

.. list-table:: 批处理系统 vs 多道程序系统
   :header-rows: 1
   :align: center
   :widths: 30 35 35

   * - 特性
     - 第二章（批处理）
     - 第三章（多道程序）
   * - 程序加载地址
     - 同一地址（``step = 0``）
     - 不同地址（``step = 2MB``）
   * - 并发性
     - 顺序执行
     - 时分复用，协作式调度
   * - 内存占用
     - 只保留当前程序
     - 同时保留多个程序
   * - 任务切换
     - 程序结束后加载下一个
     - yield 系统调用主动让出 CPU
   * - 复杂度
     - 较低
     - 增加任务管理和调度


小结
------------------------------------

本章实现了一个功能完整的批处理系统，具备：

- **自动调度**：依次加载并执行多个用户程序
- **容错性**：异常程序不会破坏内核
- **系统调用**：提供基本的 I/O 和进程管理接口
- **模块化设计**：为后续章节奠定了良好的架构基础

通过批处理系统，我们学习了：

- RISC-V 特权级机制
- 上下文切换的底层实现
- 系统调用的完整流程
- 组件化的操作系统设计

在下一章中，我们将在批处理系统的基础上，引入任务切换和协作式调度，
实现**多道程序系统**，进一步提升 CPU 利用率。
