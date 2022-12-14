# 认识 Cargo

Rust 语言的包管理工具是 `cargo`。无需再手动安装，之前安装 Rust 的时候，就已经一并安装了。

```bash
$ cargo new world_hello
$ cd world_hello
```

上面的命令使用 `cargo new` 创建一个项目，项目名是 `world_hello` ，该项目的结构和配置文件都是由 `cargo` 生成，意味着**我们的项目被 `cargo` 所管理**。

> 如果你在终端无法使用这个命令，考虑一下 `环境变量` 是否正确的设置：把 `cargo` 可执行文件所在的目录添加到环境变量中。

现在的版本，`cargo` 默认就创建 `bin` 类型的项目，顺便说一句，Rust 项目主要分为两个类型：`bin` 和 `lib`，前者是一个可运行的项目，后者是一个依赖库项目。

下面来看看创建的项目结构， `git` 已经创建了：

```bash
$ tree
.
├── .git
├── .gitignore
├── Cargo.toml
└── src
    └── main.rs
```

## 运行项目

有两种方式可以运行项目：

1. `cargo run`
2. 手动编译和运行项目

首先来看看第一种方式，在之前创建的 `world_hello` 目录下运行：

```bash
$ cargo run
   Compiling world_hello v0.1.0 (/Users/sunfei/development/rust/world_hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running `target/debug/world_hello`
Hello, world!
```

> 如果你安装的 Rust 的 `host triple` 是 `x86_64-pc-windows-msvc` 并确认 Rust 已经正确安装，但在终端上运行上述命令时，出现类似如下的错误摘要 `linking with `link.exe` failed: exit code: 1181`，请使用 Visual Studio Installer 安装 `Windows SDK`。

上述代码，`cargo run` 首先对项目进行编译，然后再运行，因此它实际上等同于运行了两个指令，下面我们手动试一下编译和运行项目：

**编译：**

```bash
$ cargo build
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
```

**运行：**

```bash
$ ./target/debug/world_hello
Hello, world!
```

在调用的时候，路径 `./target/debug/world_hello` 中有一个明晃晃的 `debug` 字段，没错我们运行的是 `debug` 模式，在这种模式下，**代码的编译速度会非常快，但运行速度慢**。 原因是，在 `debug` 模式下，Rust 编译器不会做任何的优化，只为了尽快的编译完成。

如果想要高性能的代码，添加 `--release` 来编译：

- `cargo run --release`
- `cargo build --release`

## cargo check

当项目大了后，`cargo run` 和 `cargo build` 不可避免的会变慢，那么有没有更快的方式来验证代码的正确性呢？大杀器来了，接着！

`cargo check` 是我们在代码开发过程中最常用的命令，它的作用很简单：快速的检查一下代码能否编译通过。因此该命令速度会非常快，能节省大量的编译时间。

```console
$ cargo check
    Checking world_hello v0.1.0 (/Users/sunfei/development/rust/world_hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
```

