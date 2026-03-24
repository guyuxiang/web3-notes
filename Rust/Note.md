# Rust Note

```
// 更新
rustup update
```

```
// 查看版本
rustc -V

cargo -V
```

```
// 创建项目
cargo new world_hello

// 快速编译并执行
cargo run

// 编译
cargo build
输出在./target/debug/目录下，默认允许的是运行的是 debug 模式，特点是编译快，允许慢

// 高性能模式
cargo run --release
cargo build --release

// 快速检查项目是否可以编译通过
cargo check
```

Cargo.toml 和 Cargo.lock

`Cargo.toml` 和 `Cargo.lock` 是 `cargo` 的核心文件，它的所有活动均基于此二者。

- `Cargo.toml` 是 `cargo` 特有的**项目数据描述文件**。它存储了项目的所有元配置信息，如果 Rust 开发者希望 Rust 项目能够按照期望的方式进行构建、测试和运行，那么，必须按照合理的方式构建 `Cargo.toml`。
- `Cargo.lock` 文件是 `cargo` 工具根据同一项目的 `toml` 文件生成的**项目依赖详细清单**，因此我们一般不用修改它，只需要对着 `Cargo.toml` 文件撸就行了。



## `Cargo.toml` 文件解析

#### [package 配置段落](https://course.rs/first-try/cargo.html#package-配置段落)

`package` 中记录了项目的描述信息，典型的如下：

```toml
[package]
name = "world_hello"
version = "0.1.0"
edition = "2021"
```

`name` 字段定义了项目名称

