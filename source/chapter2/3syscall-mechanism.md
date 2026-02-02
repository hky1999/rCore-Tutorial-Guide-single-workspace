# 系统调用机制

本节将介绍 `syscall` 组件，它提供了一个模块化、可扩展的系统调用框架。

## 系统调用的作用

用户程序运行在用户态（U 模式），无法直接访问硬件或内核功能。要实现 I/O、进程管理等功能，用户程序必须通过**系统调用**请求内核提供服务。

系统调用的本质是一次**受控的特权级切换**：

1. 用户程序通过 `ecall` 指令触发异常
2. CPU 切换到内核态，跳转到 Trap 处理入口
3. 内核根据系统调用号分发请求，执行相应功能
4. 内核将结果写入寄存器，通过 `sret` 返回用户态

## RISC-V 系统调用约定

RISC-V 使用寄存器传递系统调用参数和返回值：

| 寄存器 | 用途 |
|-------|------|
| `a7` (x17) | 系统调用号（SyscallId） |
| `a0-a5` (x10-x15) | 系统调用参数（最多 6 个） |
| `a0` (x10) | 系统调用返回值 |

例如，`write` 系统调用：

```rust
// 用户态调用
syscall::write(1, "Hello\n".as_bytes());

// 编译为（简化）：
li   a7, 64        # SyscallId::WRITE = 64
li   a0, 1         # fd = 1（标准输出）
la   a1, string    # buf = "Hello\n" 的地址
li   a2, 6         # len = 6
ecall              # 触发系统调用
# 返回后，a0 = 实际写入的字节数
```

## syscall 组件的设计

`syscall` 组件采用**特征对象**（trait object）设计，将系统调用分类为不同的功能模块。

### 系统调用分类

`syscall/src/kernel/mod.rs` 定义了多个 trait，每个 trait 对应一类系统调用：

```rust
pub trait IO: Sync {
    fn read(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        unimplemented!()
    }
    fn write(&self, caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        unimplemented!()
    }
    fn open(&self, caller: Caller, path: usize, flags: usize) -> isize {
        unimplemented!()
    }
    fn close(&self, caller: Caller, fd: usize) -> isize {
        unimplemented!()
    }
}

pub trait Process: Sync {
    fn exit(&self, caller: Caller, status: usize) -> isize {
        unimplemented!()
    }
    fn fork(&self, caller: Caller) -> isize {
        unimplemented!()
    }
    // ... 其他进程相关系统调用
}

pub trait Clock: Sync {
    fn clock_gettime(&self, caller: Caller, clock_id: ClockId, tp: usize) -> isize {
        unimplemented!()
    }
}

// 还有 Memory、Scheduling、Signal、Thread、SyncMutex 等 trait
```

这种设计的优点：

- **模块化**：每类系统调用独立定义，职责清晰
- **可扩展**：新增系统调用只需扩展对应 trait
- **按需实现**：不同章节可以实现不同的 trait 子集

### Caller 结构：标识调用者

```rust
pub struct Caller {
    pub entity: usize,  // 资源集标记（相当于进程号）
    pub flow: usize,    // 控制流标记（相当于线程号）
}
```

`Caller` 标识系统调用的发起者。在批处理系统中，所有应用都是单线程的，所以我们简单地传递 `Caller { entity: 0, flow: 0 }`。到多进程、多线程的章节，这个结构会发挥重要作用。

### 全局容器：注册系统调用实现

```rust
static IO: Container<dyn IO> = Container::new();
static PROCESS: Container<dyn Process> = Container::new();
static CLOCK: Container<dyn Clock> = Container::new();
// ... 其他容器

pub fn init_io(io: &'static dyn IO) {
    IO.init(io);
}

pub fn init_process(process: &'static dyn Process) {
    PROCESS.init(process);
}
```

这些全局容器保存各类系统调用的实现。内核在启动时注册实现：

```rust
// ch2/src/main.rs
syscall::init_io(&SyscallContext);
syscall::init_process(&SyscallContext);
```

### 系统调用分发

`handle` 函数根据系统调用号分发请求：