> Rust 虽然编译速度还行，但是还是不能 Go 语言相提并论，因为 Rust 需要做很多复杂的编译优化和语言特性解析，甚至连如何优化编译速度都成了一门学问: [优化编译速度](https://course.rs/profiling/compiler/speed-up.html)

## Cargo.toml 和 Cargo.lock

`Cargo.toml` 和 `Cargo.lock` 是 `cargo` 的核心文件，它的所有活动均基于此二者。

- `Cargo.toml` 是 `cargo` 特有的**项目数据描述文件**。它存储了项目的所有元配置信息，如果 Rust 开发者希望 Rust 项目能够按照期望的方式进行构建、测试和运行，那么，必须按照合理的方式构建 `Cargo.toml`。
- `Cargo.lock` 文件是 `cargo` 工具根据同一项目的 `toml` 文件生成的**项目依赖详细清单**，因此我们一般不用修改它，只需要对着 `Cargo.toml` 文件撸就行了。

> 什么情况下该把 `Cargo.lock` 上传到 git 仓库里？很简单，当你的项目是一个可运行的程序时，就上传 `Cargo.lock`，如果是一个依赖库项目，那么请把它添加到 `.gitignore` 中

现在用 VSCode 打开上面创建的"世界，你好"项目，然后进入根目录的 `Cargo.toml` 文件，可以看到该文件包含不少信息：

### package 配置段落

`package` 中记录了项目的描述信息，典型的如下：

```toml
[package]
name = "world_hello"
version = "0.1.0"
edition = "2021"
```

`name` 字段定义了项目名称，`version` 字段定义当前版本，新项目默认是 `0.1.0`，`edition` 字段定义了我们使用的 Rust 大版本，泽你使用的是 `Rust edition 2021` 大版本，详情见 [Rust 版本详解](https://course.rs/appendix/rust-version.html)

### 定义项目依赖

使用 `cargo` 工具的最大优势就在于，能够对该项目的各种依赖项进行方便、统一和灵活的管理。

在 `Cargo.toml` 中，主要通过各种依赖段落来描述该项目的各种依赖项：

- **基于 Rust 官方仓库 `crates.io`，通过版本说明来描述**
- **基于项目源代码的 git 仓库地址，通过 URL 来描述**
- **基于本地项目的绝对路径或者相对路径，通过类 Unix 模式的路径来描述**

这三种形式具体写法如下：

```toml
[dependencies]
rand = "0.3"
hammer = { version = "0.5.0"}
color = { git = "https://github.com/bjz/color-rs" }
geometry = { path = "crates/geometry" }
```

相信聪明的读者已经能看懂该如何引入外部依赖库，这里就不再赘述。详细的说明参见此章：[Cargo 依赖管理](https://course.rs/cargo/reference/specify-deps.html)。

# 不仅仅是 Hello world

打开 上一节 中创建的 `world_hello` 工程，然后进入 `main.rs` 文件。（此文件是当前 Rust 工程的入口文件，和其它语言几无区别。）

接下来，对世界友人给予热切的问候：

```rust
fn greet_world() {
     let southern_germany = "Grüß Gott!";
     let chinese = "世界，你好";
     let english = "World, hello";
     let regions = [southern_germany, chinese, english];
     for region in regions.iter() {
             println!("{}", &region);
     }
 }

 fn main() {
     greet_world();
 }
```

打开终端，进入 `world_hello` 工程根目录，运行该程序。

```console
$ cargo run
   Compiling world_hello v0.1.0 (/Users/sunfei/development/rust/world_hello)
    Finished dev [unoptimized + debuginfo] target(s) in 0.21s
     Running `target/debug/world_hello`
Grüß Gott!
世界，你好
World, hello
```

**Rust 原生支持 UTF-8 编码的字符串**，这意味着你可以很容易的使用**世界各国文字**作为字符串内容。

其次，关注下 `println` 后面的 `!`，如果你有 Ruby 编程经验，那么你可能会认为这是解构操作符，但是在 Rust 中，**这是 `宏` 操作符，你目前可以认为宏是一种特殊类型函数**。

对于 `println` 来说，我们没有使用其它语言惯用的 `%s`、`%d` 来做输出占位符，**而是使用 `{}`**，因为 Rust 在底层帮我们做了大量工作，**会自动识别输出数据的类型**，例如当前例子，会识别为 `String` 类型。

**Rust 的集合类型不能直接进行循环，需要变成迭代器**（这里是通过 `.iter()` 方法）。以后就知道这么做的好处所在。

> 实际上这段代码可以简写，在 2021 edition 及以后，支持直接写 `for region in regions`，原因会在迭代器章节的开头提到，是因为 **for 隐式地将 regions 转换成迭代器**。

# Rust 语言初印象

Rust 这门语言对于 Haskell 和 Java 开发者来说，可能会觉得很熟悉，因为它们在高阶表达方面都很优秀。简而言之，就是可以很简洁的写出原本需要一大堆代码才能表达的含义。但是，Rust 又有所不同：它的性能是底层语言级别的性能，可以跟 C/C++ 相媲美。

让我们来点狠活：

```rust
fn main() {
   let penguin_data = "\
   common name,length (cm)
   Little penguin,33
   Yellow-eyed penguin,65
   Fiordland penguin,60
   Invalid,data
   ";

   let records = penguin_data.lines();

   for (i, record) in records.enumerate() {
     if i == 0 || record.trim().len() == 0 {
       continue;
     }

     // 声明一个 fields 变量，类型是 Vec
     // Vec 是 vector 的缩写，是一个可伸缩的集合类型，可以认为是一个动态数组
     // <_>表示 Vec 中的元素类型由编译器自行推断，在很多场景下，都会帮我们省却不少功夫
     let fields: Vec<_> = record
       .split(',')
       .map(|field| field.trim())
       .collect();
     if cfg!(debug_assertions) {
         // 输出到标准错误输出
       eprintln!("debug: {:?} -> {:?}",
              record, fields);
     }

     let name = fields[0];
     // 1. 尝试把 fields[1] 的值转换为 f32 类型的浮点数，如果成功，则把 f32 值赋给 length 变量
     //
     // 2. if let 是一个匹配表达式，用来从=右边的结果中，匹配出 length 的值：
     //   1）当=右边的表达式执行成功，则会返回一个 Ok(f32) 的类型，若失败，则会返回一个 Err(e) 类型，if let 的作用就是仅匹配 Ok 也就是成功的情况，如果是错误，就直接忽略
     //   2）同时 if let 还会做一次解构匹配，通过 Ok(length) 去匹配右边的 Ok(f32)，最终把相应的 f32 值赋给 length
     //
     // 3. 当然你也可以忽略成功的情况，用 if let Err(e) = fields[1].parse::<f32>() {...}匹配出错误，然后打印出来，但是没啥卵用
     if let Ok(length) = fields[1].parse::<f32>() {
         // 输出到标准输出
         println!("{}, {}cm", name, length);
     }
   }
 }
```

看完这段代码，不知道你的余音有没有戛然而止，反正我已经在颤抖了。这就是传说中的下马威吗？😵

上面代码中，值得注意的 Rust 特性有：

- 控制流：`for` 和 `continue` 连在一起使用，实现循环控制。
- 方法语法：由于 Rust 没有继承，因此 Rust **不是传统意义上的面向对象语言**，但是它却从 `OO` 语言那里偷师了方法的使用 `record.trim()`，`record.split(',')` 等。
- 高阶函数编程：**函数可以作为参数也能作为返回值**，例如 `.map(|field| field.trim())`，**这里 `map` 方法中使用闭包函数作为参数，也可以称呼为 `匿名函数`、`lambda 函数`**。
- 类型标注：`if let Ok(length) = fields[1].parse::<f32>()`，通过 `::<f32>` 的使用，**告诉编译器 `length` 是一个 `f32` 类型的浮点数。这种类型标注不是很常用，但是在编译器无法推断出你的数据类型时，就很有用了**。
- **条件编译：`if cfg!(debug_assertions)`，说明紧跟其后的输出（打印）只在 `debug` 模式下生效**。
- 隐式返回：Rust 提供了 `return` 关键字用于函数返回，但是在很多时候，我们可以省略它。因为 Rust 是 `基于表达式的语言`。

在终端中运行上述代码时，会看到很多 `debug: ...` 的输出，上面有讲，这些都是 `条件编译` 的输出。

`cargo run` 默认是运行 `debug` 模式。因此想要消灭那些 `debug:` 输出，需要更改为其它模式，其中最常用的模式就是 `--release` 也就是生产发布的模式。