`version` 字段定义当前版本，新项目默认是 `0.1.0``

``edition` 字段定义了我们使用的 Rust 大版本。使用的是 `Rust edition 2021` 大版本，详情见 [Rust 版本详解](https://course.rs/appendix/rust-version.html)

#### [定义项目依赖](https://course.rs/first-try/cargo.html#定义项目依赖)

使用 `cargo` 工具的最大优势就在于，能够对该项目的各种依赖项进行方便、统一和灵活的管理。

在 `Cargo.toml` 中，主要通过各种依赖段落来描述该项目的各种依赖项：

- 基于 Rust 官方仓库 `crates.io`，通过版本说明来描述
- 基于项目源代码的 git 仓库地址，通过 URL 来描述
- 基于本地项目的绝对路径或者相对路径，通过类 Unix 模式的路径来描述

这三种形式具体写法如下：

```toml
[dependencies]
rand = "0.3"
hammer = { version = "0.5.0"}
color = { git = "https://github.com/bjz/color-rs" }
geometry = { path = "crates/geometry" }
```

详细的说明参见此章：[Cargo 依赖管理](https://course.rs/cargo/reference/specify-deps.html)

## [基于 cargo 的项目组织结构](https://course.rs/first-try/cargo.html#基于-cargo-的项目组织结构)

一个典型的 `Package` 目录结构如下：

```shell
.
├── Cargo.lock
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── main.rs
│   └── bin/
│       ├── named-executable.rs
│       ├── another-executable.rs
│       └── multi-file-executable/
│           ├── main.rs
│           └── some_module.rs
├── benches/
│   ├── large-input.rs
│   └── multi-file-bench/
│       ├── main.rs
│       └── bench_module.rs
├── examples/
│   ├── simple.rs
│   └── multi-file-example/
│       ├── main.rs
│       └── ex_module.rs
└── tests/
    ├── some-integration-tests.rs
    └── multi-file-test/
        ├── main.rs
        └── test_module.rs
```

这也是 `Cargo` 推荐的目录结构，解释如下：

- `Cargo.toml` 和 `Cargo.lock` 保存在 `package` 根目录下

- 源代码放在 `src` 目录下

- 默认的 `lib` 包根是 `src/lib.rs`

- 默认的二进制包根是

   

  ```
  src/main.rs
  ```

  - 其它二进制包根放在 `src/bin/` 目录下

- 基准测试 benchmark 放在 `benches` 目录下

- 示例代码放在 `examples` 目录下

- 集成测试代码放在 `tests` 目录下

关于 Rust 中的包和模块，[之前的章节](https://course.rs/basic/crate-module/intro.html)有更详细的解释。

此外，`bin`、`tests`、`examples` 等目录路径都可以通过配置文件进行配置，它们被统一称之为 [Cargo Target](https://course.rs/cargo/reference/cargo-target.html)。



## `{:?}` 是什么？（Rust 格式化基础）

Rust 的格式化占位符主要有两类：

| 占位符 | 含义                                                         |
| ------ | ------------------------------------------------------------ |
| `{}`   | 直接可视化值，支持基础数值类型、bool、 字符 & 字符串、常用标准库类型（部分支持）如std::fmt::Argument |
| `{:?}` | 会带结构信息，比如对 `String` 来说就是带引号                 |



不是所有类型都支持 `{}`比如你自己定义的结构体，但支持`{:?}`打印



solana的rust支持浮点数计算，可使用浮点数类型





在 Rust 中，函数的命名遵循 **蛇形命名法（snake_case）**，它的规则是：

- 所有字母小写
- 使用下划线分隔单词
- 不使用驼峰式命名（CamelCase）和大写字母





在 Rust 中，错误处理是通过 `Result<T, E>` 类型来实现的，它有两个变体：

- **`Ok(T)`**：表示成功的结果，其中 `T` 是返回的值类型。
- **`Err(E)`**：表示失败的结果，其中 `E` 是错误类型。



## 模块系统

Rust 的模块系统主要解决 3 个问题：

1. **代码组织**（structure）：把代码拆成多个模块/文件
2. **命名空间管理**（namespace）：避免名字冲突
3. **可见性控制**（privacy）：控制哪些能被外部访问

------

### 二、核心概念总览

Rust 模块系统的核心关键词：

| 概念    | 作用                    |
| ------- | ----------------------- |
| `mod`   | 定义模块                |
| `pub`   | 控制可见性              |
| `use`   | 引入路径（类似 import） |
| `crate` | 当前包                  |
| `super` | 父模块                  |
| `self`  | 当前模块                |
| `::`    | 路径分隔符              |

### 规则：

- `mod math` → 定义模块
- 默认都是 **私有(private)** 的
- `pub` 才能被外部访问
- 访问路径用 `::`



Rust 模块是**树结构**：

```
crate
 └── math
     ├── add()
     └── secret()
```

`crate` 是根节点



```
└── user/
     ├── mod.rs
     ├── service.rs
     └── model.rs
```

mod.rs里面放 pub mod xxx



Rust 2018+ 新风格

```
src/
 ├── main.rs
 ├── user.rs
 └── user/
     ├── service.rs
     └── model.rs
```

user.rs和user同名同级

user.rs里面放 pub mod xxx



### use 导入机制

### 重命名

```
use crate::user::service::login as user_login;
```

###  批量导入

```
use crate::user::{service, model};
```

## 绝对路径

```
crate::user::service::login();
```

## 相对路径

```
super::service::login();  // 父模块
self::helper();           // 当前模块
```



### 入口/根

普通 Rust 程序（binary）：src/main.rs

库（library）：src/lib.rs

运行在特殊运行时（如 **Anchor** / wasm / embedded）：src/lib.rs

Solana Runtime → Anchor entrypoint → #[program] 模块



因此

 一个Anchor的 crate 的默认入口文件就是 src/lib.rs。

 编译器会把 lib.rs 当成整个 crate 的根模块（crate::）

模块“被声明”的位置决定了它在模块树里的路径

比如：pub mod error; 放在 lib.rs 里，意思是：error 是 crate 根模块的直接子模块，路径是 crate::error。

Rust 的模块系统默认按“文件名或目录名”来解析：

  - mod constants; 会去找同级的 constants.rs，或者同级目录 constants/mod.rs
  - mod instructions; 会去找 instructions.rs，或者 instructions/mod.rs













 * 变量与常性 (Variables & Immutability)： 理解 let 和 let mut 的区别（Rust 默认不可变）。
 * 基本数据类型： 重点掌握 u8, u64, u128（链上处理代币金额常用）和 bool。
 * 结构体 (Structs)： Solana 的 Account（账户）本质上就是结构体。
 * 枚举 (Enums)： 逻辑判断和状态管理（如：订单状态 Open, Closed, Cancelled）的核心。
 * 函数 (Functions)： 参数传递、返回值语法。

 * 所有权 (Ownership) 与 借用 (Borrowing)： * 弄明白什么是 &（不可变引用）和 &mut（可变引用）。
   * Solana 合约中经常要传递 &mut Account。
 * Result 与 Option 模式：
   * 这是 Rust 处理错误的方式。链上开发严禁使用 panic!（会导致程序崩溃）。
   * 必须学会使用 ? 操作符来传播错误。
 * Vector (Vec) 与 String： 了解动态数组和字符串的基本操作。

 * 宏 (Macros)： * 声明式宏： 比如 msg!() 用来打印日志。
   * 过程宏 (Procedural Macros)： 这是 Anchor 的核心，如 #[program], #[derive(Accounts)], #[account].
   * 学习建议： 不需要学会怎么写这些宏，但要明白它们是在帮你生成代码。
 * Trait (特征)：
   * 理解类似接口的概念。Anchor 的账户序列化全靠 AnchorSerialize 和 AnchorDeserialize 这两个 Trait。
 * 模块系统 (Modules)： mod, use, pub 的用法，用来组织大项目的代码结构。