```rust
pub fn handle(caller: Caller, id: SyscallId, args: [usize; 6]) -> SyscallResult {
    use SyscallId as Id;
    match id {
        Id::WRITE => IO.call(id, |io| io.write(caller, args[0], args[1], args[2])),
        Id::READ => IO.call(id, |io| io.read(caller, args[0], args[1], args[2])),
        Id::EXIT => PROCESS.call(id, |p| p.exit(caller, args[0])),
        Id::CLOCK_GETTIME => CLOCK.call(id, |c| c.clock_gettime(
            caller, 
            args[0].into(), 
            args[1]
        )),
        // ... 其他系统调用
        _ => SyscallResult::Unsupported(id),
    }
}
```

`Container::call` 的实现：

```rust
impl<T: ?Sized> Container<T> {
    fn call<R>(&self, id: SyscallId, f: impl FnOnce(&T) -> isize) -> SyscallResult {
        match self.0.get() {
            Some(inner) => SyscallResult::Done(f(inner)),
            None => SyscallResult::Unsupported(id),
        }
    }
}
```

如果对应的容器已初始化，调用实现函数；否则返回不支持。

## ch2 的系统调用实现

在 `ch2/src/main.rs` 中，我们实现了 `SyscallContext` 结构：

```rust
struct SyscallContext;

impl syscall::IO for SyscallContext {
    fn write(&self, _caller: Caller, fd: usize, buf: usize, count: usize) -> isize {
        if fd == syscall::STDOUT || fd == syscall::STDERR {
            // 将用户空间的缓冲区解释为字节切片
            let slice = unsafe { core::slice::from_raw_parts(buf as *const u8, count) };
            print!("{}", core::str::from_utf8(slice).unwrap());
            count as isize
        } else {
            -1  // 不支持的文件描述符
        }
    }
}

impl syscall::Process for SyscallContext {
    fn exit(&self, _caller: Caller, _status: usize) -> isize {
        0  // 在批处理系统中，exit 只是一个标记，不做实际操作
    }
}
```

这个实现非常简单：

- `write`：只支持标准输出（fd = 1）和标准错误（fd = 2），直接调用 `print!`
- `exit`：在批处理系统中，退出由外层循环判断，这里只返回 0

### 处理系统调用的完整流程

在 `ch2/src/main.rs` 的主循环中：

```rust
loop {
    unsafe { ctx.execute() };  // 执行用户程序
    
    use scause::{Exception, Trap};
    match scause::read().cause() {
        Trap::Exception(Exception::UserEnvCall) => {
            // 发生系统调用
            use SyscallResult::*;
            match handle_syscall(&mut ctx) {
                Done => continue,
                Exit(code) => {
                    log::info!("app exit with code {code}");
                    break;
                }
                Error(id) => {
                    log::error!("unsupported syscall {}", id.0);
                    break;
                }
            }
        }
        trap => {
            log::error!("killed because of {trap:?}");
            break;
        }
    }
    unsafe { core::arch::asm!("fence.i") };
}
```

`handle_syscall` 函数：

```rust
enum SyscallResult {
    Done,
    Exit(usize),
    Error(SyscallId),
}

fn handle_syscall(ctx: &mut LocalContext) -> SyscallResult {
    use syscall::{SyscallId as Id, SyscallResult as Ret};

    // 从寄存器读取系统调用信息
    let id = ctx.a(7).into();  // a7 = 系统调用号
    let args = [ctx.a(0), ctx.a(1), ctx.a(2), ctx.a(3), ctx.a(4), ctx.a(5)];
    
    // 调用系统调用处理函数
    match syscall::handle(Caller { entity: 0, flow: 0 }, id, args) {
        Ret::Done(ret) => match id {
            Id::EXIT => SyscallResult::Exit(ctx.a(0)),
            _ => {
                *ctx.a_mut(0) = ret as _;  // 将返回值写入 a0
                ctx.move_next();            // PC += 4，跳过 ecall 指令
                SyscallResult::Done
            }
        },
        Ret::Unsupported(id) => SyscallResult::Error(id),
    }
}
```

处理流程：

