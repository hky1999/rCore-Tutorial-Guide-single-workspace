# 应用程序链接与加载

本节将介绍批处理系统如何将多个用户程序链接到内核，并在运行时加载执行。

## 应用程序的组织方式

在第一章中，我们只有一个裸机应用程序。而批处理系统需要管理多个应用程序，并依次执行它们。

观察 `user/cases.toml` 中第二章的配置：

```toml
[ch2]
base = 0x8040_0000
step = 0
cases = [
    "00hello_world",
    "01store_fault",
    "02power",
    "03priv_inst",
    "04priv_csr",
]
```

这个配置指定了：

- `base = 0x8040_0000`: 所有程序的加载地址都是 `0x80400000`
- `step = 0`: 程序间没有地址间隔（因为批处理系统每次只运行一个）
- `cases`: 要构建和运行的用户程序列表

> **为什么 step = 0？**
> 
> 在批处理系统中，程序是依次执行的。当一个程序结束后，内核会清空该地址空间，然后将下一个程序加载到同一位置。这与第三章的多道程序系统不同——那里需要同时在内存中保留多个程序，所以 `step = 0x0020_0000`，每个程序占用 2MB 空间。

## 应用程序的构建过程

用户程序的构建由 `xtask/src/user.rs` 中的 `build_for` 函数控制。当我们运行 `cargo qemu --ch 2` 时，构建过程如下：

1. **读取配置**：从 `user/cases.toml` 读取第二章的应用列表

2. **编译各个程序**：对于每个程序（如 `00hello_world`），执行：

   ```bash
   cargo build --package user_lib --bin 00hello_world \
       --target riscv64gc-unknown-none-elf
   ```

   由于 `base = 0x80400000` 且 `step = 0`，所有程序都被链接到相同的起始地址。

3. **生成链接汇编**：创建 `target/riscv64gc-unknown-none-elf/debug/app.asm` 文件：

   ```asm
   .global apps
   .section .data
   .align 3
   apps:
       .quad 0x80400000      # base 地址
       .quad 0               # step（间隔为 0）
       .quad 5               # 应用程序数量
       .quad app_0_start     # 各程序起始位置
       .quad app_1_start
       .quad app_2_start
       .quad app_3_start
       .quad app_4_start
       .quad app_4_end       # 最后一个程序的结束位置
   
   app_0_start:
       .incbin "path/to/00hello_world.bin"
   app_0_end:
   
   app_1_start:
       .incbin "path/to/01store_fault.bin"
   app_1_end:
   
   # ... 其他程序
   ```

这个汇编文件定义了一个全局符号 `apps`，它是一个数据结构，记录了：

- 前三个字段：`base`、`step`、`count`
- 后续字段：各个程序二进制数据在内核中的位置

4. **链接到内核**：`ch2/src/main.rs` 通过以下代码引入应用程序：

   ```rust
   // 在编译时将 app.asm 中定义的应用程序二进制数据嵌入内核
   core::arch::global_asm!(include_str!(env!("APP_ASM")));
   ```

   `env!("APP_ASM")` 是构建脚本设置的环境变量，指向 `app.asm` 文件。

## linker 组件：管理应用程序元数据

`linker` 组件提供了访问应用程序元数据的接口。

### AppMeta 结构

`linker/src/app.rs` 定义了应用程序元数据：

```rust
#[repr(C)]
pub struct AppMeta {
    base: u64,       // 应用加载的基地址
    step: u64,       // 应用间地址间隔
    count: u64,      // 应用数量
    first: u64,      // 第一个应用的起始地址指针
}

impl AppMeta {
    pub fn locate() -> &'static Self {
        extern "C" {
            static apps: AppMeta;  // 链接 app.asm 中的 apps 符号
        }
        unsafe { &apps }
    }

    pub fn iter(&'static self) -> AppIterator {
        AppIterator { meta: self, i: 0 }
    }
}
```

`locate()` 方法找到链接脚本中的 `apps` 符号，将其解释为 `AppMeta` 结构。

### AppIterator：遍历并加载应用

迭代器在遍历时会自动加载应用程序：

```rust
impl Iterator for AppIterator {
    type Item = &'static [u8];

    fn next(&mut self) -> Option<Self::Item> {
        if self.i >= self.meta.count {
            None
        } else {
            let i = self.i as usize;
            self.i += 1;
            unsafe {
                // 读取应用程序在内核数据段中的位置
                let slice = core::slice::from_raw_parts(
                    &self.meta.first as *const _ as *const usize,
                    (self.meta.count + 1) as _,
                );
                let pos = slice[i];          // 当前应用的数据起始
                let size = slice[i + 1] - pos;  // 应用大小
                let base = self.meta.base as usize + i * self.meta.step as usize;
                
                if base != 0 {
                    // 将应用从内核数据段拷贝到目标地址
                    core::ptr::copy_nonoverlapping::<u8>(
                        pos as _, base as _, size
                    );
                    // 清空后续空间（确保程序有足够的运行空间）
                    core::slice::from_raw_parts_mut(base as *mut u8, 0x20_0000)
                        [size..].fill(0);
                    Some(core::slice::from_raw_parts(base as _, size))
                } else {
                    Some(core::slice::from_raw_parts(pos as _, size))
                }
            }
        }
    }
}
```

这个迭代器的巧妙之处在于：

- 每次调用 `next()` 时，它会将下一个应用的二进制数据从内核数据段拷贝到 `base + i * step` 地址
- 在 ch2 中，由于 `base = 0x80400000` 且 `step = 0`，所有程序都被加载到同一地址
- 拷贝后还会清空后续的内存空间（`0x200000` = 2MB），为程序提供运行空间

## 批处理执行流程

在 `ch2/src/main.rs` 中，批处理的核心循环如下：

```rust
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
```

执行流程总结：

1. 迭代器自动将下一个应用加载到 `0x80400000`
2. 创建用户态上下文，设置 PC 为应用入口，SP 为用户栈顶
3. 通过 `ctx.execute()` 切换到用户态执行应用
4. 当发生 Trap（系统调用或异常）时，返回内核处理
5. 处理完成后，要么继续执行（系统调用），要么终止程序（退出或异常）
6. 当前应用结束后，循环继续，加载下一个应用

> **fence.i 指令的作用**
> 
> 在应用程序结束后，我们需要执行 `fence.i` 指令清空指令缓存（i-cache）。这是因为下一个应用会被加载到同一地址，如果不清空 i-cache，CPU 可能会执行到缓存中旧程序的指令，导致错误。

## 小结

本节介绍了批处理系统中应用程序的组织、构建、链接和加载过程：

- 应用程序通过 `cases.toml` 配置，由 `xtask` 构建工具编译
- 所有应用的二进制数据通过 `app.asm` 链接到内核数据段
- `linker` 组件的 `AppMeta` 和 `AppIterator` 提供了访问应用的接口
- 批处理执行流程通过迭代器依次加载并执行应用

与第三章的多道程序系统相比，批处理系统的设计更简单：

- 所有程序共用同一地址（`step = 0`）
- 每次只运行一个程序，运行结束后清空内存
- 不需要复杂的内存管理和调度算法

下一节将介绍 `kernel-context` 组件，它实现了用户态和内核态之间的上下文切换。