1. 从上下文读取系统调用号（a7）和参数（a0-a5）
2. 调用 `syscall::handle` 执行系统调用
3. 将返回值写入 a0
4. 调用 `ctx.move_next()` 使 PC 指向 `ecall` 的下一条指令
5. 返回 `Done`，继续执行用户程序

特殊处理 `EXIT` 系统调用：它会导致程序终止，返回 `Exit(code)`。

## 用户库中的系统调用

用户程序通过 `user/src/lib.rs` 中的封装函数发起系统调用。

`syscall` 组件提供了用户态的系统调用接口（`features = ["user"]`）：

```rust
// syscall/src/user.rs

#[inline(always)]
pub fn write(fd: usize, buf: &[u8]) -> isize {
    syscall(SyscallId::WRITE, [fd, buf.as_ptr() as _, buf.len(), 0, 0, 0])
}

#[inline(always)]
pub fn exit(code: i32) -> ! {
    syscall(SyscallId::EXIT, [code as _, 0, 0, 0, 0, 0]);
    unreachable!()
}

#[inline(always)]
fn syscall(id: SyscallId, args: [usize; 6]) -> isize {
    let mut ret: isize;
    unsafe {
        core::arch::asm!(
            "ecall",
            in("a7") id.0,
            inlateout("a0") args[0] => ret,
            in("a1") args[1],
            in("a2") args[2],
            in("a3") args[3],
            in("a4") args[4],
            in("a5") args[5],
        );
    }
    ret
}
```

`syscall` 函数将参数加载到寄存器，执行 `ecall` 指令，然后读取返回值。

## 系统调用的完整旅程

以 `write(1, "Hi\n", 3)` 为例，完整流程如下：

```
[用户程序] 调用 syscall::write(1, "Hi\n".as_bytes())
   ↓
[用户库] syscall(WRITE, [1, addr, 3, ...])
   ↓ 内联汇编：
   - a7 = 64 (SyscallId::WRITE)
   - a0 = 1, a1 = addr, a2 = 3
   - ecall
   ↓
[硬件] 切换到 S 模式
   - SPP = U, sepc = PC + 4
   - PC = stvec
   ↓
[kernel-context] execute_naked 的标签 1
   - 交换 sp 和 sscratch
   - 保存用户寄存器到上下文
   - 返回到 execute()
   ↓
[ch2/main.rs] 读取 scause
   - 发现是 UserEnvCall
   - 调用 handle_syscall(&mut ctx)
   ↓
[handle_syscall] 
   - id = ctx.a(7) = 64
   - args = [ctx.a(0), ctx.a(1), ctx.a(2), ...]
   - 调用 syscall::handle(caller, id, args)
   ↓
[syscall::handle]
   - match id = WRITE
   - IO.call(id, |io| io.write(caller, 1, addr, 3))
   ↓
[SyscallContext::write]
   - 解析缓冲区：slice = from_raw_parts(addr, 3)
   - print!("{}", str::from_utf8(slice))
   - 返回 3
   ↓
[handle_syscall]
   - *ctx.a_mut(0) = 3  // 写入返回值
   - ctx.move_next()     // PC += 4
   - 返回 Done
   ↓
[主循环] continue，再次调用 ctx.execute()
   ↓
[execute_naked]
   - 恢复用户寄存器（包括 a0 = 3）
   - sret
   ↓
[用户程序] 从 ecall 的下一条指令继续
   - a0 = 3（返回值）
```

## 小结

本节介绍了 `syscall` 组件的设计与实现：

- **设计理念**：通过 trait 将系统调用分类，使用容器注册实现
- **调用流程**：用户 ecall → Trap → 分发 → 执行 → 返回
- **ch2 实现**：提供了基本的 I/O 和进程系统调用

这套框架的优势在于：

- **可扩展**：新增系统调用只需扩展 trait 和 `handle` 函数
- **模块化**：不同功能的系统调用独立实现和注册
- **类型安全**：通过 `SyscallId` 类型包装，避免硬编码数字

在后续章节中，我们会逐步实现更多的系统调用，如进程管理（fork、exec）、文件系统（open、read）、线程（thread_create）等，都将复用这个框架。

下一节将介绍 `console` 组件和其他辅助功能。
